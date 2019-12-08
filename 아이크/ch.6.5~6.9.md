# 6.5 스프링 AOP

지금까지 트랜잭션 코드를 깔끔하고 효과적으로 분리해냈다. 이렇게 분리해낸 트랜잭션 코드는 투명한 부가기능 형태로 제공돼야 한다.<br>

### 투명한 부가기능을 적용하기 위한 장애물 - 프록시 팩토리 빈 방식의 접근방법의 한계

1.	부가기능이 타깃 오브젝트마다 새로 만들어지는 문제 : 스프링 프록시 팩토리 빈의 어드바이스를 통해 해결

```java
public class TransactionAdvice implements MethodInterceptor {
  PlatformTransactionManager TransactionManager;

  public void setTransactionManager(PlatformTransactionManager TransactionManager) {
    this.TransactionManager = TransactionManager;
  }

  public Object invoke(MethodInvocation invocation) throws Throwable {
    TransactionStatus status = this.TransactionManager.getTransaction(new DefaultTransactionDefinition());

    try {
      // 콜백을 호출하여 타깃의 메소드를 실행한다.
      // 타깃 메소드 호출 전후로 필요한 부가기능을 넣을 수 있다.
      // 경우에 따라서 타깃이 이에 호출되지 않게 하거나 재시도를 위한 반복적인 호출도 가능하다.
      Object ret = invocation.proceed();
      this.TransactionManager.commit(status);
      return ret;
    } catch (RuntimeException e) {
      this.TransactionManager.rollback(status);
      throw e;
    }
  }
}
```

<br>

1.	부가기능의 적용이 필요한 타깃 오브젝트마다 거의 비슷한 내용의 설정정보를 추가해주어야 하는 문제<br> 
> 반복적인 프록시의 메소드 구현을 코드 자동생성 기법을 이용해 해결했다면 반복적인 프록시 팩토리 빈 설정 문제는 설정 자동등록 기법으로 해결할 수 없을까? 또는 실제 빈 오브젝트가 되는 것은 프록시 팩토리 빈을 통해 생성되는 프록시 그 자체이므로 프록시가 자동으로 빈으로 생성되게 할 수는 없을까?

### 빈 후처리기를 이용한 자동 프록시 생성기(BeanPostProcessor)

-	`DefaultAdvisorAutoProxyCreator` 사용
-	빈 후처리기 자체를 빈으로 등록해서 사용
-	빈 오브젝트가 생성될 때마다 빈 후처리기에 보내서 후처리 작업을 요청한다.
-	빈 오브젝트의 일부를 프록시로 포장하고, 프록시를 빈으로 대신 등록할 수 있다.
-	포인트컷이 담긴 어드바이저를 등록하고 빈 후처리기를 사용하면 자동으로 프록시가 적용

### 포인트 컷의 두 가지 기능

```java
public interface Pointcut {
  ClassFilter getClassFilter(); // 프록시를 적용할 클래스인지 확인
  MethodMatcher getMethodMatcher(); // 어드바이스를 적용할 메소드인지 확인
}
```

-	이전까지는 타깃이 정해져 있기 때문에 메소드 선별만 해주면 그만이었다.
-	빈 후처리기는 프록시를 적용할 클래스인지 판단할 수 있어야 한다.

### DefaultAdvisorAutoProxyCreator 적용

```java
/* 클래스 필터를 적용한 포인트컷 */
public class NameMatchClassMethodPointcut extends NameMethodPointcut {
  public void setMappedClassName(String mappedClassName) {
    this.setClassFilter(new SimpleClassFilter(mappedClassName)); // 모든 클래스를 다 허용하던 디폴트 클래스 필터를, 프로퍼티로 만든 클래스이름을 이용해서 필터를 만들어 덮어 씌운다.
  }

  static class SimpleClassFilter(String mappedName) {
    this.mappedName = mappedName;

    private SimpleClassFilter(String mappedName) {
      this.mappedName = mappedName;
    }

    public boolean mathces(Class<?> clazz) {
      return PatternMatchUtils.simpleMatch(mappedName, clazz.getSimpleName());
    }
  }
}
```

