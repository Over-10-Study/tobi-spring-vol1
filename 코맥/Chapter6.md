# 6장 AOP
## 6.1 트랜잭션 코드의 분리
- 5장에서 살펴본 코드에서 가장 찜찜한 점은 ```UserService```에서 트랜잭션 경계설정 부분이다.
  - 비즈니스 로직과 트랜잭션 경계설정이 공존하고 있기 때문이다.

### 6.1.1 메서드 분리

```java
public void upgradeLevels() throws Exception {
  // 트랜잭션 경계설정
  TransactionStatus status = this.transactionManager
      .getTransaction(new DefaultTransactionDefinition());

  try {
    // 비즈니스 로직
    List<User> users = userDao.getAll();
    for (User user : users) {
      if (canUpgradeLevel(user)) {
        upgradeLevel(user);
      }
    }

    // 트랜잭션 경계설정
    this.transactionManager.commit(status);
  } catch (Exception e) {
    this.transactionManager.rollback(status);
    throw e;
  }
}
```

- 위는 비즈니스 로직과 트랜잭션 경계설정이 독립적으로 구분되어 있다.
  - 두 코드는 성격이 다르고 서로 주고받는 것도 없다.

```java
public void upgradeLevels() throws Exception {
  // 트랜잭션 경계설정
  TransactionStatus status = this.transactionManager
      .getTransaction(new DefaultTransactionDefinition());

  try {
    upgradeLevelsInternal();

    // 트랜잭션 경계설정
    this.transactionManager.commit(status);
  } catch (Exception e) {
    this.transactionManager.rollback(status);
    throw e;
  }
}

private void upgradeLevelsInternal() {
  // 비즈니스 로직
  List<User> users = userDao.getAll();
  for (User user : users) {
    if (canUpgradeLevel(user)) {
      upgradeLevel(user);
    }
  }
}
```

### DI를 이용한 클래스의 분리
- ```UserService``` 내부에 아직도 자리잡고 있는 트랜잭션 설정 부분을 밖으로 분리해보자.

1. DI 적용을 이용한 트랜잭션 분리
트랜잭션 설정 부분을 ```UserService```에서 분리하면 이를 사용하는 클라이언트에서는 트랜잭션이 없는 `UserService` 를 사용해야 한다.
- 직접 참조하는 코드의 전형적인 단점
- 이럴 때 간접으로 사용하기 위한 DI가 필요하다.

<그림 6-1>

- `UserService` 를 인터페이스로 분리하여 런타임시에 DI를 통해 적용한다.

<그림 6-2>

- 원래 우리가 해결하려는 문제는 비즈니스 로직과 트랜잭션 설정을 분리하는 것이다.
- `UserService` 는 트랜잭션 설정이 반드시 있어야 한다.

<그림 6-3>

2. `UserService` 인터페이스 도입

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

- 트랜잭션 코드를 분리한 결과 `UserService` 에는 단순히 비즈니스 로직만 남게된다.

3. 분리된 트랜잭션 기능

```java
// 위임 기능을 가진 UserServiceTx 클래스

public class UserServiceTx implements UserService {
  // UserService를 구현한 다른 오브젝트를 DI 받는다.
  UserService userService;

  public void setUserService(UserService userService) {
    this.userService = userService;
  }

  // DI 받은 UserService 오브젝트에 모든 기능을 위임한다.
  public void add(User user) {
    userService.add(user);
  }

  public void upgradeLevels() {
    userService.upgradeLevels();
  }
}
```

- `UserServiceTx` 는 비즈니스 로직을 갖고 있지 않는다.

```java
// 트랜잭션이 적용된 UserServiceTx

public class UserServiceTx implements UserService {
  UserService userService;
  PlatformTransactionManager transactionManager;

  public void setTransactionManager(PlatformTransactionManager transactionManager) {
    this.transactionManager = transactionManager;
  }

  public void setUserService(UserService userService) {
    this.userService = userService;
  }

  public void add(User user) {
    userService.add(user);
  }

  public void upgradeLevels() throws Exception {
    TransactionStatus status = this.transactionManager
        .getTransaction(new DefaultTransactionDefinition());

    try {
      upgradeLevelsInternal();

      this.transactionManager.commit(status);
    } catch (Exception e) {
      this.transactionManager.rollback(status);
      throw e;
    }
  }
}
```

<그림 6-4>

## 6.2 고립된 단위 테스트
현재 작성한 `UserService` 는 `UserDao`, `TransactionManager`, `MailSender` 를 의존하고 있으므로 서비스를 테스트하면 의존하고 있는 세 가지도 같이 테스트가 되는 상황이다.
- 서비스만을 위한 테스트를 하기 어렵다.
- 따라서 테스트의 대상이 환경, 외부 서버, 다른 클래스의 코드에 종속되고 영향을 받지 않도록 고립시켜야 한다.
- **테스트 대역을 사용한다.**

## 테스트 대상 오브젝트 고립시키기

