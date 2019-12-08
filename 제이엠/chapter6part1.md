1. p.428 ArgumentCapter 사용법
2. p.428 verify 방법
3. java.lan.reflect 읽어보기
4. what is class loader?
5. 

```java
public void upgradeLevels() {
		TransactionStatus status = 
			this.transactionManager.getTransaction(new DefaultTransactionDefinition());
		try {
			List<User> users = userDao.getAll();
			for (User user : users) {
				if (canUpgradeLevel(user)) {
					upgradeLevel(user);
				}
			}
			this.transactionManager.commit(status);
		} catch (RuntimeException e) {
			this.transactionManager.rollback(status);
			throw e;
		}
	}
```
트랜잭션 로직과 비즈니스 로직이 혼재되어 있다 어떻게든 빼고 싶다!
1. 비즈니스 로직 메소드 분리
2. DI(interface)를 이용한 트랜잭션 분리 ==> <<Interface>> UserService를 UserServiceImpl과 UserServiceTx로 분리한다
```java
public class UserServiceImpl implements UserService {
    UserDao userDao;
    MailSender mailSender;

    public boid upgradeLevels() {
        List<User> users = userDao.getAll();
        for (User user: users) {
            if (canUpgradeLevel(user)) {
                upgradeLevel(user);
            }
        }
    }
}

public class UserServiceTx implements UserService {
    UserService userService;
    PlatformTransactionManager transactionManager;

    pubblic void setTransactionManger(PlatformTransactionManager transactionManager) {
        this.transactionManager = transactionManager;
    }

    public void setUserService(UserService userService) {
        this.userService = userService;
    }

    public void add(User user) {
        this.userService.add(user)
    }

    public void upgradeLevels() {
        TransactionStatus status = this.transactionManager.getTransaction(new DefaultTransactionDefinition());
        try {
            userSerice.upgradeLevel();
            this.transactionManager.commmit(status);
        } catch(RunTimeException e) {
            this.transactionManager.rollback(status);
            throw e;
        }
    }
}
```
이 행동의 장점:
1. 비즈니스 로직을 담당하고 있는 UserServiceImpl의 코드를 작성할 때는 트랜잭션과 같은 기술적인 내용에는 전혀 신경스지 않아도 된다
==> 스프링이아나 트랜잭션 같은 로우레벨의 기술적인 지식은 부족한 개발자라고 할지라도, 비즈니스 로직을 잘 이해하고 자바 언어의 기초에 
충실하면 복잡한 비즈니스로 로직을 담은 UserService 클래스를 개발 할 수 있다. 
2. 두번쨰 장점은 비즈니스 로직에 대한 테스트를 손쉽게 만들어낼수 있다


## 대리인 == proxy
이렇게 마치 자신이 클라이언트가 사용하려고 하는 실제 대상인 것처럼 위장해서 클라이언트의 요청을 받아주는 것을 대리자, 대리인과 같은 역할을
한다고 해서 프록시라고 부른다, 그리고 프록시를 통해 최종적으로 요청을 위임받아 처리하는 실제 오브젝트를 타깃 또는 실체 라고 부른다
 클라이언트 ==> 프록시 ==> 타깃

 프록시의 특징:
 타깃과 같은 인터페이스를 구현했다는 것과 프록시가 타깃을 제어할 수 위치에 있는 것

 프록시의 장점:
 1. 클라이너트가 타깃에 접근하는 방법을 제어함 - 프록시 패턴
 2. 타깃에 부자적인 기능을 부여해주기 위함 - 데코레이터 패턴

 데코레이터 패턴은 타깃에 부가적인 기능을 런타임 시 다이내믹하게 부여해주기 위해 프록시를 사용하는 패턴을 말함.
 다이내믹하게 기능을 부가하는 의미는 컴파일 시점, 즉 코드상에서는 어떤 방법과 순거로 프록시와 타깃인 연결되어 사용되는지 정해져 있지 않다는 뜻

 데코레이터 패턴은 타깃의 코드를 손대지 않고, 클라이언트가 호출하는 방법도 변경 하지 않은 채로 새로운 기능을 추가할때 유용하다

 ## 프록시 vs 프록시 패턴(디자인패턴)