-	등록된 빈 중에서 Advisor 인터페이스를 구현한 것을 모두 찾는다.
-	생성되는 모든 빈에 대해 어드바이저의 포인트컷을 적용해보면서 프록시 적용 대상을 선정한다.
-	빈 클래스가 프록시 선정 대상이라면 프록시를 만들어 바꿔치기 한다.
-	`transactionAdvice`빈과 `transactionAdvisor`빈을 수정할 필요가 없어졌다.
-	프록시 팩토리 빈이 필요 없어졌다.<br>

---

### 포인트컷 표현식을 이용한 포인트컷

-	지금까지 사용했던 포인트컷은 메소드의 이름과 클래스의 이름 패턴을 각각 클래스 필터와 메소드 매처 오브젝트로 비교해서 선정하는 방식이었다.
-	이보다 더 복잡하고 세밀한 기준을 이용해 클래스나 메소드를 선정하게 하기 위해서 포인트컷 표현식을 이용한다.
-	`AspectJExpressionPointcut`은 클래스와 메소드의 선정 알고리즘을 포인트컷 표현식을 이용해 한 번에 지정할 수 있게 해준다.

```java
/* 포인트컷 테스트용 클래스 */
public class Target implements TargetInterface {
  public void hello() {}
  public void hello(String a) {}
  public int minus(int a, int b) throws RuntimeException { return e; }
  public int plus(int a, int b) { return 0; }
  public void method() {}
}
```

```java
/* 포인트컷 테스트용 추가 클래스 */
public class Bean {
  public void method() throws RuntimeException {
  }
}
```

### 포인트컷 표현식 문법

-	[] : 옵션항목이기 때문에 생략가능
-	| : OR
-	execution([접근제한자 패턴] 타입패턴 [ 타입패턴.]이름패턴 (타입패턴 | "..", ...) [throws 예외 패턴])

ex) `public int springbook.learningtest.spring.pointcut.Target.minus(int, int) throws java.lang.RuntimeException`

1.	public
	-	생략 가능
	-	조건부여 x
2.	int
	-	생략 불가
	-	\* : 모든타입
3.	`springbook.learningtest.spring.pointcut.Target`
	-	클래스 타입 패턴
	-	생략 가능
	-	패키지, 클래스, 인터페이스 이름에 \* 가능
4.	minus
	-	메소드 이름 패턴
	-	생략 불가
	-	\* : 모든 타입
5.	(int, int)
	-	메소드 파라미터의 타입 패턴
6.	`throws java.lang.RuntimeException`
	-	예외 이름에 대한 타입 패턴
	-	생략 가능

```java
@Test
public void methodSignaturePointcut() throws SecurityException, NoSuchMethodException {
  AspectJExpressionPointcut pointcut = new AspectJExpressionPointcut();
  pointcut.setExpression("execution(public int springbook.learningtest.spring.pointcut.Target.minus(int, int) throws java.lang.RuntimeException)");

  // Target.minus();
  assertThat(pointcut.getClassFilter().matches(Target.class) && pointcut.getMethodMatcher().matches(Target.class.getMethod("minus", int.class, int.class), null), is(true));

  //Bean.method();
  assertThat(pointcut.getClassFilter().matches(Bean.class) && pointcut.getMethodMatcher().matches(Target.class.getMethod("method"), null), is(false));
}
```

```java
/* 포인트컷과 메소드를 비교해주는 테스트 헬퍼 메소드 */
public void pointcutMatches(String expression, Boolean expected, Class<?> clazz, String methodName, Class<?> ... args) throws Exception {
  AspectJExpressionPointcut pointcut = new AspectJExpressionPointcut();
  pointcut.setExpression(expression);

  // 포인트컷의 클래스 필터와 메소드 메처 두 가지를 동시에 만족하는지 확인한다.
  assertThat(pointcut.getClassFilter().matches(clazz) && pointcut.getMethodMatcher().matches(clazz.getMethod(methodName, args), null), is(expected));
}
```

```java
/* 타깃 클래스의 메소드 6개에 포인트컷 선정 여부를 검사하는 헬퍼 메소드 */
public void targetClassPointcutMatches(String expression, boolean... expected) throws Exception {
  pointcutMatches(expression, expected[0], Target.class, "hello");
  pointcutMatches(expression, expected[1], Target.class, "hello", String.class);
  pointcutMatches(expression, expected[2], Target.class, "plus", int.class, int.class);
  pointcutMatches(expression, expected[3], Target.class, "minus", int.class, int.class);
  pointcutMatches(expression, expected[4], Target.class, "method");
  pointcutMatches(expression, expected[5], Bean.class, "method");
}
```