```java
// upgradeLevels() 테스트

@Test
public void upgradeLevels() throws Exception {
  // 1. DB 테스트 데이터 준비
  userDao.deleteAll();
  for (User user : users) userDao.add(user);

  // 2. 메일 발송 여부 확인을 위해 목 오브젝트 DI
  // 테스트를 의존 오브젝트와 서버 등에서 고립
  MockMailSender mockMailSender = new MockMailSender();
  userServiceImpl.setMailSender(mockMailSender);

  // 3. 테스트 대상 실행
  userService.upgradeLevels();

  // 4. DB에 저장된 결과 확인
  // 최종 결과를 확인
  checkLevelUpgraded(users.get(0), false);
  ...

  // 5. 목 오브젝트를 이용한 결과 확인
  // 목 오브젝트를 통해 메일 발송 요청이 나간 적이 있는지만 확인
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

- 첫 번째와 네 번째에서 직접 DB에 접근하는 코드를 목 오브젝트로 대체해보자.

```java
// 사용자 레벨 업그레이드 작업 중에서 UserDao를 사용하는 코드

public void upgradeLevels() {
  // 업그레이드 후보 사용자 목록을 가져온다.
  List<User> users = userDao.getAll();
  for (User user : users) {
    if (canUpgradeLevel(user)) {
      upgradeLeve(user);
    }
  }
}

protected void upgradeLevel(User user) {
  user.upgradeLevel();
  // 수정된 사용자 정보를 DB에 반영한다.
  userDao.update(user);
  sendUpgradeEMail(user);
}
```

- 업그레이드 작업에서 DB를 사용하는 코드는 `getAll()`, `update()` 메서드 두 개이다.

```java
// UserDao 오브젝트

static class MockUserDao implements UserDao {
  // 레벨 업그레이드 후보 User 오브젝트 목록
  private List<User> users;
  // 업그레이드 대상 오브젝트를 저장해둘 목록
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

  // 테스트에 사용되지 않는 메서드
  // 만약을 위해서 익셉션을 날린다.
  public void add(User user) { throw new UnsupportedOperationException(); }
  public void deleteAll() { throw new UnsupportedOperationException(); }
  public User get(String id) { throw new UnsupportedOperationException(); }
  public int getCount() { throw new UnsupportedOperationException(); }
}
```

- 목 오브젝트를 사용하지 않을 때는 일일이 DB에 직접 저장해서 다시 가져와야 하지만,
- `MockUserDao` 오브젝트를 사용하면 **준비된 테스트용** 리스트를 메모리게 갖고 있다가 돌려주기만 하면 된다.

```java
// MockUserDao를 사용해서 만든 고립된 테스트

@Test
public void upgradeLevels() throws Exception {
  // 고립된 테스트에서는 테스트 대상 오브젝트를 직접 생성하면 된다.
  UserServiceImpl userServiceImpl = new UserServiceImpl();

  // 목 오브젝트로 만든 UserDao를 직접 DI 해준다.
  MockUserDao mockUserDao = new MockUserDao(this.users);
  userServiceImpl.serUserDao(mockUserDao);

  MockMailSender mockMailSender = new MockMailSender();
  userServiceImpl.setMailSender(mockMailSender);

  userService.upgradeLevels();

  // MockUserDoa로부터 업데이트 결과를 가져온다.
  List<User> updated = mockUserDao.getUpdated();
  assertThat(updated.size(), is(2));
  // 업데이트 횟수와 정보를 확인한다.
  checkUserAndLevel(updated.get(0), "joytouch", Level.SILVER);
  checkUserAndLevel(updated.get(1), "madnite1", Level.GOLD);

  List<String> request = mockMailSender.getRequests();
  assertThat(request.size(), is(2));
  assertThat(request.get(0), is(users.get(1).getEmail()));
  assertThat(request.get(1), is(users.get(3).getEmail()));
}

