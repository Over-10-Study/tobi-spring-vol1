AOP
===

### 트랜잭션 코드의 분리

지금까지 서비스 추상화 기법을 적용해 트랜잭션 기술에 독립적으로 만들었다. 하지만 아직 트랜잭션 경계설정과 비즈니스 로직이 공존한다.**분리해보자**

1.	메소드 분리

```java
// 분리 전
public void upgradeLevels() throws Exception {
  TransactionStatus status = this.transactionManager.getTransaction(new DefaultTransactionDefinition());

  try {
    List<User> users = userDao.getAll();
    for (User user : users) {
      if (canUpgradeLevel(user)) {
        upgradeLevel(user);
      }
    }

    this.transactionManager.commit(status);
  } catch (Exception e) {
    this.transactionManager.rollback(status);
    throw e;
  }
}
```

```java
// 분리 후
public void upgradeLevels() throws Exception {
  TransactionStatus status = this.transactionManager.getTransaction(new DefaultTransactionDefinition());

  try {
    upgradeLevelsInternal();
    this.transactionManager.commit(status);
  } catch (Exception e) {
    this.transactionManager.rollback(status);
    throw e;
  }
}

private void upgradeLevelsInternal() {
  List<User> users = userDao.getAll();
  for (User user : users) {
    if (canUpgradeLevel(user)) {
      upgradeLevel(user);
    }
  }
}
```

여전히 트랜잭션 코드가 UserService 안에 존재한다.<br> 2. DI를 이용한 클래스 분리

```java
// UserService 인터페이스
public interface UserService {
  void add(User user);
  void upgradeLevels();
}
```

```java
// 트랜잭션 코드를 제거한 UserService 구현 클래스
public class UserServiceImpl implements UserService {
  UserDao userDao;
  MailSender mailSender;

  public void upgradeLevels() {
    List<User> users = userDao.getAll();
    for (User user : users) {
      if (canUpgradeLevel(user)) {
        upgradeLevel(user);
      }
    }
  }
}
```

```java
// 트랜잭션 처리를 담은 UserServiceTx 클래스
// 같은 인터페이스를 구현한 다른 오브젝트에게 작업을 위임한다.
public class UserServiceTx implements UserService {
  UserService UserService;
  PlatformTransactionManager transactionManager;

  public void setTransactionManager(PlatformTransactionManager transactionManager) {
    this.transactionManager = transactionManager;
  }

  // UserService를 구현한 다른 오브젝트를 DI 받는다.
  public void setService(UserService userService) {
    this.userService = userService;
  }

  // DI 받은 UserService 오브젝트에 모든 기능을 위임한다.
  public void add(User user) {
    userService.add(user);
  }

  public void upgradeLevels() {
    TransactionStatus status = this.transactionManager.getTransaction(new DefaultTransactionDefinition());

    try {
      userService.upgradeLevels();

      this.transactionManager.commit(status);
    } catch (Exception e) {
      this.transactionManager.rollback(status);
      throw e;
    }
  }
}
```

### 트랜잭션 경계설정 코드 분리의 장점

1.	비즈니스 로직에서 트랜잭션과 같은 기술적인 내용을 전혀 신경 안써도 된다.
2.	비즈니스 로직에 대한 테스트를 쉽게 할 수 있다.

---

UserService가 동작하려면 세 가지 타입의 의존 오브젝트가 필요하다. 1. DB와 데이터를 주고 받는 UserDao 오브젝트 2. 메시지 발송을 하는 MailSender 오브젝트 3. 트랜잭션 처리를 담당하는 TransactionManager 오브젝트<br>

### 테스트의 대상이 환경이나, 외부 서버, 다른 클래스의 코드에 종속되고 영향을 받지 않도록 고립시킬 필요가 있다.

-	UserServiceImpl과 UserServiceTx를 분리하였기 때문에 고립된 테스트 대상으로 만들 수 있다.