-	장점 : 로직이 짧은 문자열에 담긴다. 때문에 클래스나 코드를 추가할 필요가 없어서 코드와 설정이 단순해진다.
-	단점 : 런타임 시점까지 문법의 검증이나 기능 확인이 되지 않는다.

##### 포인트컷 표현식이 execution(\* \*..\*ServiceImpl.upgrade\*(..))로 되어 있는데, TestUserService 클래스로 등록된 빈이 선정된 이유?

-	포인트컷 표현식의 클래스 이름에 적용되는 패턴은 클래스 이름 패턴이 아닌 타입패턴이기 때문에
-	`TestUserService`클래스이자, 슈퍼클래스인 `UserServiceImple`, 구현 인터페이스인 `UserService`세 가지 모두 적용된다.

---

UserService에 트랜잭션을 적용해온 과정을 정리해보자.

1.	트랜잭션 서비스 추상화
	-	특정 트랜잭션과 관련한 코드를 수정하면 직접 관련없는 많은 코드를 수정해야한다.
	-	런타임 시에 다이나믹하게 연결해주는 DI방식을 이용해 해결한다.
2.	프록시와 데코레이터 패턴
	-	여전히 비즈니스 로직에 트랜잭션을 적용하고 있다.
	-	DI를 이용해 데코레이터 패턴을 적용하여 해결한다.
3.	자동 프록시 생성 방법과 포인트컷
	-	트랜잭션 적용 대상이 되는 빈을 일일이 프록시 팩토리 빈을 설정해줘야한다.
	-	빈 후처리기를 이용하여 해결한다.
	-	포인트컷 표현식 사용해서 적용 대상 선택
4.	부가기능의 모듈화
	-	트랜잭션과 같은 부가기능은 핵심기능과 같은 방식으로는 모듈화하기가 매우 힘들다.
	-	부가기능은 여타 핵심기능과 같은 레벨에서는 독립적으로 존재하는 것 자체가 불가능하다.

---

### 애스펙트

-	부가될 기능을 정의한 코드인 어드바이스와, 어드바이스를 어디에 적용할지를 결정하는 포인트컷을 가진다.
-	어드바이저는 단순한 형태의 애스펙트

### AOP

-	애플리케이션의 핵심적인 기능에서 부가적인 기능을 분리해서 애스펙트라는 독특한 모듈로 만들어서 설계하고 개발하는 방법
-	관점 지향 프로그래밍

### AOP 적용 기술

1.	프록시를 이용한 AOP
	-	메소드 호출 과정에 참여해서 부가기능을 제공한다.
	-	타깃 메소드를 호출하는 전후에 다양한 부가기능을 제공할 수 있다.
2.	바이트코드 생성과 조작을 통한 AOP
	-	뭔소린지 모르겠다.

### AOP 용어 정리

-	타깃
	-	부가기능을 부여할 대상
	-	다른 부가기능을 제공하는 프록시 오브젝트도 가능
-	어드바이스
	-	타깃에게 제공할 부가기능을 담은 모듈
	-	`MethodInterceptor`등
-	조인 포인트
	-	어드바이스가 적용될 수 있는 위치
	-	스프링 프록시 AOP에서는 메소드의 실행 단계
	-	타깃 오브젝트가 구현한 인터페이스의 모든 메소드는 조인 포인트
-	포인트컷
	-	어드바이스를 적용한 조인 포인트를 선별하는 작업 또는 그 기능을 정의한 모듈
-	프록시
	-	클라이언트와 타깃 사이에 투명하게 존재하면서 부가기능을 제공하는 오브젝트
	-	스프링은 프록시를 이용해 AOP를 지원한다.
-	어드바이저
	-	어떤 부가기능(어드바이스)을 어디에(포인트컷) 전달할 것인가를 알고있는 AOP의 기본적인 모듈
-	애스펙트
	-	한 개 또는 그 이상의 포인트컷과 어드바이스의 조합으로 만들어진다.
	-	싱글톤 형태의 오브젝트로 존재

### 프록시 방식 AOP를 적용하기 위한 네 가지 빈