// id와 level을 확인하는 간단한 헬퍼 메서드
private void checkUserAndLevel(User updated, String expectedId, Level expectedLevel) {
  assertThat(updated.getId(), is(expectedId));
  assertThat(updated.getLevel(), is(expectedLevel));
}
```

- 위 테스트 결과 `UserService` 오브젝트는 외부 환경에서 완전히 고립돼서 테스트 된다.
  - DB에 실제로 저장되는지와 메일이 진짜 보내지는지는 중요하지 않다.
-고립된 테스트를 하면 테스트가 다른 의존 대상에 영향을 받을 경우를 대비해 복잡하게 준비할 필요도 없고, 테스트 수행 성능도 크게 향상된다.
  - 책에서 위 테스트는 목 오브젝트를 사용하기 전보다 사용한 후가 500배 정도 빠르다고 한다.

### 6.2.3 단위 테스트와 통합 테스트
- 단위 테스트: 테스트 대상 클래스를 목 오브젝트 등의 테스트 대역을 이용해 외부와 고립시켜서 테스트 하는 것
- 통합 테스트: 두 개 이상의 성격이나 계층이 다른 오브젝트를 연동하도록 만들어 테스트하거나, 외부의 DB나 파일, 서비스등의 리소스가 참여하는 테스트
  - 예를 들어 스프링의 테스트 컨텍스트 프레임워크를 이용해서 컨텍스트에서 생성되고 DI된 오브젝트를 테스트하는 것

단위 테스트와 통합 테스트 가이드 라인
- 항상 단위 테스트 먼저 고려
- 하나의 클래스나 성격과 목적이 같은 긴밀한 클래스 몇 개를 모아서 외부와의 의존관계를 모두 차단하고 필요에 따라 스텁이나 목 오브젝트 등의 테스트 대역을 통해 단위 테스트로 만든다.
- 외부 리소스를 사용해야만 가능한 테스트는 통합 테스트로 한다.
- 단위 테스트가 힘든 코드가 DAO이다.
  - 따라서 DAO는 직접 DB와 연동하는 통합 테스트를 기본적으로 실시한다.
  - DAO의 하나하나 메서드가 통합 테스트로 충분히 검증되면 목 오브젝트와 같은 테스트 대역으로 DAO 메서드를 사용-는 곳에서는 대체할 수 있다.
- 여러 개의 의존 관계가 있는 동작은 통합테스트를 한다.
- 단위 테스트를 만들기 너무 복잡하다면 통합 테스트를 처음부터 만들어본다.
- 스프링 테스트 컨텍스트 프레임워크를 이용하는 테스트는 통합 테스트이다.
  - 최대한 스프링의 지원 없이 테스트한다.
  - 스프링 자체 테스트도 필요하므로 이러한 통합 테스트가 필요할 때가 있다.

### 6.2.4 목 프레임워크
- 단위 테스트를 만들기 위해서는 스텁이나 목 오브젝트의 사용이 필수적이다.
- 목 오브젝트를 직접 만드는 일은 매우 번거롭다.
따라서, 여러 목 오브젝트를 편리하게 만드는 프레임워크가 존재한다.

1. Mockito 프레임워크
- Mockito의 특징은 목 클래스를 일일이 만들어줄 필요가 없다는 것이다.
  - 간단한 메서드 호출만으로 다이내믹하게 목 오브젝트를 만들어 준다.
- Mockito를 통해 만들어진 목 오브젝트는 메서드의 호출과 관련된 모든 내용을 자동으로 저장해두고, 검증할 수 있다.

```java
// 스태틱 메서드를 호출하여 목 오브젝트를 생성할 수 있다.
UserDao mockUserDao = mock(UserDao.class);

// getAll() 메서드가 불릴 때 스텁 기능을 추가한다.
when(mockUserDao.getAll()).thenReturn(this.users);

// 테스트를 진행하는 동안 mockUserDao의 update() 메서드가 두 번 호출됐는지 확인
verify(mockUserDao, times(2)).update(any(User.class));
```

Mockito 목 오브젝트 사용법은 다음과 같다.
- 인터페이스를 이용해 목 오브젝트를 만든다.
- 목 오브젝트가 리턴할 값이 있으면 이를 지정해준다. 메서드가 호출되면 예외를 강제로 던질수도 있다.
- 테스트 대상 오브젝트에 DI 해서 목 오브젝트가 테스트 중에 사용되도록 만든다.
- 테스트 대상 오브젝트를 사용한 후에 목 오브젝트의 특정 메서드가 호출됐는지, 어떤 값을 가지고 몇 번 호출됐는지 검증한다.

```java
// Mockito를 적용한 테스트 코드

