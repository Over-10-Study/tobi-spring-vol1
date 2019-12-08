JDK의 팩토리빈의 한계: 부가기능이 타깃 오브젝트마다 새로 만드러지는 문제는 스프링 ProxyFactoryBean의 어드바이스를 통해 해결 : DI덕분!!

하지만 무수한 xml설정의 늪에서는 아직 못벗어났다! ==> DefaultAdviosrAutoProxyCreator

DefulaTAdvisorAutoProxyCreator : 빈후처리기. 빈 후처리기를 스프링에 적용하는 방법은 간단하다. 빈 후처리기 자체를 빈으로 등록하는 것이다.
스프링은 빈 후처리가 빈으로 등록되어 있으며 빈 오브젝트가 생성될 때마다 빈후처리기에 보내서 후처리 작업을 요청한다.

빈후처리기능 동작설명:
1. 빈 후처리가 등록되어 있으면 스프링은 빈오브젝트를 만들 때마다 후처리기에게 빈을 보낸다.
2. DefaultAdvisorAutoProxyCreator는 빈으로 등록된 모든 오드바이저 내의 포인트컷을 이용해 전달받은 빈이 프록시 적용대상인지 확인
3. 프록시 적용 대상이면 재당된 프록시 생성기에 현재 빈에 대한 프록시를 ㅁ나들게 하고, 만들어진 프록시에 어드바ㅣ저를 연결해준다.
4. 빈 후처리기는 프록시가 생성되면 원래 컨테이너가 전달해준 빈 오브잭트 대신 프록시 오브젝트를 컨테이너에게 돌려준다.
5. 컨테이너는 최종저그오 빈 후처리기가 돌려준 오브젝트를 빈으로 들고하고 사용

적용할 빈을 선정하는 로직이 추가된 포인트컷이 담긴 어드바이저를 등록하고 빈후처리기를 사용하면 일일이 ProxyFacoyBean 빈을 등록하지 않아도 타깃오브젝에
자동으로 프록시가 적용된다. (-_-;; ProxyFactoryBean왜 이렇게 열심히 배움 ㅠㅠ)

Pointcut 선정 기능을 모두 적용한다면 먼저 프록시를 적용할 클래스인지 판단하고 나서, 적용 대상 클래스인 경우에는 어드바이스를 적용할 메소드인 확인하는 식으로 동작한다. 

```java
@Test
public void classNamePointcutAdvisor() {
    NameMatchMethodPointcut classMethodPointCut = new NameMatchMethodPointcut() {
        public ClassFilter getClassFilter() {
            return new ClassFilter() {
                public bollean matches(Class<?> clazz) {
                    return clazz.getSimpleName().startsWith("HelloT*");
                }
            };
        }
    };
    classMethodPointcut.setMappedName("sayH*");
    checkAdviced(new HelloTarget(), classMethodPointcut, true);

    class HelloWorld extends HelloTarget {};
    checkAdviced(new HelloWorld(), classMethodPointcut, false);

    class HelloToby extends HelloTarget {};
    checkAdviced(new HelloToby(), classMethodPointcut, true);
}

private void checkAdviced(Object target, Pointcut pointcut, boolean adviced) {
    ProxyFactoryBean pfBean = new PrxoyFactoryBean();
    pfBean.setTarget(target);
    pfBean.addAdvisor(new DefuaultPointcutAdvisor(pointcut, new UppercaseAdvice()));
    Hello proxedHello = (Hello) pfBean.getObject();

    if (adviced) {
        assertThat(proxiedHello.sayHello("Toby"), is ("HELLO TOBY"));
        assertThat(proxiedHello.sayHi("Tobby"), is ("HI TOBY"));
        assertThat(prxiedHello.sayThankYou("Toby"), is("Thank You Toby"));
    }

    else {
        assertThat(proxiedHello.sayHello("Toby"), is("Hello Toby"));
        assertThat(proxiedHello.sayHi("Toby"), is("Hi Toby"));
        assertThat(proxiedHello.sayThankYou("Toby"), is ("Thanks you Toby");)
    }
}
```