```java
@Test
public void upgradeLevels() throws Exception {
  // DB 테스트 데이터 준비
  userDao.deleteAll();
  for (User user : users) userDao.add(user);

  // 메일 발송 여부 확인을 위해 Mock 오브젝트 DI
  MockMailSender mockMailSender = new MockMailSender();
  userServiceImpl.setMailSender(mockMailSender);

  // 테스트 대상 실행
  userServcie.upgradeLevels();

  // DB에 저장된 결과 확인
  checkLevelUpgraded(users.get(0), false);
  checkLevelUpgraded(users.get(1), true);
  checkLevelUpgraded(users.get(2), false);
  checkLevelUpgraded(users.get(3), true);
  checkLevelUpgraded(users.get(4), false);

  // Mock 오브젝트를 이용해서 메일 발송이 있었는지 확인
  List<String> request = mockMailSender.getRequests();
  assertThat(request.size(), is(2));
  assertThat(request.get(0), is(users.get(1).getEmail()));
  assertThat(request.get(1), is(users.get(3).getEmail()));
}

private void checkLevelUpgraded(User user, boolean upgraded) {
  User userUpdate = userDao.get(user.getId());
  ...
}
```

-	userDao와 DB가 직접 의존하고 있는 테스트 방식도 Mock 오브젝트를 이용해 바꿔보자

```java
static class MockUserDao implements userDao {
  private List<User> users;
  private List<User> updated = new ArrayList();

  private MockUserDao(List<User> users) {
    this.users = users;
  }

  public List<User> getUpdated() {
    return this.updated;
  }

  // 스텁 기능 제공
  public List<User> getAll() {
    return this.users;
  }

  // 목 오브젝트 기능 제공
  public void update(User user) {
    updated.add(user);
  }

  // 테스트에 사용되지 않는 메소드
  public void add(User user) { throw new UnsupportedOperationException(); }
  public void deleteAll() { throw new UnsupportedOperationException(); }
  public User get(String id) { throw new UnsupportedOperationException(); }
  public int getCount() { throw new UnsupportedOperationException(); }
}
```

-	UserDao 인터페이스를 구현하지 않기 위해 UnsupportedOperationException을 던진다.
-	스텁 기능 : 아직 개발되지 않은 코드를 임시로 대치하는 역할

```java
// MockUserDao를 사용해서 만든 고립된 테스트
@Test
public void upgradeLevels() throws Exception {
  UserServiceImpl userServiceImpl = new UserServiceImpl();

  // Mock 오브젝트로 만든 UserDao를 직접 DI 해준다.
  MockUserDao mockUserDao = new MockUserDao(this.users);
  userServiceImpl.setUserDao(mockUserDao);

  MockMailSender mockMailSender = new MockMailSender();
  userServiceImpl.setMailSender(mockMailSender);

  userServcie.upgradeLevels();

  // MockUserDao로 부터 업데이트 결과를 가져온다.
  List<User> updated = mockUserDao.getUpdated();
  // 업데이트 횟수와 정보를 확인한다.
  assertThat(updated.size(), is(2));
  checkUserAndLevel(updated.get(0), "joytouch", Level.SILVER);  
  checkUserAndLevel(updated.get(1), "madnite1", Level.GOLD);

  List<String> request = mockMailSender.getRequests();
  assertThat(request.size(), is(2));
  assertThat(request.get(0), is(users.get(1).getEmail()));
  assertThat(request.get(1), is(users.get(3).getEmail()));
}

private void checkUserAndLevel(User updated, String expectedId, Level expectedLevel) {
  assertThat(updated.get(), is(expectedId));
  assertThat(updated.getLevel(), is(expectedLevel));
}
```

-	DB에서 모든 사용자의 정보를 다시 가져와 일일이 확인해야 했지만 이제는 그럴 필요가 없다.

---

### 단위 테스트와 통합 테스트

-	단위 테스트 : 의존 오브젝트나 외부의 리소스를 사용하지 않도록 고립시킨 테스트
-	통합 테스트 : 두 개 이상의 다른 오브젝트가 연동하도록 만든 테스트 혹은 DB나 파일, 서비스 등의 리소스가 참여하는 테스트

### 테스트 방법 선택에 대한 가이드라인

1.	항상 단위테스트를 먼저 고려한다.
2.	외부와의 의존관계를 모두 차단한 테스트를 만든다.
3.	외부 리소스를 사용해야만 가능한 테스트는 통합테스트로 만든다.
4.	DAO는 DB까지 연동하는 테스트로 만드는 편이 효과적이다.
5.	DAO역할을 스텁이나 목 오브젝트로 대체해서 테스트 한다.
6.	단위 테스트를 충분히 거치면 통합 테스트의 부담이 줄어든다.
7.	단위 테스트가 너무 복잡하다면 처음부터 통합 테스트를 고려해 본다.
8.	스프링 테스트 컨텍스트 프레임워크를 이용하는 테스트는 통합 테스트다.