전자는 클라이언트와 사용 대상 사이에 대리 역할을 맡은 오브젝트를 두는 방법을 총칭
후자는 프록시를 사용하는 방법 중에서 타깃에 대해 접근 방법을 제어하려는 목적을 가진 경우
프록시 패턴의 프록시는 타깃의 기능을 확장하거나 추가하지 않는다. 대신 클라이언트가 타깃에 접근 하는 방식을 변경해준다.
==> 타깃 오브젝트를 생성하기가 복잡하거나 당장 필요하지 않은 경우에는 꼭 필요한 시점까지 오브젝트를 생성하지 않는 편이 좋다. 그런데 타깃 오브젝트에 대한 레퍼런스가 미리 필요할 수 있다. (클라이언트에게 타깃에 대한 래퍼런스를 넘겨햐 하는데, 실제 타깃 오브젝트는 만드는 대신 프록시를 넘겨주는 것. 그리고 프로시의 메소드를 통해 타깃을 사용하려고 시도하면, 그때 프록시가 타깃 오브젝트를 생성하지 않고 요청을 위임해주는 식이다)

#프록시를 직접 만들 때 문제점
1. 타깃의 인터페이시를 구현하고 위임하는 코드를 작성하기 번거롭다. 부가기능이 필요없는 메소드도 구현해서 타깃으로 위힘하는 코드를 일일히 만들어ㅜ저야함
2. 부가기능의 중복 메소드마다
==> JDK의 다이내믹 프록시

## 리플렉션
리플렉션은 자바의 코드 자체를 추상화해서 접근하도록 하는 것

## 다이내믹 프록시:
다이내믹 프록시는 프록시 팩토리에 의해 런타임 시에 다이내믹하게 만들어지는 오브젝트다.

## reflection
public Object(1) invoke(Object obj(2), Object... args(3))

(2) --- 메소드를 실행시킬 대상 오브젝트
(3) --- 파라미터 목록
(1) --- 결과를 오브젝트 타입으로 반환!


## 
스프링의 빈은 기본적으로 클래스 이름과 프로퍼티로 정의된다. 스프링은 지정된 클래스 이름을 가지고 리플랙션을 이용해서 해당 클래스의 오브젝트를 만든다
스프링은 내부적으로 리플랙션 API를 이용해서 빈 정의에 나오는 클래스 이름을 가지고 빈 오브젝트를 생성한다

사실 스프링은 private 생성자를 가진 클래스도 빈으로 등록해주면 리플렉션을 이용해 오브젝트를 만들어준다

## 프록시 팩토리 빈의 한계
1. 한 번에 여러개의 클래스에 공통적인 부가기능을 제공하는 것은 쉽지 않은 일이다. 하나의 타깃에 여러 개의 부가기능을 적용하려고 할 때도 문제이다
2. TrasnactionHandler 오브젝트가 프록시 팩토리 빈 개수만큼 만들진다. TransactionHandler는 타깃 오브젝트를 프로퍼티로 갖고 있다. 따라서 트랜잭션 부가기능을 제공하는 동일한 코드임에도 불구하고 타깃 오브젝트가 달라지면 새로운 TrasactionHandler object를 만들어야 한다.

## MethodInterceptor vs InvocationHandler
InvocationHandler의 invoke()메소드는 타깃 오브젝트에 대한 정보를 제공하지 않는다. 따라서 타깃은 InvocationHandler를 구현한 클래스가 직접 알고 있어야 한다. 반면에 MethodInterceptor의 invoke()메소드는 ProxyFactoryBean으로부터 타깃 오브젝트에 대한 정보까지도 함께 제공받는다.

InvocationHandler를 구현했을 때와 달리 MethodInterceptor를 구현한 UppercaseAdvice에는 타깃 오브젝트가 등장하지 않는다. MethodInterceptor로는 메도 정보와 함께 타깃 오브젝트가 담긴 MethodInvocation 오브젝트가 전달된다. MethodInvation은 타깃 오브젝트의 메소드를 실행 할 수 있는 기능이 있기 때문에 MethodInterceptor는 부가기능을 제공하는 데만 집중할 수 있다. 

ProxtFactoryBean에 있는 인터페이스 자동검출 기능을 사용해 타깃 오브젝트가 구현하고 있는 인터페이스 정보를 알아낸다

## 어드바이스는 타깃 오브젝트에 종속되지 않는 순수한 부가기능을 담은 오브젝트이다
프록시는 클라이언트로부터 요청을 받으면 먼저 포인트컷에게 부가기능을 부여할 메소드인지를 확인해달라고 요청한다. 포인트컷은 인터페이스를 구현해서 만들면된다. 프록시는 포잍컷으로부터 부가기능을 적용할 대상 메소드인지 확인받으면, MethoInterceptio타입의 어드바이스를 호출한다. 

## 어드바이저 = 포인트컷 + 어드바이스
포인트컷 없이 어드바이저만 존재할 수 있다.
어드바이저는 템플릿/콜백 패턴을 이용한다.