```java
public class NameMatchClassMethodPointcut extends NameMatchMethodPointcut {
    public void setMappedClassName(String mappedClassName) {
        this.setClassFilter(new SimpleClassFilter(mappedClassName));
    }
    static class SimplClassFilter implements ClassFilter {
        String mappedName;

        private SimpleClassFilter(String mappedNAme) {
            this.mappedName = mappedName;
        }

        public boolean mathches(Class<?> clazz) {
            return PatternMatchUtils.simpleMatch(mappedName, clazz.getSimpleName());
        }
    }
}
```
자동 프록시 생성기 : DefaultAdvisorAutoProxyCreator 작동 순선
1. 등록된 빈 중에서 Advisor 인터페이스를 구현한 것을 모두 찾는다.
2. 생성되는 모든 빈에 대해 어드바이저의 포인트컷을 적용해보면서 프록시 적용 대상을 선정
3. 선정대상이라면 프록시를 만들어 원래 빈 오브젝트와 바꿔치기한다.
4. 원래 빈 오브젝트는 프록시 뒤에 연결돼서 프록시를 통해서만 접근 가능하게 바뀌는 것이다
5. 따라서 타깃 빈에 의존한다고 정의한 다른 빈들은 프록시 오브젝트를 대신 DI 박데 되는 것이다

```java
<bean class= "org.springframework.aop.framework.autoproxy.DefaultAdvisorAutoProxyCreator" />
//다른 빈에서 참조되거나 코드에서 빈 이름으로 조회될 필요가 없는 빈이라면 아이디를 등록하지 않아도 무방하다.
<bean id="transactionPointcut" class="springbook.service.NameMatchClassMEthodPointcut">
    <property name="mappedClassName" value="*ServiceImple"/>
    <property name ="mappedName" value="upgrade*"/>
</bean>
```
##자동프록시 생성기와 테스트
자동프록시 생성기를 쓰면서 생긴 문제점:
ProxyFactoryBean를 빈으로 등록해서 사용했을 때는 TestUserService를 수동DI로 해주어서 TestUserService를 사용했는ㄷ
자동프록시 생성기는 그런 작업을 아예 없애 버렸다 (DefaultAdvisorProxyCreator를 사용하면 자동으로 연결해준다 다이나믹하게)
그래서 생긴 문제점들:
1. TestUserService가 UserServiceTest 클래스 내부에 정의된 스태틱 클래스라는 점
2. 포인트컷이 트랜잭션 어드바이스를 적용해주는 대상 클래스의 이름 패턴이 *ServiceImple이라고 되어 있어서 TestUserService 클래스는 빈으로 등록을 해도 포인트컷이 프록시 적용대상으로 선정해주지 않는다는 점이다

해결책:
1. 스태틱 클래스도 빈등록 하는데 아무 문제 없다
2. TestUserService ==> TestUserServiceImpl로 바꾼다.
```java
static class TestUserServiceImple extends UserServiceImple {
    private String id = "madnite1"

    protected void upgradeLevel(User user) {
        if (user.getId().equals(this.id) throw new TestUserServiceException();
        super.upgradeLevel(user);
    }
}

<bean id="testUserService" class="springbook.user.service.UserServiceTest$TestUserServiceImple" parent="userService" />
//$스태틱멤버 클래스는 $로 지정한다.
//프로퍼티 정의를 포함해서 userService빈의 설정을 상속받는다. 
```
```java
public class UserServiceTest {
    @Autowired UserService userService;
    @Autowired UserService testUserService; 
}

@Test
public void upgradeAllOrNothing() {
    userDao.deleteAll();
    for (User user : users) userDao.add(user);

    try {
        this.testUserService.upgradeLevels();
        fail("TestUserServiceException expected")
    } catch(TestUserServiceExcpeion e) {

    }
    checkLevelUpgraded(users.get(1), false);
}
```

##AOP란 무엇인가

서비스 추상화:
특정 로우 레벨의 기술에 종속되는 코드가 되어 버린다. (JDBC, JTA 등등)
예를 들어 JDBC의 로컬 트랜잭션 방식을 적용하던 것을 JTA에 적용하는 순간 모든 트랜잭션 적용 코드를 수정한다

==> 트랜잭션 적용이라는 추상적인 작업 내용은 유지한 채로 구체적인 구현 방법을 자유롭게 바꿀 수 있도록 ==> 서비스추상화

구체적인 구현 내용을 담은 의존 오브젝트는 런타임 시에 다이내믹하게 연결해준다는 DI를 활용한 전형적인 접근 방법.

트랜잭션 추상화란 결국 인터페이스와 DI를 ㅗㅇ해 무엇을 하는지는 남기고, 그것을 어떻게 하는지를 분리한 것이다. 어떻게 할지는 더 이상 비즈니스 로직 코드에는 영향을 주지 않고 독립적으로 변경할 수 있게 됐다.