> 좋은 코드를 만들려는 노력을 게을리하면 테스트 작성이 불편해지고, 테스트를 잘 만들지 않게 될 가능성이 높아진다. 테스트가 없으니 과감하게 리팩토링할 엄두를 냊 못할 것이고 코드의 품질은 점점 떨어지고 유연성과 확장성을 잃어갈지 모른다.

---

단위 테스트를 만들기 위해서는 스텁이나 목 오브젝트의 사용이 필수적이다.

### Mockito 프레임워크

-	목 오브젝트를 편리하게 작성하도록 도와주는 프레임워크
-	사용 방법
	1.	인터페이스를 이용해 목 오브젝트를 만든다.
	2.	목 오브젝트가 리턴할 값이 있으면 이를 지정해준다.
	3.	테스트 대상 오브젝트에 DI해서 목 오브젝트가 테스트 중에 사용되도록 만든다.
	4.	목 오브젝트의 특정 메소드가 호출됐는지, 어떤 값을 가지고 몇 번 호출됐는지를 검증한다.

```java
@Test
public void mockUpgradeLevels() throws Exception {
  UserServiceImpl userServiceImpl = new UserServiceImpl();

  // 다이나믹한 목 오브젝트 생성, 메소드 리턴 값 설정, DI
  UserDao mockUserDao = mock(UserDao.class);
  when(mockUserDao.getAll()).thenReturn(this.users);
  userServiceImpl.setUserDao(mockUserDao);

  MailSender mockMailSender = mock(MailSender.class);
  userServiceImpl.setMailSender(mockMailSender);

  userServiceImpl.upgradeLevels();

  // 목 오브젝트가 제공하는 검증 기능
  verify(mockUserDao, times(2)).update(any(User.class)); // any : 파라미터 무시하고 호출된 횟수만 확인한다.
  verify(mockUserDao, times(2)).update(any(User.class));
  verify(mockUserDao).update(users.get(1));
  assertThat(users.get(1).getLevel(), is(Level.SILVER));
  verify(mockUserDao).update(users.get(3));
  assertThat(users.get(3).getLevel(), is(Level.GOLD));

  // 파라미터를 정밀하게 검사하기 위해 캡쳐할 수도 있다.
  ArgumentCaptor<SimpleMailMessage> mailMessageArg = ArgumentCaptor.forClass(SimpleMailMessage.class);
  verify(mockMailSender, times(2)).send(mailMessageArg.capture());
  List<SimpleMailMessage> mailMessages = mailMessageArg.getAllValues();
  assertThat(mailMessages.get(0).getTo()[0], is(users.get(1).getEmail()));
  assertThat(mailMessages.get(1).getTo()[0], is(users.get(3).getEmail()));
}
```

---

### 프록시

-	클라이언트의 요청을 받아주는 대리인과 같은 역할
-	사용 목적에 따라 두 가지로 구분한다.
	1.	타깃에 접근하는 방법을 제어
	2.	타깃에 부가적인 기능을 부여<br>

### 데코레이터 패턴

-	타깃에 부가적인 기능을 런타임 시 다이나믹하게 부여해주기 위해 프록시를 사용하는 패턴
-	프록시를 여러개 사용하여 단계적으로 위임하는 구조를 가질 수 있다.
-	각 데코레이터는 위임하는 대상에도 인터페이스로 접근하기 때문에, 위임의 대상이 최종 타깃인지 프록시인지 모른다.
-	그래서 데코레이터의 위임 대상을 인터페이스로 선언하고 런타임시에 주입받을 수 있도록 만든다. >데코레이터 패턴은 타깃의 코드를 손대지 않고, 클라이언트가 호출하는 방법도 변경하지 않은 채로 새로운 기능을 추가할 때 유용한 방법이다.<br>

### 프록시 패턴

-	프록시는 타깃의 기능을 확장하거나 추가하지 않는 대신, 클라이언트가 타깃에 접근하는 방식을 변경해준다.
-	타깃 오브젝트를 생성하기 복잡하거나 당장 필요없는 경우, 필요한 시점까지 생성하지 않는 편이 좋다.
-	프록시 메소드를 통해 타깃을 사용하려고 시도하면, 그때 프록시가 타깃 오브젝트를 생성하고 요청을 위임한다.
-	다른 서버의 존재하는 오브젝트를 사용해야 한다면, 원격 오브젝트에 대한 프록시를 만들어 사용한다.
-	타깃에 대한 접근권한을 제어하기 위해 사용할 수도 있다.
-	구조적으로 데코레이터와 유사하지만 프록시는 타깃 클래스의 정보를 알고 있는 경우가 많다.
-	생성을 지연하는 프록시라면 구체적인 생성방법을 알아야 하기 때문에 타깃에 직접적인 정보를 알아야 한다.<br>