1.	자동 프록시 생성기
	-	`DefaultAdvisorAutoProxyCreator`클래스를 등록
	-	DI되지도 DI하지도 않는다.
	-	애플리케이션 컨텍스트가 빈 오브젝트를 생성하는 과정에 빈 후처리기로 참여한다.
	-	빈으로 등록된 어드바이저를 이용해 프록시를 자동으로 생성하는 기능을 담당한다.
2.	어드바이스
	-	부가기능을 구현한 클래스를 등록
3.	포인트컷
	-	`AspectJExpressionPointcut`을 등록
	-	expression 프로퍼티에 포인트컷 표현식을 넣어주면 된다.
4.	어드바이저
	-	`DefaultPointcutAdvisor`클래스를 등록
	-	어드바이스와 포인트컷을 프로퍼티로 참조하는 기능

---


# 6.6 트랜잭션 속성

트랜잭션이라고 모두 같은 방식으로 동작하는 것은 아니다.`DefaultTransactionDefinition`인터페이스는 트랜잭션의 동작방식에 영향을 줄 수 있는 네 가지 속성을 정의한다.

1.	트랜잭션 전파
	-	`PROPAGATION_REQUIRED`
		-	진행 중인 트랜잭션이 없으면 새로 시작
		-	이미 시작된 트랜잭션이 있으면 참여
		-	`A`, `B`, `A->B`, `B->A`
	-	`PROPAGATION_REQUIRED_NEW`
		-	항상 새로운 트랜잭션을 시작한다.
	-	`PROPAGATION_NOT_SUPPORTED`
		-	트랜잭션 없이 동작하도록 트랜잭션 무시
		-	특정 메소드만 트랜잭션 적용에서 제외시키려 할 때
	-	트랜잭션을 시작하려고 할 때 `getTransaction()`을 하는 이유
		-	트랜잭션 전파속성을 포함하고 있어서
2.	격리수준
	-	모든 DB 트랜잭션은 격리 수준을 갖고 있어야한다.
	-	적절하게 격리수준을 조절해서 가능한 많은 트랜잭션을 동시에 진행시키면서도 문제가 발생하지 않게 제어가 필요하다.
	-	기본적으로 디폴트 격리수준
3.	제한시간
	-	트랜잭션을 수행하는 제한시간 설정
	-	디폴트는 제한시간 없음
4.	읽기전용
	-	`readOnly`로 데이터 조작 방지 및 엑세스 성능 향상

---

### 선택적으로 트랜잭션을 정의하려면?

-	`TransactionInterceptor`어드바이스를 이용해보자
-	`TransactionAttribute`인터페이스 `TransactionDefinition`의 네 가지 기본 항목 + `rollbackOn()`
-	`TransactionAdvice`는 런타임 예외에만 트랜잭션을 처리한다.
-	`TransactionInterceptor`는 런타임 예외가 발생하면 트랜잭션 롤백, 체크 예외일 경우 트랜잭션 커밋
-	`TransactionAttribute`의 `rollbackOn()`을 통해 체크 예외일 때 롤백, 런타임 예외 시 커밋 가능

---

### 포인트컷과 트랜잭션 속성의 적용 전략

-	트랜잭션 부가기능을 적용할 후보 메소드를 선정하는 작업은 포인트컷에 의해 진행된다.
-	어드바이스의 트랜잭션 전파 속성에 따라서 메소드별로 트랜잭션의 적용 방식이 결정된다.
-	포인트컷 표현식과 트랜잭션 속성을 정의할 때 따르면 좋은 몇 가지 전략을 생각해보자.

-	타깃 클래스의 메소드는 모두 적용 후보

-	단순한 조회 작업만 하는 메소드는 모두 적용(읽기 전용)

-	트랜잭션용 포인트컷 표현식에는 메소드나 파라미터, 예외에 대한 패턴을 정의하지 않는 게 바람직하다.

-	클래스보다는 인터페이스 타입을 기준으로 타입 패턴을 적용하는 것이 좋다.(클래스에 비해 변경 빈도가 적고, 일정한 패턴을 유지하기 쉽다.)

-	포인트컷 표현식 대신 빈표현식도 고려해볼 만하다. ex) `bean(*Service)`

-	너무 다양하게 트랜잭션 속성을 부여하면 관리하기 힘들다.

### 프록시 방식 AOP는 같은 타깃 오브젝트 내의 메소드를 호출할 때는 적용되지 않는다.
<img width="593" alt="토비의_스프링_3_1_Vol_1_pdf_526_881페이지_" src="https://user-images.githubusercontent.com/34293961/65859143-7e9df900-e3a2-11e9-976a-b74af12a62d8.png">

