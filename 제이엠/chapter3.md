## 같이 살펴보기
p.193 리스트 2-21 테스트 db 설정 변경에서 
<property name="url">이 역할을 뭘까?
스프링 배포판 압출 풀어보고 프레임워크 소스코드와 함께 태스트 코드도 발견 할 수 있다
- p. 236: 그러나 스프링의 DI는 넓게 보자면 객체의 생성과
관계설정에 대한 제어권한을 오브젝트에서 제거하고 외부로 위임했다는 IoC라는 개녕
을 포괄한다. 그런 의미에서 JdbcContext를 스프링을 이용해 UserDao 객체에서 시용하
게 주입했다는 건 DI의 기본을 따르고 있다고 볼 수 있다.


## ApplicationContext 테스트에 응용하기
@Before 메소드다 테스트 메소드 개수만큼 반복되기 때문에
자원이 많이 들어가기 때문에 매번 ApplicationContext를 생성하는 것은 낭비다.
As is:
@BeforeEach
ApplicationContext context = new GenericXmlApplicationContext("applicationContext.xml")

To be:
```java
@RunWith(SpringJunit4ClassRunner.class)
@ContextConfiguration(locations="/applicationContext.xml")
public class UserDaoTest {
    @Autowired
    private ApplicationContext context; //테스트 오브젝트가 만들어지고 나면 스프릥 테스트 컨텍스트에 의해 자동으로 값이 주입된다.

    @BeforeEach
    public boid setUp() {
        this.dao = this.context.getBean("userDao", UserDao.class);
    }
}
```
### annoation 설명
@RunWith = 프레임워크의 테스트 실행 방법을 확장할 때 사용하는 애노테이션이다. SpringJUnit4ClassRunner라는 확장 클래스를 지정해주면 JUnit이 테스트를 진행하는 중에 테스트가 사용하 애ㅡㄹ리케이션 컨텍스트를 만들고 관리 작업 ㄱㄱ
@ContextCongifuration = 애플리케이션 컨텍스트이 설정파일 위치 지정

wht this works?
스프링의 JUnit 확장기능은 테스트 시해되기 전에 딱 한번만 application context를 만들어두고, 테스트 오브젝트가 만들어질 때마다 특별한 방법을 이용해 application context 주입

위 방법은 @ContextConfiguration을 사용하고 같은 위치를 지정해두면 다른 테스트 클래스에서도 사용 할 수 있다

## @Autowired
스프링의 DI에 사용되는 특별한 애노테이션
@Autowired가 붙어있다면 변수타입과 일치하는 컨텍스트 내의 빈을 찾는다. 타입이 일치하면 주입 ㄱㄱ. 수정자 메소드나 생성자 필요없다

이상한점이 있다....
우리는 ApplicationContext에 @Configuration과 @Bean를 사용해서 등록을 안해주었는데 어째서 ApplicationContext가 주입이 가능 한 것인가?
답: 스프링 애플리케이션 컨텍스트는 초기화할때 자기 자신도 빈으로 등록한다. 따라서 ApplicationContext에는 ApplicationContext라를 타입이 빈이 존재한다

정리: @Autowire는 ApplicationContext에 있는 빈을 모두 가지고 올 수 있다! 자기 자신도!

## DI에 꼭 interface가 필요할까?
그렇다:
1. 소프트에웨어에서 절대 바뀌지 않는 다는 말은 없다. 덧붙여 interfact를 만들어 연결하는 일은 어려운 일이 아니다
2. 인터페이스를 사이에 두면 다른 차원의 서비스가 가는해진다 
3. 테스트 때문이다.. 테스트를 잘 활용하려면 자동으로 실행 가능하며 빠르게 동작하도록 테스트 코드를 만들어야 한다. 그러기 위해서는 가능한 한 작은 단위의 대상에 국한해서 테스트해야 한다.

@DirtiesContext: 이 애노테이션은 스프링의 테스 컨텍스트 프레임워크에 해당 클래스의 태스트에서 애플리케이션 컨택스트의 상태를 변경한다는 것을 알려준다(근데 예제에서는 이게 안됨)
method레벨에 붙일 수 있다


## OCP의 핵심
문제의 핵심은 변하지 않는, 그러나 많은 곳에서 중복되는 코드와 로직에 따라 자꾸 확장되고 자주 변하는 코드를 잘 분리해내는 작업이다.

## UserDAo try/catch/finally 중보제거 순서
1. method 추출: 하지만 메소드 추출 리팩토링을 적용하는 경우에 분리시킨 메소드를 다르 ㄴ곳에 재사요할 수 있어야 하는데, 이건 반대로 분리시키고 남은 메소드가 재사용이 필요한 부분이고, 분리된 메스도는 DAO로직마다 새롭게 만들어서 확장돼야 하는 부분이기 떄문이다.

2. UserDao를 추상클래스로 전환한뒤 abstract protected PrepateStatement makeStatement(Connection c)를 추상메소드로 만든다 : 하지만 DAO로직마다 상속을 통해 새로운 클래스를 만들어야한다

3. UserDao에서 변하지 않는 부분(Context)와 변하는 부분을 구분한뒤 변하는 부분을 전략패턴으로 나눈다

## 상속의 문제점:
확장구조가 이미 클래스를 설계하는 시점에서 고정되어 버린다. 이미 컴파일 시점에서 클래스 레벨에서 그 관계과 결정되어 있다. 따라서 그 관계에 대한 유연성이 떨어진다.