### 다이나믹 프록시

-	프록시는 두 가지의 기능으로 구성된다.
	1.	타깃과 같은 메소드를 구현하고 있다가 메소드가 호출되면 타깃 오브젝트로 위임한다.
	2.	지정된 요청에 대해서는 부가기능을 수행한다.

```java
public class UserServiceTx implements UserService {
  // 타깃 오브젝트
  UserService userServcie;

  // 메소드 구현과 위임
  public void add(User user) {
    this.userService.add(user);
  }

  // 메소드 구현
  public void upgradeLevels() {
    TransactionStatus status = this.transactionManager.getTransaction(new DefaultTransactionDefinition());
    try {
      // 위임
      userService.upgradeLevels();

      // 부가기능 수행
      this.transactionManager.commit(status);
    } catch (RuntimeException e) {
      this.transactionManager.rollback(status);
      throw e;
    }
  }
}
```

<br>

### 프록시를 만들기가 번거로운 이유

1.	타깃의 인터페이스를 구현하고 위임하는 코드를 작성하기가 번거롭다.
	-	부가기능이 필요 없는 메소드도 구현해야한다.
	-	타깃 인터페이스의 메소드가 추가되거나 변경될 때마다 함께 수정해야한다.
2.	부가기능 코드가 중복될 가능성이 많다.

**이런 문제를 해결하는 데 유용한 것이 바로 JDK의 다이나믹 프록시다.**

---

### 리플렉션

-	다이나믹 프록시는 리플렉션 기능을 이용해서 프록시를 만들어준다.
-	리플렉션은 자바의 코드 자체를 추상화해서 접근하도록 만든 것이다.
-	리플렉션의 Method라는 인터페이스를 이용해서 메소드를 호출하는 방법
	1.	Class 타입의 정보를 가져온다.
	2.	이 클래스 정보에서 특정 이름을 가진 메소드 정보를 가져온다.
	3.	invoke() 메소드를 사용해 메소드를 실행한다.

```java
public class ReflectionTest {
  @Test
  public void invokeMethod() throws Exception {
    String name = "Spring";

    assertThat(name.length(), is(6));

    Method lengthMethod = String.class.getMethod("length");
    assertThat((Integer)lengthMethod.invoke(name), is(6));
  }
}
```

<br>

### 다이나믹 프록시를 이용해 프록시 만들기

-	다이나믹 프록시가 인터페이스 구현 클래스의 오브젝트를 만들어주지만, 프록시로서 필요한 부가기능 제공 코드는 직접 작성해야한다.
-	부가기능은 프록시 오브젝트와 독립적으로 InvocationHandler를 구현한 오브젝트에 담는다.
-	타깃 인터페이스의 모든 메소드 요청이 하나의 메소드로 집중되기 때문에 중복되는 기능을 효과적으로 제공할 수 있다.

```java
public class UppercaseHandler implements InvocationHandler {
  Hello target;

  public UppercaseHandler(Hello target) {
    this.target = target;
  }

  public object invoke(object proxy, Method method, Object[] args) throws Throwable {
    // 타깃으로 위임, 인터페이스의 메소드 호출에 모두 적용된다.
    String ret = (String)method.invoke(target, args);
    // 부가기능 제공
    return ret.toUpperCase();
  }
}
```

<br>

### 다이나믹 프록시의 확장

-	InvocationHandler 방식의 또 한가지 장점은 타깃의 종류에 상관없이도 적용이 가능하다는 점이다.

```java
public class UppercaseHandler implements InvocationHandler {
  Object target;

  public UppercaseHandler(Object target) {
    this.target = target;
  }

  public object invoke(object proxy, Method method, Object[] args) throws Throwable {
    String ret = method.invoke(target, args);
    if (ret instanceof String && method.getName().startsWith("say")) {
      return ((String)ret).toUpperCase();
    } else {
      return ret;
    }
  }
}
```

---

### 다이나믹 프록시를 위한 팩토리 빈

-	스프링은 내부적으로 리플렉션 API를 이용해서 빈 오브젝트를 생성한다.
-	하지만 다이나믹 프록시는 내부적으로 다이나믹하게 새로 정의해서 사용하기 때문에 미리 빈으로 등록할 방법이 없다.<br>