@Test
public void mockUpgradeLevels() throws Exception {
  UserServiceImpl userServiceImpl = new UserServiceImpl();

  // 다이내믹한 목 오브젝트 생성과 메서드의 리턴 값 설정, 그리고 DI
  UserDao mockUserDao = mock(UserDao.class);
  when(mockUserDao.getAll()).thenReturn(this.users);
  userSuerviceImpl.serUserDao(mockUserDao);

  // 리턴 값이 없는 메서드를 가진 목 오브젝트는 더욱 간단함
  MaolSender mockMailSender = mock(MailSender.class);
  userServiceImpl.serMailSender(mockMailSender);

  userServiceImpl.upgradeLevels();

  // 목 오브젝트가 제공하는 검증 기능
  verify(mockUserDao, times(2)).update(any(User.class));
  verify(mockUserDao, times(2)).update(any(User.class));
  verify(mockUserDao).update(users.get(1));
  assertThat(users.get(1).getLevel(), is(Level.SILVER));
  verify(mockUserDao).update(users.get(3));
  assertThat(users.get(3).getLevel(), is(Level.GOLD));

  ArgumentCaptor<SimpleMailMessage> mailMessageArg = ArgumentCaptor.forClass(SimpleMailMessage.class);
  // 파라미터를 정밀하게 검사하기 위해 캡처 가능
  verify(mockMailSender, times(2)).send(mailMessageArg.capture());
  List<SimpleMailMessage> mailMessages = mailMessageArg.getAllValues();
  assertThat(mailMessages.get(0).getTo()[0], is(users.get(1).getEmail()));
  assertThat(mailMessages.get(1).getTo()[0], is(users.get(3).getEmail()));
}
```

## 6.3 다이내믹 프록시와 팩토리 빈
### 6.3.1 프록시와 프록시 패턴, 데코레이터 패턴
기존의 트랜잭션을 분리한 `UserService` 클래스는
- 부가 기능 `UserServiceTx`
- 핵심 기능 `UserServiceImpl`
로 나뉘었는데, 핵심 기능은 부가 기능(트랜잭션 기능)의 존재를 모르게 하기 위해 `UserServiceTx` 에서 `UserServiceImpl` 를 사용하는 방식으로 구현했다.
- 클라이언트는 부가 기능인 `UserServiceTx` 를 사용하도록 했다.
- 클라이언트가 핵심 기능을 사용한다면 부가 기능이 적용될 기회가 사라진다.

<그림 6-8>

이 문제를 해결하기 위해서는
- 클라이언트는 인터페이스를 통해서만 핵심기능을 사용하게 하고,
- 부가기능 자신도 같은 인터페이스를 구현한 뒤에 자신이 그 사이에 끼어들어야 한다.

<그림 6-9>

이렇게 자신이 마치 클라이언트가 사용하려고 하는 실제 대상인 것처럼 위장해서 클라언트의 요청을 받아주는 것을
- 대리자, 대리인과 같은 역할이라 하여 **프록시** 라고 한다.
그리고 프록시를 통해 최종적으로 요청을 위임받아 처리하는 실제 객체를 **타깃** 또는 **실체** 라고 한다.

<그림 6-10>

프록시의 특징은
- 타깃과 같은 인터페이스를 구현
- 프록시가 타깃을 제어할 수 있는 위치에 있음

프록시는 사용 목적에 따라 두 가지로 구분한다.
- 클라이언트가 타깃에 접근하는 방법을 제어
- 타깃에 부가적인 기능을 부여해주기 위함

1. 데코레이터 패턴
데코레이터 패턴은 타깃에 부가적인 기능을 런타임 시 다이내믹하게 부여해주기 위해 프록시를 사용하는 패턴이다.
- 데코레이터라는 이름은 내용물은 동일하지만 포장으로 부가적인 효과를 부여한다는 의미다.
- 프록시가 꼭 한 개로 제한되지는 않는다.
- 프록시가 직접 타깃을 사용하도록 고정시킬 필요도 없다.
- 프록시가 여러 개인만큼 순서를 정해서 단계적으로 위임하는 구조다.

예를 들어 소스코드를 출력하는 기능을 가진 핵심 기능이 있다. 여기서 여러 데코레이터를 추가할 수 있다.
- 라인넘버
- 문법에 따라 색 변경
- 특정 폭으로 소스를 자르거나 페이지 표시

<그림 6-11>

앞서 살펴본 `UserService` 도 데코레이터 패턴이 적용되었다.
- 데코레이터: `UserServiceTx`
- 타깃: `UserServiceImpl`

인터페이스를 통한 데코레이터 정의와 런타임 시의 다이내믹한 구성 방법은 스프링의 DI를 이용한다.

```html
<!-- 데코레이터 패턴을 위한 DI 설정 -->

<!-- 데코레이터 -->
<bean id="userService" class="springbook.user.service.UserServiceTx">
    <property name="transactionManager" ref="transactionManager" />
    <property name="userService" ref="userServiceImpl" />
</bean>

<!-- 타깃 -->
<bean id="userServiceImpl" class="springbook.user.service.ㅕserServiceImpl">
    <property name="userDao" ref="userDao" />
    <property name="mailSender" ref="mailSender" />
</bean>
```

- 데코레이터 패턴은 인터페이스를 통해 위임하므로 어느 데코레이터에서 타깃으로 연결될지 코드 레벨에선 미리 알 수 없다.
- **데코레이터 패턴은 타깃의 코드와 클라이언트 호출 방법을 수정하지 않고 새로운 기능을 추가할 때 유용하다.**

```html
<!-- 데코레이터 -->
<bean id="userService" class="springbook.user.service.UserServiceTx">
    <property name="transactionManager" ref="transactionManager" />
    <property name="userService" ref="userServiceDeco" />
</bean>

<!-- 데코레이터2 -->
<bean id="userServiceDeco" class="springbook.user.service.UserServiceDeco">
    <property name="userService" ref="userServiceImpl" />
</bean>

<!-- 타깃 -->
<bean id="userServiceImpl" class="springbook.user.service.UserServiceImpl">
    <property name="userDao" ref="userDao" />
    <property name="mailSender" ref="mailSender" />
</bean>
```

2. 프록시 패턴
일반적으로 사용하는 프록시라는 용어와 디자인 패턴에서 프록시 패턴은 구분해야한다.
- 프록시: 클라이언트와 사용 대상 사이에 대리 역할을 맡은 객체를 두는 방법을 총칭
- 프록시 패턴: 프록시를 사용하는 방법 중에서 타깃에 대한 접근 방법을 제어하려는 목적을 가진 경우

프록시 패턴의 프록시는 타깃의 기능을 확장하거나 추가하지 않는다.
- **타깃에 접근하는 방식을 변경한다.**

타깃 객체를 생성하기가 복잡하거나 당장 필요하지 않은 경우에는 꼭 필요한 시점까지 객체를 생성하지 않는 것이 좋다. 그런데 **타깃 객체에 대한 레퍼런스가 미리 필요할 수가 있다.** 이럴 때 프록시 패턴을 적용할 수 있다.
- 클라이언트에게 타깃에 대한 레퍼런스를 넘겨야 하는데,
- 실제 타깃 객체를 만드는 대신 프록시를 넘겨준다.
- 프록시의 메서드를 통해 타깃을 사용하려고 시도하면, 그때 프록시가 타깃 객체를 생성하고 요청을 위임한다.

원격 객체를 이용하는 경우에도 프록시를 사용하면 편리하다.
- 각종 리모팅 기술을 이용하여 다른 서버에 존재하는 객체를 사용해야 한다면
- 원격 객체에 대한 프록시를 만들어두고,
- 클라이언트는 마치 로컬에 존재하는 객체를 쓰는 것처럼 프록시를 사용할 수 있다.
- 프록시는 클라이언트의 요청을 받으면 네트워크를 통해
- 원격의 객체를 실행하고 결과를 받아서 클라이언트에게 돌려준다.

구조적으로 보면 데코레이터와 유사하지만,
- 프록시는 코드에서 자신이 만들거나 접근할 타깃 클래스 정보를 알고 있는 경우가 많다.

### 6.3.2 다이내믹 프록시
프록시 역시 목 객체와 같이 구현하는데 많은 번거로움이 있다.
- 프록시도 Mockito와 같은 프레임워크가 존재한다.

1. 프록시의 구성과 프록시 작성의 문제점
프록시는 다음의 두 가지 기능으로 구성된다.
- 타깃과 같은 메서드를 구현하고 있다가 메서드가 호출되면 타깃 객체로 위임한다.
- 지정된 요청에 대해서는 부가기능을 수행한다.

```java
// UserServiceTx 프록시의 기능 구분