프록시와 데코레이터 패턴:
하지만 여전히 비즈니스 로직 코드에는 트랜잭션을 적용하고 있다는 사실은 여전히 드러나 있다. 트랜잭션이라는 부가적인 기능을 어디에
적용할 것인가는 여전히 코드에 노출시켜야 했다. 

클라이언트가 인터페이스와 DI를 통해 접근하도록 설계하고, 데코레이터 패턴을 적용해서, 비즈니스 로직을 담은 클래스의 코드에는 전형 영향을 주지
않으면서 트랜잭션이라는 부가기능을 자유롭게 부여할 수 있는 구조 만듦

트랜잰셕은 처리하는 코드의 일종의 데코레이터에 담겨서, 클라이너트와 비즈니스 로직을 담은 타깃 클래스 사이에 존재하도록 만들엇다.
그래서 클라이언트가 일종의 대리자인 프록시 역할을 하는 트랜잭션 데코레이터를 거쳐서 타깃에 접근 할 수 있게 됐다

비즈니스 로직 코드에 트랜잭션의 코드의 (부가기능의 흔적이 남아있다)
==> 프록시를 이용한 데코레이터 패턴 이용 BUT 비즈니스 로직 인터페이싀 모든 메소드마다 트랜잭션 기능을 부여하는 코드를 넣어 프록시 클래스 만드는 작업 큰집 AND 트랜잭션 기능을 부여하지 않아도 되는 메소드조차 프록시로서 위임 기능이 필요하기 때문에 일일이 구현
==> JDK의 다이내믹 프록시 사용 (Proxy.newProxy())) 덕분에 프록시 클래스 코드 작성의 부담도 덜고, 부가기능 부여 코드 여기저기저 중복되는 문제 해결 전에는 프록시를 우리가 일일히 우리가 만들워줫어했는데 다이낵믹 프록시가 만들어줌 BUT 동일한 기능의 프록시를 여러 오브젝트에 적용할 경우 오브젝트 단위로 종복이 일어나는 문제는 해결하지 못했다. (그리고 target정보를 알고 있어야 한다)
==> JDK의 다이내믹 프록시와 같은 프록시 기술을 추상화한 스프링의 프록시 팩토리 빈을 이용해서 다이내믹 프록시 생성 방법에 DI를 도입. 내부적으로 템플릿/콜백(Invocation interface에 필요한 정보를 담고있다 쓰기만 하면 된다)) 패턴을 활용하는 스프링의 프록시 팩토리 빈 덕분에 부가기능을 담은
어드바이스와 부가기능 선정 알고리즘을 담은 포인트컷에 프록시에 분리될 수 있었고 여러 프록시에서 공유해서 사용할 수 있게 했다 BUT 트랜잭션 적용 대상이 되는 빈마다 일일이 프록시 팩토리 빈을 설정해줘야 한다는 부담이 있다
==> 자동 프록시 생성법과 포인컷: 프록시를 적용할 대상을 일일이 지정하지 않고 패턴을 이용해 자동으로 선정 할 수 있도록, 클래스를 선정하는 기능을 담은
확장된 포인트것을 사용했다. 결국 트랜잭션 부가기능을 어디에 적용하는지에 대한 정보를 포인트컷에 독립적인 정보로 완전히 분리할 수 있었다. 

##AOP란
이렇게 애플리케이션의 핵심적인 기능에서 부가적인 기능을 분리해서 애스페트라는 동극한 모듈로 만들어서 설계하고 개발하는 방법 = AOP
이름만 들으면 마치 OOP가 아닌 다른 프로그래밍 언어 또는 패러다임으로 느껴지지만, AOP는 OOP를 돕는 보조적인 기술이지 OOP를 
완전히 대체하는 새로운 개념은 아니다. 부가기능이 핵심기능 안으로 침투해 들어가 버리면 핵심기능 설계에 객체지향 기술의 가치를 온전히 부여하기 힘들어진다. 부가도니 코드로 인해 객체지향적인 설계가 주는 장점을 잃어버리기 십상이다. 

스프링 AOP의 부가기능을 담은 어드바이스가 적용되는 대상은 오브젝트의 메소드다. 프록시 방식을 사용했기 떄문에 메소드 호출 과정에 참여해서
부가기능을 제공해주게 되어 있다. 어드바이스가 구현하는 MethodInterceptor 인터페이스는 다이내믹 프록시의 InvocationHandler와 마찬가지로
프록시로부터 메소드 요청정보를 전달받아서 타깃 오브젝트의 메소드를 호출한다. 타깃의 메소드를 호출하는 전후에 다양한 부가기능을 제공 할 수 있다.

