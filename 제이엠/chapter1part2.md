## Component란?
책에서 나오는 유저다오와 코넥션메이커는 각각 애플리케이션의 핵심적인 데이터 로직과 기술을 담당하고 있고, 다오팩토리는
는 이런 애플리케이션의 오브젝트들을 구성하고 그 관계를 정의하는 책임을 맡고 있다

전자가 실질적인 로직을 담당하는 컴포넌트라면 후자는 애플리케이션을 구성하는 컴포넌트 구조와 관계를 정의한 설계도 같은 역할이다

== 컴포넌트는 controller, service, repository, domain 객체들, 등 로직을 담당하고 애플리캐이션이 동작하고 하는 애들이고
설계도는 관계설정만

Component vs Bean

## 제어의 역전이란?
일반적으로 프로그램의 흐름은 프로그램이 시작되는 지점에서 당므에 사용할 오브젝틀르 결정하고, 결정한 오브젝트를 생성하고,
만들어진 오브젝트에 있는 메소드를 호출하고, 그 오브젝트 메소드 안에서 다음에 사용할 것을 결정하고 호출하는 식의 
작업이 반복된다.
제어의 역전이란 이런 제어 흐름의 개념을 거꾸로 제어 흐름의 개념을 거꾸로 뒤집는 것이다. 제어의 역전에서는 오브젝트가 
오브젝트가 자신이 사용할 오브젝틀르 스스로 선택하지 않는다. 당연히 생성하지도 않는다. 또 자신이 어떻게 만들어지고 어디서 사용되는지를 알 수 없다.

--> 제어의 흐름이란: 객체가 자신이 사용할 객체의 생성권과 관리권

반면에 프레임워크는 거꾸로 애플리케이션 코드가 프레임워크에 의해 사용된다(예시: cofiguration, application context). 보통 프레임워크위에 개발한 클래스를 등록해두고, 프레임워크가 흐름을 주도하는 중에 개발자가 만든 애플리케이션 코드를 사용하도록 만드는 방식이다.

java colleciont framework는 왜 framework인가? https://www.quora.com/Why-is-Collection-in-Java-called-a-framework-but-not-a-library-It-seems-counter-intuitive-to-the-definition-of-a-framework-which-follows-the-Dont-call-us-well-call-you-principle

## @Confguration
== 애플리케이션 컨텍스트 또는 빈 팩토리가 사용할 설정정보라는 표시
참고로 Ioc Contatiner, BeanFactory, ApplicationContext는 직접 bean를 만들지 않는다
@Configuration안에 있는 @Bean method로 가서 가져온다

Bean instance 생성과정:
getBean() 메소드는 ApplicationContext가 관리하는 오브젝트를 요청하는 메소드다. getBean()의 파라미터인 "userDao"는 ApplicationContext에 등록된 빈의 이름이다. DaoFactory에서 @Bean이라는 애노테이션을 userDao라는 이름의 메소드에 붙였는데 이 메소드 이름이 바로 빈의 이름이 된다

## Application Context의 동작방식
빈목록에 실제로 객체는 없는것 같다 그냥 이름만 등록해둔다:
getBean() 메소드를 호출하면 자신의 빈 목록에서 오청한 이름이 있는지 찾고, 있다면 빈을 생성하는 메소드를 호출해서 오브젝트를 생성시킨 후 클라이언트에 돌려준다

## Application Context의 장점
1. 클라이언트는 구체적인 팩토리 클래스를 알 필요가 없다 (DaoTest가 new DaoFactory()를 알고 있어야 한다)
2. 애플리케이션 컨텍스트는 종합 IoC 서비스를 제공해준다(오브젝트가 만들어지는 방식, 시점, 전략을 다르게 가져갈 수도 있도, 이에 부가적으로 자동생성, 후처리, 정보의 조합, 설정방식등 기능이 다양하다)
3. 애플리케이션 컨텍스트는 빈을 검색하는 다양한 방법을 제공한다 (annotation 기반)

## 동일성(identity) vs 동등성(비교)
동일성: 주소값만 같으면 같다
동등성: hasCode, equals해서 같으면 같다

## 싱글턴의 문제점
1. private 생성자를 갖고 있기 때문에 상속불가능(왜일까?)
2. 싱글턴은 테스트 하기 힘들다(mock object주입이 어렵다...interface등 다형성이 없기 때문에)
3. 서버환경에서는 싱글통ㄴ이 하나만 만들어지는 것을 보장 못함(JVM 분산)
4. 싱글톤의 사용은 전역 상태를 만들수 있기 때문에 바람직하지 못하다(유혹이 많은가 봐요)


## 의존검색 vs 의존 주입
```java
public class UserDaoTest {
    public static void main(String[] args) {
        ApplicationContext context = new AnnotaionConfigApplicationContext(DaoFactory.class);
        UserDao dao = context.getBean("userDat", UserDao.class);
    }
}
```
```java
public UserDao() {
    ApplicationContext context = new AnnotationConfigApplicationContext(DaoFactory.class);
    this.connectionMaker = context.getBean("connectionMaker", ConnectionMaker.class);
}
```
UserDao
하지만 의존검색은 적어도 한번은 필요하다