public class UserServiceTx implements UserService {
  // 타깃 객체
  UserService userService;

  // 메서드 구현과 위임
  public void add(User user) {
    this.userService.add(user);
  }

  // 메서드 구현
  public void upgradeLevels() {
    // 부가기능 수행
    TransactionStatus status = this.transactionManager
      .getTransaction(new DefaultTransactionDefinition());

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

프록시의 역할은 위 코드에서도 볼 수 있듯이 위임과 부가작업이다. 그렇다면 프록시가 만들기 번거러운 이유는 무엇일까?
- 타깃의 인터페이스를 구현하고 위임하는 코드 작성
- 부가기능 코드 중복

위 문제를 해결하는 것이 다이내믹 프록시이다.

2. 리플렉션
다이내믹 프록시는 리플렉션 기능을 이용하여 프록시를 만든다.
- 리플렉션: 자바의 코드 자체를 추상화해서 접근한다.

자바의 모든 클래스는 그 클래스 자체의 구성정보를 담은 Class 타입의 객체를 하나씩 가지고 있다.
- `클래스이름.class` 또는 객체의 `getClass()` 메서드 호출하면
- 해당 객체의 메타 정보를 가져오거나 객체를 조작할 수 있다.

```java
// 예제
String name = "Spring";

// length() 메서드 정보 가져오기
// String 메서드 중에서 "length"라는 이름을 갖고, 파라미터가 없는 메서드
Method lengthMethod = String.class.getMethod("length");

// length() 메서드 실행하기
// Method 인터페이스에 정의된 invoke() 메서드 사용하기
int length = lengthMethod.invoke(name); // == int length = name.length();
```

```java
// 리플렉션 학습 테스트

public class ReflectionTest {
  @Test
  public void invokeMethod() throws Exception {
    String name = "Spring";

    // length()
    assertThat(name.length(), is(6));

    Method lengthMethod = String.class.getMethod("length");
    assertThat((Integer)lengthMethod.invoke(name), is(6));

    // charAt()
    assertThat(name.charAt(0), is('S'));

    Method charAtMethod = String.class.getMethod("charAt", int.class);
    assertThat((Character)charAtMethod.invoke(name, 0), is('S'));
  }
}
```

3. 프록시 클래스

```java
// Hello 인터페이스

interface Hello {
  String sayHello(String name);
  String sayHi(String name);
  String sayThankYou(String name);
}
```

```java
// 타깃 클래스

public class HelloTarget implements Hello {
  public String sayHello(String name) {
    return "Hello " + name;
  }

  public String sayHi(String name) {
    return "Hi " + name;
  }

  public String sayThankYou(String name) {
    return "Thank You " + name;
  }
}
```

```java
// 클라이언트 역할의 테스트

@Test
public void simpleProxy() {
  // 타깃은 인터페이스를 통해 접근하자.
  Hello hello = new HelloTarget();
  assertThat(hello.sayHello("Toby"), is("Hello Toby"));
  assertThat(hello.sayHi("Toby"), is("Hi Toby"));
  assertThat(hello.sayThankYou("Toby"), is("Thank You Toby"));
}
```

```java
// 프록시 클래스
// 데코레이터 패턴을 적용하여 부가기능 추가

public class HelloUppercase implements Hello {
  // 위임할 타깃 객체
  Hello hello;

  public HelloUppercase(Hello hello) {
    this.hello = hello;
  }

  public String sayHello(String name) {
    // 위임과 부가기능 적용
    return hello.sayHello(name).toUpperCase();
  }

  public String sayHi(String name) {
    return hello.sayHi(name).toUpperCase();
  }

  public String sayThankYou(String name) {
    return hello.sayThankYou(name).toUpperCase();
  }
}
```

이 프록시 적용은 두 가지 문제점이 있다.
- 인터페이스의 모든 메서드를 구현해 위임해야함
- 부가기능인 리턴 값을 대문자로 바꾸는 기능이 모든 메서드에 중복

4. 다이내믹 프록시 적용
다이내믹 프록시는 프록시 팩토리에 의해 런타임 시 다이내믹하게 만들어지는 객체다.

<그림 6-13>

- 클라이언트는 다이내믹 프록시 객체를 타깃 인터페이스를 통해 사용가능하다.
  - 프록시 팩토리에게 타깃 인터페이스 정보만 제공하면 해당 인터페이스를 구현한 클래스 객체를 자동으로 생성
- 프록시로서 필요한 부가기능은 직접 작성
  - `InvocatoinHandler` 를 구현한 객체에 담는다.
  - 클라이언트의 모든 요청을 리플렉션 정보로 변환하여 `invoke()` 메서드에 넘긴다.

```java
// InvocationHandler 구현 클래스

public class UppercaseHandler implements InvocationHandler {
  // 다이내믹 프록시로부터 전달받은 요청을 다시 타깃 객체에 위임해야하므로
  // 타깃 객체를 주입받는다.
  Hello target;

  public UppercaseHendler(Hello target) {
    this. target = target;
  }

  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    // 타깃으로 위임(인터페이스의 메서드 호출에 모두 적용됨)
    String ret = (String)method.invoke(target, args);
    // 리턴된 값은 다이내믹 프록시가 받아서 최종적으로 클라이언트에게 전달됨
    return ret toUpperCase();   // 부가기능
  }
}
```

```java
// 다이내믹 프록시 생성

// 생성된 다이내믹 프록시 객체는 Hello 인터페이스를 구현하고 있으므로
// Hello 타입으로 캐스팅해도 안전함
Hello proxiedHello = (Hello)Proxy.newProxyInstance(
        getCalss().getClassLoader(),    // 동적으로 생성되는 다이내믹 프록시 클래스의 로딩에 사용함
        new Class[] { Hello.class },    // 구현할 인터페이스
        new UppercaseHandler(new HelloTarget())); // 부가기능과 위임 코드를 담음
```

- 다이내믹 프록시는 한 번에 하나 이상의 인터페이스를 구현할 수 있다.
  - 인터페이스 배열 사용
- 아직까지는 큰 장점이 보이지 않는다.

5. 다이내믹 프록시의 확장
다이내믹 프록시는 직접 정의하는 것보다 훨씬 유연하고 많은 장점이 있다.
- `InvocationHandler` 방식은 타깃의 종류에 상관없이 적용 가능하다.
- 리플렉션의 Method 인터페이스를 사용하기 때문

어떤 종류의 인터페이스를 구현한 타깃이든 상관없이 재사용가능하고, 메서드의 리턴 타입이 스트링인 경우에만 대문자로 바꿔주기

```java
// 확장된 UppercaseHandler

public class UppercaseHandler implements InvocationHandler {
  // 어떤 종류의 인터페이스를 구현한 타깃에도 적용 가능
  Object target;
  private UppercaseHandler(Object target) {
    this.target = target;
  }

  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    // 호출한 메서드의 리턴 타입이 String인 경우에만 적용
    Object ret = method.invoke(target, args);
    if (ret instanceof String) {
      return ((String)ret).toUpperCase();
    } else {
      return ret;
    }
  }
}
```

### 6.3.3 다이내믹 프록시를 이용한 트랜잭션 부가기능
다이내맥 프록시를 이용하여 `UserServiceTx` 를 간단하게 구현해보자.

1. 트랜잭션 `InvocationHandler`

```java
// 다이내믹 프록시를 위한 트랜잭션 부가기능

public class TransactionHandler implements InvocationHandler {
  // 부가기능을 제공할 타깃 오브젝트. 어떤 타입의 오브젝트에도 적용 가능
  private Object target;
  // 트랜잭션 기능을 제공하는 데 필요한 트랜잭션 매니저
  private PlatformTransactionManager transactionManager;
  // 트랜잭션을 적용할 메서드 이름 패턴
  private String pattern;

  public void setTarget(Object target) {
    this.target = target;
  }

  public vodi setTransactionManager(PlatformTransactionManager transactionManager) {
    this.transactoinManager = transactionManager;
  }

  public void setPattern(String pattern) {
    this.pattern = pattern;
  }

  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    // 트랜잭션 적용 대상 메서드를 식별해서 트랜잭션 경계설정 기능을 부여
    if (method.getName().startWith(pattern)) {
      return invokeInTransaction(method, args);
    } else {
      return method.invoke(target, args);
    }
  }

  private Object invokeInTransaction(Method method, Object[] args) throws Throwable {
    TransactionStatus status = this.transactionManager.getTransaction(new DefaultTransactionDefinition());
    try {
      // 트랜잭션을 시작하고 타깃 오브젝트의 메서드를 호출, 예외가 없으면 커밋
      Object ret = method.invoke(target, args);
      this.transactionManager.commit(status);
      return ret;
    } catch (InvocationTargetException e) {
      // 예외가 발생하면 트랜잭션 롤백
      this.transactionManager.rollback(status);
      throw e.getTargetException();
    }
  }
}
```

- 트랜잭션이 필요한 모든 객체에 적용가능하다.
- 타깃 객체의 메서드 중 선별적으로 적용할 대상을 정한다.
- `Method.invoke()` 를 이용한 타깃 객체의 메서드를 호출할 때 발생하는 예외는 `InvocationTargetException` 로 한 번 포장돼서 전달된다.

2. `TransactionHandler`와 다이내믹 프록시를 이용하는 테스트

```java
// 다이내믹 프록시를 이용한 트랜잭션 테스트

@Test
public void upgradeAllOrNothing() throws Exception {
  ...
  TransactionHandler txHandler = new TransactionHandler();
  // 트랜잭션 핸들러가 필요한 정보와 객체를 DI 한다.
  txHandler.setTarget(testUserUservice);
  txHandler.setTransactionManager(transactionManager);
  txHandler.setPattern("upgradeLevels");

  // UserService 인터페이스 타입의 다이내믹 프록시 생성
  UserService txUserService = (UserService)Proxy.newProxyInstance(
    getClass().getClassLoader(), new Class[] { UserService.class }, txHandler
  );
}
```

### 6.3.4 다이내믹 프록시를 위한 팩토리 빈
이제 `TransactionHandler`와 다이내믹 프록시를 스프링의 DI를 통해 사용할 수 있도록 만들자.
- DI의 대상이 되는 다이내믹 프록시 객체는 일반적인 스프링의 빈으로 등록할 수 없다.

스프링의 빈 등록 과정은 기본적으로 **클래스 이름과 프로퍼티로** 정의된다.
- 스프링은 지정된 클래스 이름을 가지고 리플렉션을 통해 해당 클래스의 객체를 만든다.
- 클래스의 이름을 갖고 있다면 아래와 같이 리플랙션 API를 통해 객체를 만든다.

```java
Date now = (Date) Class.forName("java.util.Date").newInstance();
```

- 위의 `newInstance()` 메서드는 통해 파라미터가 없는 생성자를 호출하고, 이로 생성된 객체를 반환한다.

다이내믹 프록시 객체는 위와 같은 방식으로 객체를 생성하지 않는다.
- 클래스 자체가 다이내믹하게 새로 정의되기 때문에 어떤 클래스인지 알 수 없다.
- 다이내믹 프록시는 `Proxy` 클래스의 `newProxyInstance()` 라는 스태틱 팩토리 메서드를 통해 만든다.

1. 팩토리 빈
- 팩토리 빈: 스프링을 대신해서 객체의 생성로직을 담당하도록 만들어진 특별한 빈
- 팩토리 빈을 만드는 가장 간단한 방법은 `FactoryBean` 인터페이스를 구현하는 것이다.

```java
// FactoryBean 인터페이스

public interface FactoryBean<T> {
  // 빈 오브젝트를 생성해서 돌려준다.
  T getObject() throws Exception;
  // 생성되는 객체의 타입을 알려준다.
  Class<? extends T> getObjectType();
  // getObject()가 돌려주는 객체가 항상 같은 싱글톤 객체인지 알려준다.
  boolean isSingleton();
}
```

`FactoryBean` 인터페이스를 구현한 클래스를 스프링의 빈으로 등록하면 팩토리 빈으로 동작한다.

```java
// 생성자를 제공하지 않는 클래스

public class Message {
  String text;

  // 생성자가 private으로 선언되어 외부에서 생성자를 통해 객체를 만들 수 없다.
  private Message(String text) {
    this.text = text;
  }

  public String getText() {
    return text;
  }

  // 생성자 대신 사용할 수 있는 스태틱 팩토리 메서드
  public static Message newMessage(String text) {
    return new Message(text);
  }
}
```

private 생성자를 가진 클래스는 빈으로 등록해서 사용할 수 없다.
- 하지만 스프링은 private 생성자를 가지고 있더라도 리플렉션을 이용하여 객체를 생성한다.
- private 생성자를 가진 클래스는 일반적으로 스태틱 팩토리 메서드로 만들어져야하는 이유가 있다.
- 이를 무시하고 리플렉션으로 객체를 생성하는 것은 위험하다.

```java
// Message의 팩토리 빈 클래스

public class MessageFactoryBean implements FactoryBean<Message> {
  // 객체를 생성할 때 필요한 정보를 팩토리 빈의 프로퍼티로 설정해서
  // 대신 DI 받을 수 있게 한다.
  String text;

  public void setText(String text) {
    this.text = text;
  }

  // 실제 빈으로 사용될 객체를 직접 생성한다.(복잡한 방식의 객체 생성도 가능)
  public Message getObject() throws Exception {
    return Message.newMessage(this.text);
  }

  public Class<? extends Message> getObjectType() {
    return Message.class;
  }

  // getObject() 메서드가 반환하는 객체가 싱글톤인지 알려준다.
  // 이 팩토리 빈은 매번 요청할 때마다 새로운 오브젝트를 만들므로 false로 설정한다.
  // 이는 팩토리 빈의 동작방식에 관한 설정이고 만들어진 빈 오브젝트는 싱글톤으로 스프링이 관리한다.
  public boolean isSingleton() {
    return false;
  }
}
```

2. 팩토리 빈의 설정 방법

```html
<bean id="message" class="...spring.factorybean.MessageFactoryBean">
    <property name="text" value="Factory Bean" />
</bean>
```

일반적인 빈 설정과 다른 점은
- `message` 빈 오브젝트 타입이 설정된 `MessageFactoryBean`이 아니라 `Message` 타입이라는 것이다.
- 팩토리 빈 설정은 `getObjectType()`이 반환해주는 타입으로 결정된다.

3. 다이내믹 프록시를 만들어주는 팩토리 빈
팩토리 빈의 `getObject()` 메서드에 다이내믹 프록시 객체를 만들어주는 코드를 넣어주는 방식으로 다이내믹 프록시 객체를 생성한다.

```java
// 트랜잭션 프록시 팩토리 빈

// Object: 생성할 객체 타입을 지정할 수 있지만 범용적으로 해둠
public class TxProxyFactoryBean implements FactoryBean<Object> {
  // TransactionHandler를 생성할 때 필요
  Object target;
  PlatformTransactionManager transactionManager;
  String pattern;
  // 다이내믹 프록시를 생성할 때 필요(UserSerivce 외의 인터페이스 타깃에도 적용 가능)
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

  // FactoryBean 인터페이스 구현 메서드
  // DI 받은 정보를 이용해서 TransactionHandler를 사용하는 다이내믹 프록시를 생성
  public Object getObject() throws Exception {
    TransactionHandler txHandler = new TransactionHandler();
    txHandler.setTarget(target);
    txHandler.setTransactionManager(transactionManager);
    txHandler.setPattern(pattern);
    return Proxy.newProxyInstance(getClass().getClassLoader(), new Class[] { serviceInterface },
                txHandler);
  }

  public Class<?> getObjectType() {
    // 팩토리 빈이 생성하는 객체의 타입은 DI 받은 엔터페이스 타입에 따라 달라진다.(재사용 가능)
    return serviceInterface;
  }

  public boolean isSingleton() {
    return false;
    // 싱글톤이 아니라는 뜻이 아니라 getObject()가 매번 같은 객체를 리턴하지 않는다는 의미(?)
  }
}
```

<그림 6-15>

- 스프링 빈에는 팩토리 빈과 `UserServiceImpl`만 빈으로 등록되어 있다.
  - `UserServiceImpl`는 타깃 객체로서 `serviceInterface`에 DI 하기 위함

```html
<!-- UserService에 대한 트랜잭션 프록시 팩토리 빈 -->

<bean id="userService" class="...service.TxProxyFactoryBean">
    <property name="target" ref="userServiceImpl" />
    <property name="transactionManager" ref="transactionManager" />
    <property name="pattern" value="upgradeLevels" />
    <property name="serviceInterface" value="...service.UserService" />
</bean>
```

- `target`, `transactoinManager` 프로퍼티는 다른 빈을 가르키는 것이므로 `ref` 애트리뷰트로 설정
- `pattern`은 스트링으로 된 문자열이므로 `value` 사용해 값 지정
- `serviceInterface`는 Class 타입으로 이를 설정하려면 `value` 애트리뷰트를 통해 클래스 또는 인터페이스 이름을 넣어준다.

### 6.3.5 프록시 팩토리 빈 방식의 장점과 한계
1. 프록시 팩토리 빈의 재사용
타깃 객체를 변경해주는 것으로 트랜잭션이 필요한 여러 객체에 같은 다이내믹 프록시를 사용할 수 있다.
- 필요한 타깃 객체에 맞는 프로퍼티 정보를 설정해서 빈으로 등록하면 된다.
- 기존의 코드를 바꾸지 않고 부가적인 기능을 추가해줄 수 있다.

2. 프록시 팩토리 빈 방식의 장점
앞서 데코레이터 패턴이 적용된 프록시는 장점이 많음에도 불가하고 두 가지 문제점이 있었다.
- 프록시 클래스를 일일이 만들어야 하는 번거로움
- 부가적인 기능이 여러 메서드에 반복되어 코드 중복

프록시 팩토리 빈은 위 두 가지 문제점을 해결해준다.
- 다이내믹 프록시로 타깃 인터페이스를 구현하는 클래스를 일일이 만들 필요가 없다.
- 하나의 핸들러 메서드로 수많은 메서드에 부가기능을 부여할 수 있다.
- 팩토리 빈을 이용한 DI로 다이내믹 프록시 생성 코드도 제거할 수 있었다.

3. 프록시 팩토리 빈의 한계
하나의 클래스 안에 존재하는 여러 개의 메서드에 부가기능을 한 번에 제공하는 것은 가능하다.
- 하지만 여러 개의 클래스에 공통적인 부가기능을 제공하는 것은 불가능하다.
- 하나의 타깃에 여러 개의 부가기능을 적용할 때도 문제
- `TransactionHandler` 객체가 프록시 팩토리 빈 개수만큼 만들어진다.

## 6.4 스프링의 프록시 팩토리 빈
### 6.4.1 `ProxyFactoryBean`
스프링은 일관된 방법으로 프록시를 만들 수 있게 도와주는 추상 레이어를 제공한다.
- `ProxyFactoryBean`은 프록시를 생성해서 빈 오브젝트로 등록하게 해주는 팩토리 빈이다.
- `TxProxyFactoryBean`과 달리 순수하게 프록시를 생성하는 작업만을 담당하고 프록시를 통해 제공하는 부가기능은 별로의 빈에 둘 수 있다.