##AspectJ
프록시처럼 간접적인 방법이 아니라, 타깃 오브젝트를 뜯어고쳐서 부가기능을 직접 넣어주는 직접적인 방법을 사용한다. 부가기능을 넣는다고 타깃 오브젝트의 소스코드를 수정할 수 는 없으니, 컴파인될 타깃의 클래스 파일 자체를 수정하거나 클래스가 JVM에 로딩되는 시점을 가로채서 바이트코드를 조작하는 조잡한
방법을 사용한다. 트랜잭션 코드가 UserService클래스에 비즈니스 로직과 함께 있었을 때처럼 만들어버리는 것이다. 

쓰는 이유:
1. 스프링과 같은 DI컨테이너의 도움을 받아서 자동 프록시 생성 방식을 사용하지 않아도 AOP적용 가능하기 때문
2. 프록시보다 유연하고 강력함. 프록시를 AOP의 핵심 매커니즘으로 사용하면 부가기능을 부여할 대상은 클라이너트가 호출할때 사용하는 메소드로 제한된다. 하지만 바이트코드를 직접 조작해서 AOP를 적용하면 오브젝트의 생성, 필드 값의 조회와 조작, 스태틱 초기화 등의 다양한 작업에 부가기능을 부여해줄 수 있다.

## 용어정리 하고 넘어가자
타깃: 부가기능을 부가할 대상. 핵심기능을 다믕ㄴ 클래스일 수도 있지만 경우에 따라서는 다른 부가기능을 제공하는 프록시 오브젝트
어드바이스: 어드바이스는 타깃에게 제공할 부가기능을 담은 모듈. 어드바이스는 여러 종류가 있다. MethodInterceptor처럼 메소드 호출 과정에
전반적으로 참여하는 것도 있지만, 예외가 발생했을 때만 동작하는 어드바이스처럼 메소드 호출 과정의 일부에서만 동작하는 어드바이스도 있다. 

## 조인포인트
조인포인트란 어드바이스 적용될수 있는 위치를 뜻한다. 스프링의 AOP에서 조인 포인트는 메소드의 실행 단계뿐이다 타깃 오브젝트가 구현한 인터페이스의 모든 메소드는 조인포인트가 된다.

## 포인트컷
포인트 컥시안 어드바이스를 적용할 조인 포인트를 선별하는 작업 또는 그 기능을 정의하는 모듈. 스프링의 AOP의 조인 포인트는 메소드의 실행이므로 스프링의 포인트컷은 메소드를 선정하는 기을 갖고 있다. 메소드는 클래스 안에 존재하는 것이기 때문에 메소드 선정이란 결국 클래스를 선정하고 그 안의 메소들르 선정하는 과정을 거치게 된다.

## 프록시
프록시는 클라이언트와 타깃 사이에 투명하게 존재하면서 부가기능을 제공하는 오브젝트다. DI를 통해 타깃 대신 클라이언트에게 주입되며, 클라이넡의 메소드 호출을 대신 받아서 타깃에 위임해주면서, 그 과정에서 부가기능을 부여한다

## 어드바이저
어드바이저는 포인트컷과 어드바이스를 하나씩 갖고 있는 오브젝트다. 어드바이저는 어떤 부가기능(어드바이스)를 어디에(포인트컷)에 전달할 것인가를 알고 있는 AOP의 가장 기본이 되는 모듈이다. 스프링은 자동 프록시 생성기가 어드바이저를 AOP 작업의 정보로 활용한다. 어드바이저는 스프링 AOP에서만 사용되는 특별한 용어이고, 일반적인 AOP에서는 사용되지 않는다

## 애스팩트
OOP의 클래스와 마찬가지로 애스팩트는 AOP의 기본 모듈이다. 한 개또는 그 이상의 포인트컷과 어드바이스의 조합으로 만들어지며 보통 싱글톤 형태의 오브젝트로 존재한다. 따라서 클래스와 같은 모듈 정의와 오브젝트와 같은 실체의 구분이 특별히 없다. 두 가지 모두 애스팩트라고 불린다. 스프링의 어드바이저는 아주 단순한 애스펙트라고 볼 수도 있다. 