### 팩토리 빈

-	스프링을 대신해서 오브젝트의 생성로직을 담당하도록 만들어진 특별한 빈
-	일반적으로 private 생성자를 가진 클래스를 빈으로 등록하는 일은 권장되지 않는다.
-	스프링 빈에는 팩토리 빈과 UserServiceImpl만 빈으로 등록한다.

```java
// 트랜잭션 프록시 팩토리 빈
public class TxProxyFactoryBean implements FactoryBean<Object> {
  // TransactionHandler를 생성할 때 필요
  Object target;
  PlatformTransactionManager transactionManager;
  String pattern;
  // 다이나믹 프록시를 생성할 때 필요
  Class<?> serviceInterface;

  public void setTarget(Object target) {
    this.target = target;
  }

  public void setTransactionManager(PlatformTransactionManager transactionManager) {
    this.transactionManager = transactionManager;
  }

  public void setPattern(String pattern) {
    this.pattern = pattern;
  }

  public void setServiceInterface(Class<?> serviceInterface) {
    this.serviceInterface = serviceInterface;
  }

  // FactoryBean 인터페이스 구현 메소드
  // DI 받은 정보를 이용해서 다이나믹 프록시를 생성한다.
  public Object getObject() throws Exception {
    TransactionHandler txHandler = new TransactionHandler();
    txHandler.setTarget(target);
    txHandler.setTransactionManager(transactionManager);
    txHandler.setPattern(pattern);
    return Proxy.newProxyInstance(getClass().getClassLoader, new Class[] { serviceInterface }, txHandler);
  }

  // 다양한 타입의 프록시 오브젝트 생성에 재사용할 수 있다.
  public Class<?> getObjectType() {
    return serviceInterface;
  }

  // 매번 같은 오브젝트를 반환하지는 않는다.
  public boolean isSingleton() {
    return false;
  }
}
```

### 프록시 팩토리 빈 방식의 장점

1.	프록시 기법을 아주 빠르고 효과적으로 적용할 수 있다.
2.	타깃 인터페이스를 구현하는 클래스를 일일이 만드는 번거로움을 제거할 수 있다.
3.	하나의 핸들러 메소드를 구현하는 것만으로도 수많은 메소드에 부가기능을 부여가능하기 때문에 중복 문제도 해결된다.
4.	번거로운 다이나믹 프록시 생성 코드도 제거할 수 있다.
5.	DI 설정 만으로 다양한 타깃 오브젝트에 적용 가능하다.<br>

### 프록시 팩토리 빈의 한계

1.	한 번에 여러 개의 클래스에 공통적인 부가기능을 제공하는 것이 불가능하다.
	-	거의 비슷한 프록시 팩토리 빈의 설정이 중복되는 것은 막을 수 없다.
2.	하나의 타깃에 여러 개의 부가기능을 적용할 경우 어렵다.
	-	설정파일이 복잡해진다.
3.	TransactionHandler 오브젝트가 프록시 팩토리 빈 개수만큼 만들어진다.
	-	TransactionHandler는 타깃 오브젝트를 프로퍼티로 갖기 때문에 타깃 오브젝트가 변경되면 새로운 TransactionHandler 오브젝트를 만들어야 한다.

---

### 스프링의 프록시 팩토리 빈

-	스프링은 프록시 오브젝트를 생성해주는 기술을 추상화한 팩토리 빈을 제공해준다.<br>

### 어드바이스

-	스프링은 단순히 메소드 실행을 가로채는 방식 외에도 부가기능을 추가하는 여러 가지 방법을 제공한다.
-	타깃 오브젝트에 적용하는 부가기능을 담은 오브젝트를 스프링에서 어드바이스라 한다.
-	스프링의 ProxyFactoryBean은 인터페이스 자동검출 기능으로 타깃 오브젝트가 구현하고 있는 인터페이스 정보를 알아내고, 프록시를 만든다.<br>

### 포인트컷

-	메소드 선정 알고리즘을 담은 오브젝트
-	프록시는 클라이언트로부터 요청을 받으면 포인트컷에게 부가기능을 부여할 메소드인지를 확인해달라고 요청한다.
-	확인 받은 프록시는 MethodInterceptor 타입의 어드바이스를 호출한다.

### 어드바이저

-	어드바이저 = 포인트컷 + 어드바이스