## context와 contextMethod
이 둘은 일정한 구조를 갖는다. 변하지 않는 구조. try/catch/finally

## Client, Context, Strategy 용어정리
Context == template, 바뀌지 않는 부분과 바뀌는 부분을 담고있는 것
Client interface생성과 호출의 역할을 갖고 있다


```java
public void add(final User user) throws SQLException {
    class AddStatement implements StatementStrategy {
        public PreparedStatement makePreparedStatement(Connection c) throws SQL Exception {
            PreparedStatement ps = c.prepareStatement(inser blah blah)
            ps.setString(1, user.getId());
            ps.setString(2, user.getName());
            ps.setString(3, user.getPassword());

            return ps;
        }
    }

    StatementStrategy st = new AddStatement();
    jdbcContextWithStatementStrategy(st);
}
```
장점: 메소드마다 추가해야 했던 클래스 파일을 하나 줄일 수 있다는 것도 장점이고, 내부클래스의 특징을 이용해 로컬 변수를 바로 가져다 사용할 수 있다는 것

## Client, Strategy, Context
Client == UserDao의 메소드
Startegy == Statement를 구현하는 것들
Context == jdbcContextWithStatementStrategy()

## jdbcContextWithStatementStrategy 분리하기
```java
//as is
public class UserDao {
	private DataSource dataSource;
		
	public void setDataSource(DataSource dataSource) {
		this.dataSource = dataSource;
	}
	
	public void jdbcContextWithStatementStrategy(StatementStrategy stmt) throws SQLException {
		Connection c = null;
		PreparedStatement ps = null;

		try {
			c = dataSource.getConnection();

			ps = stmt.makePreparedStatement(c);
		
			ps.executeUpdate();
		} catch (SQLException e) {
			throw e;
		} finally {
			if (ps != null) { try { ps.close(); } catch (SQLException e) {} }
			if (c != null) { try {c.close(); } catch (SQLException e) {} }
		}
	}


	public void add(final User user) throws SQLException {
		jdbcContextWithStatementStrategy(
				new StatementStrategy() {			
					public PreparedStatement makePreparedStatement(Connection c)
					throws SQLException {
						PreparedStatement ps = 
							c.prepareStatement("insert into users(id, name, password) values(?,?,?)");
						ps.setString(1, user.getId());
						ps.setString(2, user.getName());
						ps.setString(3, user.getPassword());
						
						return ps;
					}
				}
		);
    }
////////////////////////////////////////////////////////
//to be
public class UserDao {
	private DataSource dataSource;
		
	public void setDataSource(DataSource dataSource) {
		this.jdbcContext = new JdbcContext();
		this.jdbcContext.setDataSource(dataSource);

		this.dataSource = dataSource;
	}
	
	private JdbcContext jdbcContext;
	
	public void add(final User user) throws SQLException {
		this.jdbcContext.workWithStatementStrategy(
				new StatementStrategy() {			
					public PreparedStatement makePreparedStatement(Connection c)
					throws SQLException {
						PreparedStatement ps = 
							c.prepareStatement("insert into users(id, name, password) values(?,?,?)");
						ps.setString(1, user.getId());
						ps.setString(2, user.getName());
						ps.setString(3, user.getPassword());
						
						return ps;
					}
				}
		);
    }

public class JdbcContext {
	DataSource dataSource;
	
	public void setDataSource(DataSource dataSource) {
		this.dataSource = dataSource;
	}
	
	public void workWithStatementStrategy(StatementStrategy stmt) throws SQLException {
		Connection c = null;
		PreparedStatement ps = null;

		try {
			c = dataSource.getConnection();

			ps = stmt.makePreparedStatement(c);
		
			ps.executeUpdate();
		} catch (SQLException e) {
			throw e;
		} finally {
			if (ps != null) { try { ps.close(); } catch (SQLException e) {} }
			if (c != null) { try {c.close(); } catch (SQLException e) {} }
		}
	}
}
```
## 왜 jdbcContext를 interface를 빼지 않앗나?
jdbcContext를 그 자체로 독립적인 JDBC 켄텍스트를 제공해주는 서비스 오브젝트 로서 의미 있을 뿐이고 구현 방법이 바뀔 가능 성은 없기 때문이다

## DI가 되기 위한 조건
DI를 위해서는 주입되는 오브젝트와 주입받는 오브젝트 양쪽 모두 스프링 빈으로 등록되야 한다. 스프링이 생성하고 관리하는 IoC대상이어야 DI에 참여할 수 있기 때문이다. 따라서 JdbcContext는 다른 빈을 DI 받기 위해서라도 스피링빈으로 등록되어야 한다(XML)

JDBCContext DI를 통해 다른 빈에 의존하고 있다. JDBCContext는 dataSource 프로퍼티를 통해 DataSource 오브젝트를 주입받도록 되어 있다. 

## 전략패턴관 템플릿과 콜백
템플릿과 콜백은 전랴패턴의 일종이다
전략 패턴의 기본 구조에 익명 내부 클래스를 활용한 방식이다

전략 패턴과의 차이점:
여러 개의 메소드를 가진 일반적인 인터페이스를 사용할 수 있는 전략 패턴의 전략과 달리 템플릿, 콜백 패턴의 콜백은 보통 단일 메소드 인터페이스를 사용한다(funcional interface).

콜백은 일반적으로 하나의 메소드를 가진 인터페이스를 구현한 익명 내부 클래스로 만들어진다고 본다
