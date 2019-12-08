# Chapter 06 - AOP

> 가장 인기 있는 AOP의 적용 대상은 트랜잭션 경계설정 기능이다.

## 6.1 트랜잭션 코드의 분리
> 트랜잭션 경계설정과 비즈니스 로직을 분리해보자.

### DI를 이용한 클래스의 분리

* 보통 이렇게 인터페이슬 리용해 구현 클래스를 클라이언트에 노출하지 않고 런타임 시에 DI를 통해 적용하는 방법을 쓰는 이유는, 일반적으로 구현 클래스를 바꿔가면서 사용하기 위해서이다.
![enter image description here](https://lh3.googleusercontent.com/PN-RByLQcCRNQgaxY85aH6n9FTqP0GXk6wr1YzZsd2FmNApAUyiy1VY1vo6_N-g5c8CXsOXVBnw)

* 그런데, 한번에 두개의 구현클래스를 동시에 이용하면 어떨까?
* 단지, UserServiceImpl는 트랜잭션의 경계설정이라는 책임만 맡고, 실직적인 비즈니스 로직은 UserService의 구현 클래스에 실제적인 로직 처리 작업을 위임한 것이다.
![enter image description here](https://lh3.googleusercontent.com/CrA01sCpRTwDtN-m2MqLtOjMKHckYVGSNfG9g4dpChD4zrfzRmDVB5YlRCdvIFh1GI1CSyNXVig)

```java
public  class  UserServiceTx  implements  UserService{
	private UserService userService;
	// 구체적인 기술은 알지 못하지만 transactionManager라는 이름의 빈으로 등록된 트랜잭션 매니저를 DI로 받아뒀다가 트랜잭션 안에서 동작하도록 만들어줘야 하는 메소드 호출의 전과 후에 필요한 트랜잭션 경계설정 API를 사용해주면 된다.
    private PlatformTransactionManager transactionManager;

    public UserServiceTx(UserService userService, PlatformTransactionManager transactionManager) {
        this.userService = userService;
        this.transactionManager = transactionManager;
    }

	@Override
	public void upgradeLevels() {

	    TransactionStatus status = this.transactionManager.getTransaction(new DefaultTransactionDefinition());

	    try {
		    // 비즈니스 로직을 책임지는 UserServiceImpl
	        userService.upgradeLevels();
	        this.transactionManager.commit(status);
	    } catch (RuntimeException e) {
	        this.transactionManager.rollback(status);
	        throw e;
	    }
	}
}
```

* 그럼 결국 만들어질 빈 오브젝트 의존 관계는
![enter image description here](https://lh3.googleusercontent.com/-RwkWi6X43m5e-VvFNksUws-N-rW2h4OZwTECNx59RhLCWYZXhsezTQHf5A6Flv9avYlA-eDN7E)

![enter image description here](https://lh3.googleusercontent.com/Zd3Jqm5KuSbfDP1tUhv4p9BcZcsTtyJW-RptLSWBDY50JRsrIJTxa38S8Ynyc9WfPIflxItgDls)

* 정리
	* 이제 비즈니스 로직을 담당하고 있는 UserServiceImpl 의 코드를 작성할 때는 트랜잭션과 같은 기술적인 내용에는 전혀 싱경 쓰지 않아도 된다.
	* 비즈니스 로직에 대한 테스를 손쉽게 만들어 낼 수 있다.	

## 6.2 고립된 단위 테스트
* 스텁 vs Mock 오브젝트
	* 스텁: 테스트를 위한 대역
	* 목 오브젝트: 테스트를 위한 대역이자, 테스트 검증에도 참여할 수 있다.

* 단위 테스트
	* 테스트 대상 클래스를 목 오브젝트 등의 테스트 대역을 이용해 의존 오브젝트나 외부의 리소스를 사용하지 않도록 고립시켜서 테스트 하는 것
* 통합 테스트
	* 두 개 이상의, 성격이나 계층이 다른 오브젝트가 연동하도록 만들어 테스트하거나, 또는 외부의 DB나 파일, 서비스 등의 리소스가 참여하는 테스트 하는 것

* 단위 테스트와 통합테스트 중에서 어떤 방법을 쓸지는 어떻게 결정할 것인가?
	* 항상 단위 테스트를 먼저 고려한다.
	* 하나의 클래스나 성격과 목적이 같은 긴밀한 클래스 몇개를 모아서 외부와의 의존관계를 모두 차단하고,
	* 필요에 따라 스텁이나 목 오브젝트 등의 테스트 대역을 이용하도록 테스트를 만든다.
	* 외부 리소스를 사용해야만 가능한 테스트는 통합 테스트로 만든다.
	* 단위 테스트로 만들기 어려운 코드도 있다. 그러면 통합테스트를 사용한다. ex) DAO
	* 여러 개의 단위가 의존관계를 가지고 동작할 때를 위한 통합 테스트는 필요하다.
	* 단위 테스트를 만들기가 너무 복잡하다고 생각 되면 처음부터 통합테스트를 고려해 본다.
	* 스프링 테스트 컨텍스트 프레임워크를 이요하는 테스트는 통합 테스트다.

* 결국 Mockito 사용하자!
* given
* when
* verify

## 6.3 다이내믹 프록시와 팩토리 빈
* 이렇게 마치 자신이 클라이언트가 사용하려고 하는 실제 대상인 것처럼 위장해서 클라이언트의 요청을 받아주는 것을 대리자, 대리인과 같은 역할을 한다고 해서 프록시 라고 부른다.
* 프록시를 통해 최종적으로 요청을 위임받아 처리하는 실제 오브젝트를 타깃 또는 실체라고 부른다.
![enter image description here](https://lh3.googleusercontent.com/FHIdS8pHab4P0TJ6zzMD8ikAgw_6qIGyZgCCVtfr7MJDaDNnp80N1AUsp_xFu61sSSPAuk704CI)

![enter image description here](https://lh3.googleusercontent.com/oFAvN4cY6WpIxcq1LuCULCnfC1W5nBUYtCSZ5J59_SWx979t9wYtLNrMKyQX-2W5ARO5zi--evQ)

* 데코레이터 패턴
	* 타깃에 부가적인 기능을 런타임 시 다이내믹하게 부여해주기 위해 프록시를 사용하는 패턴을 말한다.
	* 프록시가 꼭 한개로 제한되지 않는다.
	* 프록시가 직접 타깃을 사용하도록 고정시킬 필요도 없다.
	* 데코레이터 패턴에서는 같은 인터페이스를 구현한 타겟과 여러개의 프록시를 사용 할 수 있다.
	* 프록시가 여러개인 만큼 순서를 정해서 단계적으로 위임하는 구조로 만들면 된다.

![enter image description here](https://lh3.googleusercontent.com/r2dLmQNB2eDUImNBc3p-vLthbB10X5hE_1KBjsru-ANg1pnO-BI-DTbPwEl3dtEKhM8LNAhFEfs)

![enter image description here](https://lh3.googleusercontent.com/cpF3JxtXetPMxRZ0s734gbMZ0Z1F9AQ3MsZtc7bZhIz60BLwe-4dvSSjOCOxmuwxCtHGdmmqwmM)

* 데코레이터 패턴은 타깃의 코드를 손대지 않고,
* 클라이언트가 호출하는 방법도 변경하지 않은채로 새로운 기능을 추가할 때 유용한 방법이다.

* 프록시 패턴
	* 프록시 패턴은 타깃의 기능 자체에는 관여하지 않으면서 접근하는 방법을 제어해주는 프록시를 이용하는 것이다.
	* 다만 프록시는 코드에서 자신이 만등거나 접근할 타깃 클래스 정보를 알고 있는 경우가 많다.
	* 자바에서는 java.lang.reflect 패키지 안에 프록시를 손쉽게 만들 수 도 있도록 지원해주는 클래스가 있다.
	* 타깃과 같은 메소드를 구현하고 있다가 메소드가 호출되면 타깃 오브젝트로 위임한다.
	* 지정된 요청에 대해서는 부가기능을 수행한다.
![enter image description here](https://lh3.googleusercontent.com/EiUKX_YA1aNdDsBvNsNdsbF25HkxkYe-Lw5VuKwXqLghNntHih5w9oR4GrPPQc2QpUfb3sMPClE)

```java
public class UserServiceTx implements UserService {
	UserService userSErvice;	// 타깃 오브젝트
	
	// 메소드 구현과 위임
	public void add(User user) {
		this.userService.add(user);
	}

	public void upgradeLevels() {
		// 부가 기능 수행
		TransactionStatus status = this.transactionManager.getTransaction(new DefaultTransactionDefinition());
		try {
			userService.upgradeLevels(); // 위임
		}
		...
	}
}
```
* 타깃의 인터페이스를 구현하고 위임하는 코드를 작성하기 번거롭다.
* 부가기능이 필요 없는 메소드도 구현해서 타깃으로 위임하는 코드를 일일이 만들어 줘야 한다.
* 타깃 인터페이스의 메소드가 추가되거나 변경될 때마다 함께 수정해줘야 한다.
* 부가기능 코드가 중복될 가능성이 많다.

> 리플렉션을 사용하면 가능하다. 그 중에서도 다이내믹 프록시를 사용해보자.
![enter image description here](https://lh3.googleusercontent.com/wSQplJuJH_ZZI3GAG_qtRmwwVWPs7iQvOJdykeeeSGn40KgIpbGAnjA5X5_8zORt3Yx6hW_84r8)
* 다이내믹 프록시는 프록시 팩토리에 의해 런타임 시 다이내믹하게 만들어지는 오브젝트다.
* 타깃의 인터페이스와 같은 타입으로 만들어진다.
* 프록시 팩토리에게 인터페이스 정보만 제공해주면 해당 인터페이스를 구현한 클래스의 오브젝트를 자동으로 만들어주기 때문이다.![enter image description here](https://lh3.googleusercontent.com/1NevsROkHv9E4LIRchdyGY_s24XY2FBGpfV3p4v7m_Kb2UW7iBIq31gCNVW4Smesoej7Jkf_nN4)

```java
public class UppercaseHandler implements InvocationHandler {
    private Object target;

    public UppercaseHandler(Object target) {
        this.target = target;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        Object returnValue = method.invoke(target, args);

		// return 값이 String이 아닐수도 있고
		// method 의 이름이 say로 시작하는 경우에만 지정해야하는 경우도 있고
        if (returnValue instanceof String && method.getName().startsWith("say")) {
            return ((String) returnValue).toUpperCase();
        }

        return returnValue;
    }
}
```

* 트랜잭션 InvocationHandler 적용
```java
public class TransactionHandler implements InvocationHandler{
    private Object target;	 // 부가기능을 제공할 타깃 오브젝트
    private PlatformTransactionManager transactionManager;	// 트랜잭션 기능을 제공하는데 필요한 트랜잭션 매니저
    private String pattern;	// 트랜잭션에 적용할 메소드 이름 패턴

    public void setTarget(Object target) {
        this.target = target;
    }

    public void setTransactionManager(PlatformTransactionManager transactionManager) {
        this.transactionManager = transactionManager;
    }

    public void setPattern(String pattern) {
        this.pattern = pattern;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
	    // 트랜잭션 적용 대상 메소드를 선별해서 트랜잭션 경계설정 기능을 부여해준다.
        if (method.getName().startsWith(pattern)) {
            return invokeInTransaction(method, args);
        }
        return method.invoke(target, args);
    }

    private Object invokeInTransaction(Method method, Object[] args) throws Throwable {
        TransactionStatus status = this.transactionManager.getTransaction(new DefaultTransactionDefinition());

        try {
	        // 트랜잭션을 시작하고 타깃 오브젝트의 메소드를 호출한다. 예외가 발생하지 않았다면 커밋!
            Object returnValue = method.invoke(target, args);
            this.transactionManager.commit(status);
            return returnValue;
        } catch (InvocationTargetException e) {
	        // 예외가 발생하면 롤백!
            this.transactionManager.rollback(status);
            throw e.getTargetException();
        }
    }
}
```

* 다이내믹 프록시를 DI를 통해 사용할 수 있도록 하자!  
* 팩토리 빈 사용!
```java
// 생성할 오브젝트 타임을 지정할 수도 있지만,
// 범용적으로 사용하기 위해 Object로 했다.+
public class TxProxyFactoryBean implements FactoryBean<Object> {
    private Object target;
    private PlatformTransactionManager platformTransactionManager;
    private String pattern;
    private Class<?> serviceInterface;	// 다이내믹 프록시를 생성할때 필요하다. UserService 외의 인터페이스를 가진 타깃에도 적용 가능하다.

    public TxProxyFactoryBean(Object target, PlatformTransactionManager platformTransactionManager, String pattern, Class<?> serviceInterface) {
        this.target = target;
        this.platformTransactionManager = platformTransactionManager;
        this.pattern = pattern;
        this.serviceInterface = serviceInterface;
    }

    public void setTarget(Object target) {
        this.target = target;
    }

    @Override
    // 빈 오브젝트를 생성해서 돌려준다.
    public Object getObject() throws Exception {
	    // DI 받은 정보를 이용해서 TransactionHandler를 사용하는 다이내믹 프록시를 생성한다.
        TransactionHandler txHandler = new TransactionHandler();
        txHandler.setTarget(target);
        txHandler.setTransactionManager(platformTransactionManager);
        txHandler.setPattern(pattern);

        return Proxy.newProxyInstance(
                getClass().getClassLoader()
                ,new Class[] { serviceInterface }
                ,txHandler);
    }

    @Override
    // 생성되는 오브젝트 타입을 알려준다.
    public Class<?> getObjectType() {
	    // 팩토리 빈이 생성하는 오브젝트 타임은 DI 받은 인터페이스 타입에 따라 달라진다.
	    // 따라서 다양한 타입의 프록시 오브젝트 생성에 재사용 할 수 있다.
        return serviceInterface;
    }

    @Override
    // getObject() 가 돌려주는 오브젝트가 항상 같은 싱글톤 오브젝트인지 알려준다.
    public boolean isSingleton() {
	    // 매번 같은 Object를 리턴하지 않는다.
        return false;
    }

}
```
![enter image description here](https://lh3.googleusercontent.com/H7mFLab3yI0NFPdJtFZ1auI2eWDfZms1FRrjtSoQz13_iVG5CmC48cnha0Aryl4dfeTpgVJtDo8)

![enter image description here](https://lh3.googleusercontent.com/b7AzDOt1JICcPxBVVfXHdgchdggGF21r-senON_E6XPHw6O53EF6nufJ_NXLC49h_oqbQiSHl1jI)

* 인터페이스를 구현하는 프록시 클래스를 일일이 만들 필요 없다!
* 부가적인 기능이 여러 메소드에 반복적으로 나타나지 않는다!



# Chapter 06-2 - AOP

## 6.5 스프링 AOP
* 아직 해결해야할 문제
	* 프록시 팩토리 빈 방식의 접근 방법의 한계
		* 부가기능의 적용이 필요한 타깃 오브젝트마다 거의 비슷한 내용의 ProxyFactoryBean 빈 설정정보를 추가해주는 부분이다.
		* 설정을 매번 복사해서 붙이고 target 프로퍼티의 내용을 수정해줘야 한다.
		* 단순하고 쉽지만, 실수하기 쉽다.
		* 결국, 중복문제다.
* 반복적인 프록시의 메소드 구현을 코드 자동생성 기법을 이용해 해결했다. 그러면, ProxyFactoryBean 설정문제는 자동등록 기법으로 해결할수 없을까?

### 빈 후처리기를 이용한 자동 프록시 생성기
* `DefaultAdvisorAutoProxyCreator`
	* 어드바이저를 이용한 자동 프록시 생성기다.
	* 적용하는 방법은 빈 후처리기 자체를 빈으로 등록하는 것이다.
		* 빈 오브젝트가 생성될 때마다 빈 후처리기에 보내서 후처리 작업을 요청한다.
		* 빈 오브젝트의 프로퍼티를 강제로 수정할 수도 있고,
		* 별도의 초기화 작업을 수행할 수도 있다.
		* 따라서, 스프링이 설정을 참고해서 만든 오브젝트가 아닌 다른 오브젝트를 빈으로 등록시키는 것이 가능하다.
		* 이를 잘 이용하면 스프링이 생성하는 빈 오브젝트의 일부를 프록시로 포장하고,
		* 프록시를 빈으로 대신 등록할 수도 있다.

![빈 후처리기를 이용한 프록시 자동생성](https://lh3.googleusercontent.com/AnwfXw0MxWtvVJ3IZHI25ZdgGR_79_GA4GUhHnrezkMZNXk6FJuQixoDbf63eQ6OTTvK-W71KX6i)


### AOP란 무엇인가?
#### 트랜잭션 서비스 추상화
* 인터페이스와 DI를 통해 무엇을 하는지는 남기고, 그것을 어떻게 하는지는 분리한 것이다.
* 어떻게 할지는 더 이상 비즈니스 로직 코드에는 영향을 주지 않고 독립적으로 변경할 수 있게 됐다.

#### 프록시와 데코레이터 패턴
* 여전히 비즈니스 로직 코드에는 트랜잭션을 적용하고 있다는 사실이 드러나 있다.
	* 클라이언트가 인터페이스와 DI를 통해 접근하도록 설계하고,
	* 데코레이터 패턴을 적용해서,
	* 비즈니스 로직을 담은 클래스의 코드에는 전혀 영향을 주지 않으면서 트랜잭션이라는 부가기능을 자유롭게 부여했다.
	* 트랜잭션을 처리하는 코드는 일종의 데코레이터에 담겨서,
	* 클라이언트와 비즈니스로직을 담은 타깃 클래스 사이에 존재하도록 만들었다.
	* 그래서, 클라이언트가 일종의 대리자인 프록시 역할을 하는 트랜잭션 데코레이터를 거쳐서 타깃에 접근할 수 있게 됐다.

#### 다이내믹 프록시와 프록시 팩토리 빈
* 비즈니스 로직 인터페이스의 **모든 메소드마다 트랜잭션 기능을 부여하는 코드를 넣어 프록시 클래스를 만드는 작업이 짐**이 됐다.
	* 트랜잭션 기능을 부여하지 않아도 되는 메소드조차 프록시로서 위임 기능이 필요하기 때문에 일일이 다 구현해저야 했다.
	* 프록시 클래스 없이도 프록시 오브젝트를 런타임 시에 만들어주는 JDK 다이내믹 프록시 기술을 적용했다.
		* 덕분에 프록시 클래스 코드 작성의 부담도 덜고, 부가기능 부여 코드가 여기저기 중복돼서 나타나는 문제도 일부 해결할 수 있었다.
		* 일부 메소드에만 트랜잭션을 적용해야하는 경우에는, 메소드를 선정하는 패턴 등을 이용했다.
	* 하지만, 동일한 기능의 프록시를 여러 오브젝트에 적용할 경우 오브젝트 단위로 중복이 일어나는 문제는 해결하지 못했다.
	* JDK 다이내믹 프록시와 같은 프록시 기술을 추상화한 **스프링의 프록시 팩토리 빈**을 이용했다.
		* 내부적으로 템플릿/콜백 패턴을 활용하는 스프링의 프록시 팩토리 빈 덕분에 
		* 부가기능을 담은 어드바이스와 부가기능 선정 알고리즘을 담은 포인트컷은 프록시에서 분리될 수 있었고,
		* 여러 프록시에서 공유해서 사용할 수 있게 됐다.

#### 자동 프록시 생성 방법과 포인트컷
* 트랜잭션 적용 대상이 되는 빈마다 일일이 프록시 팩토리 빈을 설정해줘야 한다는 부담이 남아있다.
	* 이를 해결하기 위해, 스프링 컨테이너의 빈 생성 후처리 기법을 활용해 컨테이너 초기화 시점에서 자동으로 프록시를 만들어주는 방법을 도입했다.
		* 프록시를 적용할 대상을 일일이 지정하지 않고 패턴을 이용해 자동으로 선정할 수 있도록,
		* 클래스를 선정하는 기능을 담은 확장된 포인트컷을 사용했다.
		* 결국, 트랜잭션 부가기능을 어디에 적용하는지에 대한 정보를 포인트컷이라는 독립적인 정보로 완전히 분리할 수 있었다.
		* 처음에는, 클래스와 메소드 선정 로직을 담은 코드를 직접 만들어서 포인트컷으로 사용했지만,
		* 최종적으로는 포인트컷 표현식이라는 좀 더 편리하고 깔끔한 방법을 활용해서 간단한 설정만으로 적용 대상을 손쉽게 선택할 수 있게 됐다.

#### 부가기능 모듈화
* 관심사가 같은 코드를 분리해 한데 모으는 것은 소프트웨어 개발의 가장 기본이 되는 원칙이다.
* 하지만, 트랜잭션 적용 코든는 기존에 써왔던 방법으로는 간단하게 분리해서 독립된 모듈로 만들 수가 없었다.
* 트랜잭션 경계설정 기능은 다른 모듈의 코드에 부가적으로 부여되는 기능이라는 특징이 있기 때문이다.
* 그래서 앞에 기술들을 사용했다.(다이내믹 프록시, IoC/DI 컨테이너 빈 생성, 빈 후처기 기술 ...)
* 트랜잭션과 같은 부가기능은 핵심기능과 같은 방식으로는 모듈화하기 매우 힘들다.
* 이름 그대로 부가기능이기 때문에 스스로는 독립적인 방식으로 존재해서는 적용되기 어렵기 때문이다.
	* 사용자 관리와 같은 핵심기능을 가진 모듈은 그 자체로 독립적으로 존재할 수 있으며,
	* 독립적으로 테스트도 가능하고,
	* 최소한의 인터페이스를 통해 다른 모듈과 결합해 사용되면 된다.
	* 반면에, **부가기능은 여타 핵심기능과 같은 레벨에서는 독립적으로 존재하는 것 자체가 불가능하다.**
		* 테스트하기 쉬운 코드로 개발하기
			* [발표 문서](https://www.slideshare.net/OKJSP/okkycon-120497749)
			* [발표 동영상](https://youtu.be/Cz_a2gQp63c)

#### AOP: 애스펙트 지향 프로그래밍
* 전톡적인 객체지향 기술의 설계 방법으로는 독립적인 모듈화가 불가능한 트랜잭션 경계설정과 같은 부가기능을 어떻게 모듈화할 것인가를 연구해온 사람들은
* 이 부가기능 모듈화 작업은 기존의 객체지향 설계 패러다임과는 구분되는 새로운 특성이 있다고 생각했다.
* 그래서 이런 부가기능 모듈을 객체지향 기술에서 주로 사용하는 오브젝트와는 다르게 특별한 이름으로 불렀다.
	* 그것이 바로 **애스팩트(aspect)**다.
	* 애스팩트는 부가될 기능을 정의한 코드인 어드바이스와, 어드바이스를 어디에 적용할지를 결정하는 포인트컷을 함께 갖고 있다.
	* 어드바이저는 아주 단순한 형태의 애스팩트라고 볼 수 있다.
	* 애스팩트는 그 단어의 의미대로, **애플리케이션을 구성하는 한 가지 측면**이라고 생각 할 수 있다.
	* 기존의 객체지향 설계 기법으로 해결할 방법은 없었다.
![enter image description here](https://lh3.googleusercontent.com/EXq5YwTbMu4Cl6sAuqBa13mYPv95ruWnaV0hcbLwEXIvYBEbbBDBnJ5m7sO6YS5zVPnYxMtbOkKT)
* 이렇게 애플리케이션의 핵심적인 기능에서 부가적인 기능을 분리해서 애스펙트라는 독특한 모듈로 만들어서 설계하고 개발하는 방법을 **애스펙트 지향 프로그래밍(Aspect Oriented Programming: AOP)**라고 부른다.
* AOP는 OOP를 돕는 보조적인 기술이지 OOP를 완저히 대체하는 새로운 개념은 아니다.
* AOP는 애스펙트를 분리함으로써 핵심기능을 설계하고 구현할 때 객체지향적인 가치를 지킬 수 잇도록 도와주는 것이라고 보면 된다.
	* AOP는 결국 애플리케이션을 **다양한 측면에서 독립적으로 모델링하고, 설계하고, 개발할 수 있도록 만들어주는 것**이다.
	* 그래서 **애플리케이션을 다양한 관점에서 바라보며 개발할 수 있게 도와준다.**
	* 애플리케이션을 **사용자 관리라는 핵심 로직 대신 트랜잭션 경계설정이라는 관점에서 바라보고 그 부분에 집중해서 설계하고 개발할 수 있게 된다는 뜻** 이다.
* 이렇게 애플리케이션을 특정한 관점을 기준으로 바라볼 수 있게 해준다는 의미에서 AOP를 관점지향 프로그래밍이라고 한다.