- 해결 방법
  1. 스프링 API를 이용해 프록시 오브젝트에 대한 레퍼런스를 가져온 뒤에 같은 오브젝트의 메소드 호출도 프록시를 이용하도록 강제한다.(비추)
	2. AspectJ와 같은 타깃의 바이트코드를 직접 조작하는 방식의 AOP 기술을 적용하는 것.(14장에서 설명)

### 트랜잭션 경계설정의 일원화

-	비즈니스 로직을 담고 있는 서비스 계층 오브젝트의 메소드가 트랜잭션 경계를 부여하기에 가장 적절하다.
-	다른 계층이나 모듈에서 DAO에 직접 접근하는 것은 차단한다.(서비스 계층을 통한 접근)

---

### 애노테이션 트랜잭션 속성과 포인트컷

-	클래스나 메소드에 따라 제각각 속성이 다른, 세밀하게 튜닝된 트랜잭션 속성을 적용해야 하는 경우도 있다.
-	설정파일에서 일괄적으로 속성을 부여하는 대신에 직접 타깃에 트랜잭션 속성정보를 가진 애노테이션을 지정하는 방법이 있다.

### `@Transactional`

-	메소드, 클래스, 인터페이스에 사용 가능
-	기본적으로 트랜잭션 속성을 정의하는 것이지만, 동시에 포인트컷의 자동등록에도 사용된다.
-	메소드마다 다르게 설정할 수도 있으므로 매우 유연한 트랜잭션 속성 설정이 가능해진다.
-	트랜잭션 부가기능 적용 단위는 메소드(단점 : 동일한 속성 정보에 대해 반복적으로 애노테이션 부여)

### 대체 정책

-	스프링은 `@Transactional`을 적용할 때 4단계의 대체 정책을 이용하게 해준다.
-	가장 먼저 발견되는 속성정보를 사용
	1.	타깃 메소드
	2.	타깃 클래스
	3.	선언 메소드
	4.	선언 타입(클래스, 인터페이스)

```java
[1]
public interface Service {
  [2]
  void method1();
  [3]
  void method2();
}
[4]
public class ServiceImpl implements Service {
  [5]
  public void method1() {

  }
  [6]
  public void method2() {

  }
}
// 1. 타깃 메소드 -> [5], [6]
// 2. 타깃 클래스 -> [4]
// 3. 선언 메소드 -> [2], [3]
// 4. 선언 타입 -> [1]
```

### `@Transactional`잘 사용하기

1.	타입 레벨에 정의되고 공통 속성을 따르지 않는 메소드에 대해서만 부여
2.	타깃 클래스보다는 인터페이스에 두는게 바람직하다.(구현 클래스가 바뀌어도 트랜잭션 속성 유지 가능)
3.	프록시 방식의 AOP가 아닌 방식으로 사용할 경우 타깃 클래스에 두자.
4.	프록시 AOP 잘 모르겠으면 맘편하게 타깃 클래스와 타깃 메소드에 적용하자.
5.	트랜잭션 적용에 대해 파악하기 어렵기 때문에 사용할 때 실수하지 않도록 주의하고, 별도의 코드 리뷰를 거칠 필요가 있다.

```java
@Transactional
public interface UserService {
  void add(User user);
  void deleteAll();
  void update(User user);
  void upgradeLevels();

  @Transactional(readOnly=true)
  User get(String id);

  @Transactional(readOnly=true)
  List<User> getAll();
}
```

---

### 트랜잭션 지원 테스트

-	테스트 내에서 트랜잭션을 제어할 수 있는 네 가지 속성
	1.	`@Transactional`
	2.	`@Rollback`
	3.	`@TransactionConfiguration`
	4.	`@NonTransactional`, `Propagation.NEVER`
-	DB가 사용되는 통합 테스트를 별도의 클래스로 만들어둔다면 기본적으로 클래스 레벨에 부여
-	DB가 사용되는 통합 테스트는 가능한 롤백 테스트로 만드는 것이 좋다.
-	테스트가 기본적으로 롤백 테스트로 되어 있다면 테스트 사이에 서로 영향을 주지 않으므로 독립적이고 자동화된 테스트로 만들기가 매우 편하다.
