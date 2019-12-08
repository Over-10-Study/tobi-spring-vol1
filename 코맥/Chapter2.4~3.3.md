# 2.4 스프링 테스트 적용

```java
public class UserDaoTest {
	private UserDao dao;

	// ...

	@Before
	public void setUp() {
		ApplicationContext context = new   GenericXmlApplicationContext("applicationContext.xml");
		this.dao = context.getBean("userDao", UserDao.class);
    // ...
	}

	@Test
	public void andAndGet() throws SQLException {		
		// ...
	}

	@Test
	public void getUserFailure() throws SQLException {
		// ...
	}


	@Test
	public void count() throws SQLException {
		// ...
	}
}
```

위 코드는 **애플리케이션 컨텍스트 생성 박식** 에 다음과 같은 문제점이 있다.
- ```@Before```메소드로 애플리케이션 컨텍스트를 생성하므로 테스트 메소드 개수만큼 애플리케이션 컨텍스트가 생성된다.
- 애플리케이션 컨텍스트가 만들어질 때는 모든 싱글톤 빈 오브젝트를 초기화한다.
- 빈 오브젝트를 초가화하는데 독자적으로 많은 리소스를 할당하거나 독립적인 스레드를 띄우는 작업이 있을 수 있다.
- 위 문제를 포함해서 빈 오브젝트를 초기화하는데는 많은 시간이 필요하다.

**테스트는 가능한 한 독립적으로 매번 새로운 오브젝트를 만들어서** 사용하는것이 원칙이다. 하지만 애플리케이션 컨텍스트와 같이 생성에 많은 시간과 자원이 소모되는 경우에는 테스트 전체가 공유하는 오브젝트를 만들 수도 있다. 그리고 다음과 같은 이유도 존재한다.
- 애플리케이션 컨텍스트는 초기화되고 나면 내부의 상태가 거의 바뀌지 않는다.
- 빈은 싱글톤이므로 상태를 갖지 않는다.
- DB 상태는 각 테스트에서 알아서 관리함

테스트 메소드뿐 아니라 테스트 클래스 사이에서도 애플리케이션 컨텍스트 하나를 공유해서 사용할 수 있다.

### 2.4.1 테스트를 위한 애플리케이션 컨텍스트 관리

```java

@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations="/applicationContext.xml")
public class UserDaoTest {
  @Autowired
	ApplicationContext context;

	private UserDao dao;

	// ...

	@Before
	public void setUp() {
		this.dao = context.getBean("userDao", UserDao.class);
    // ...
	}

	@Test
	public void andAndGet() throws SQLException {		
		// ...
	}

	@Test
	public void getUserFailure() throws SQLException {
		// ...
	}


	@Test
	public void count() throws SQLException {
		// ...
	}
}
```

- ```@RunWith```: JUnit 프레임워크의 테스트 실행 방법을 확장할 때 사용하는 애노테이션

SpringJUnit4ClassRunner라는 JUnit용 테스트 컨텍스트 프레임워크 확장 클래스를 지정하여 테스트를 진행하는 동안 애플리케이션 컨텍스트는 만들고 관리한다.

- ```@ContextConfiguration```: 자동으로 만들어줄 애플리케이션 컨텍스트의 설정파일 위치를 지정한다.

스프링은 같은 이 애노테이션을 통해 같은 애플리케이션 컨텍스트의 설정파일을 가진 테스트 클래스에서는 하나의 애플리케이션 컨텍스트를 만들어 이를 **공유** 하도록 한다.

- ```@Autowired```: 스프링의 DI에 사용되는 특별한 애노테이션(Vol.2에서 자세히 설명)

이 애노테이션이 붙은 인스턴스 변수가 있으면, 테스트 컨텍스트 프레임워크는 변수 타입과 일치하는 컨텍스트 내의 빈을 찾는다. 같은 타입의 빈이 두 개 이상이면 변수의 이름이 같은 빈을 우선으로 선택하고 이름으로도 판단할 수 없으면 에러가 발생한다.

위 코드에서 ```ApplicationContext``` 클래스는 설정파일에 없는데도 주입될 수 있는 이유는 스프링 애플리케이션 컨텍스트는 초기화할 때 자기 자신도 빈으로 등록하기 때문이다.

## 2.4.2 DI와 테스트
인터페이스를 두고 DI를 적용해야 하는 이유 세 가지
- 절대 바뀌지 않을 것이라고 확신할 수 있는 코드는 없다.
- 클래스의 구현 방식은 바뀌지 않더라도 DI를 적용하면 다른 차원 서비스 기능을 도입할 수 있다.
- 테스트

### 테스트 코드에 의한 DI
애노테이션으로 자동으로 DI를 해줄 수도 있지만 테스트 클래스마다 같은 DI를 사용할 필요는 없다. 특정한 클래스에서는 다른 DI를 적용하는 방법 역시 스프링에서 제공한다.

```java
// ...
@DirtiesContext
public class UserDaoTest {
  @Autowired
  UserDao dao;

  @Before
  public void setUp() {
    // ...
    DataSource dataSource = new SingleConnectionDataSource(
      "jdbc:mysql://localhost/testdb", "spring", "book", true);
    dao.setDataSource(dataSource);
  }
}
```

- ```@DirtiesContext```: 스프링의 테스트 컨텍스트 프레임워크에게 해당 클래스의 테스트에서 애플리케이션 컨텍스트의 상태를 변경한다는 것을 알린다.

이 애노테이션이 붙은 테스트 클래스에는 애플리케이션 컨텍스트 공유를 허용하지 않는다.

메소드 레벨에서 이 애노테이션을 사용할 수 있다.

```java
@DirtiesContext(classMode = DirtiesContext.ClassMode.BEFORE_EACH_TEST_METHOD)
```

### 테스트를 위한 별도의 DI 설정
프로그램이 동작하는 환경은 여러 가지가 존재한다. 개발 서버, 배포 서버, 테스트 서버 등 이 그 예시이다. 스프링에서는 테스트를 위한 설정파일을 따로 만들 수 있다. 기존 프로덕션에서 사용하는 설정 파일은 ```applicationContext.xml```이다. 이를 테스트에서만 사용하는 설정파일로 사용하고 싶다면 ```test-applicationContext.xml```으로 만들서 사용할 수 있다.

이 파일을 ```@ContextConfiguration```에 설정해주면 해당 설정파일을 사용할 수 있다.

별도로 스프링부트에서는 ```application-test.properties```를 사용할 수 있다.

### 컨테이너 없는 DI 테스트
DAO예제에서 ```UserDaoTest``` 클래스는 ```UserDao``` 코드가 DB에 정보를 잘 등록하고 잘 가져오는지만 확인하면 된다. 스프링 컨테이너에서 ```UserDao```가 동작함을 확인하는 일은 기본적인 관심사가 아니다. 이러한 경우 시간이 오래걸리는 스프링 컨테이너를 사용해서 테스트를 할 필요는 없다.

```java
public class UserDaoTest {
  UserDao dao;

  //...

  @Before
  public void setUp() {
    // ...
    dao = new UserDao();
    DataSource dataSource = new SingleConnectionDataSource(
      "jdbc:mysql://localhost/testdb", "spring", "book", true);
    dao.setDataSource(dataSource);
  }
}
```

위 코드와 같이 직접 DI를 했을 때 장점은 다음과 같다.
- 애플리케이션 컨텍스트를 생성할 필요가 없으므로 테스트 시간이 절약된다.
- 스프링의 API에 의존하지 않고 자신의 관심에만 집중해서 깔끔하게 테스트 코드를 만들 수 있다.

> 침투적 기술 VS 비침투적 기술
> - 침투적 기술: 기술을 적용했을 때 애플리케이션 코드에 기술 관련 API가 등장하거나, 특정 인터페이스나 클래스를
> 사용하도록 강제하는 기술
> - 비침투적 기술: 애플리케이션 로직을 담은 코드에 아무런 영향을 주지 않고 적용이 가능하다.
> 스프링은 비침투적인 기술의 대표적인 예다.

### DI를 이용한 테스트 방법 선택
위에서 총 세 가지 방법을 살펴봤는데, 아래와 같은 순서로 선택하는 것을 추천한다.
- 항상 스프링 컨테이너 없이 테스트할 수 있는 방법을 가장 우선적으로 고려한다.
  - 수행 속도가 빠르고 테스트 자체가 간결하다.
- 테스트 전용 설정파일을 따로 만들어 사용한다.
- ```@DirtiesContext``` 애노테이션을 사용하여 수동 DI 해서 테스트한다.
- 다양한 테스트 지원 기능은 Vol.2에서 다룬다.


## 2.5 학습 테스트로 배우는 스프링
학습 테스트란?
- 자신이 만들지 않은 프레임워크나 다른 개발팀에서 만들어서 제공한 라이브러리 등에 대해서 테스트를 하는 형식
학습 테스트는 사용할 프레임워크나 라이브러리의 동작 방식과 사용 방법을 익히기 매우 좋다.

### 2.5.1 학습 테스트의 장점
- 다양한 조건에 따른 기능을 손쉽게 확인해볼 수 있다.
  - 자동화된 테스트 코드로 다양한 조건에 따른 기능을 빠르게 확인 가능하다.
- 학습 테스트 코드를 개발 중에 참고할 수 있다.
  - 학습 테스트를 위해 사용한 설정 등을 참고할 수 있다.
> 나는 새로운 프레임워크나 기술을 사용할 때면, 먼저 팀원들과 함께 학습 테스트를 만들면서 사용법을 익힌다.
> 그리고 이렇게 만든 학습 테스트를 애플리케이션 테스트의 일부로 추가해둔다.

- 프레임워크나 제품을 업그레이드할 때 호환성 검증을 도와준다.
  - 버전이 업그레이드되면서 생길 수 있는 버그를 빠르게 잡을 수 있다.
- 테스트 작성에 대한 좋은 훈련이 된다.
- 새로운 기술을 공부하는 과정이 즐거워진다.
  - 책이나 레퍼런스 문서 등만을 읽는 것보다 코드를 만들기 때문에 덜 지루하다.

스프링 학습 테스트를 참고할 때 가장 좋은 소스는 바로 **스프링 자신에 대한 테스트 코드다.**

### 2.5.3 버그 테스트
버그 테스트란?
- 실패하는 테스트

버그 테스트의 필요성과 장점은 다음과 같다.
- 테스트의 완성도를 높여준다.
- 버그의 내용을 명확하게 분석하게 해준다.
- 기술적인 문제를 해결하는데 도움이 된다.
  - 버그를 만드는 테스트를 외부의 전문가나 포럼, 메일링 리스트 등 커뮤니티의 도움을 받을 수 있다.

> 동등분할
> 같은 결과를 내는 값의 범위를 구분해서 각 대표 값으로 테스트를 하는 방법을 말한다.
> 한 메소드의 결과값이 true, false, 예외발생 세 가지라면 모든 결과에 대한 테스트 코드를 만든다.
> 경계값 분석
> 한 메서드의 숫자 입력을 테스트 할 때 0이나 그 주변 값 또는 최대값, 최소값 등을 테스트한다.

# 3장 템플릿
템플릿이란?
- 변경이 거의 일어나지 않으며 일정한 패턴으로 유지되는 특성을 가진 부분을 자유롭게 변경되는 성질을 가진 부분으로부터 독립시켜서 효과적으로 활용할 수 있도록 하는 방법

## 3.1 다시 보는 초난감 DAO
이전까지 살펴본 DAO는 여러가지 개선 작업을 거쳤지만, 예외처리 기능을 추가하지 못했다. DB를 사용할 때 커넥션 풀을 사용하는데 ```close()```를 호출하기 전에 예외가 발생한다면 해당 리소스를 다시 커넥션 풀에 반환하지 못하므로 결국 커넥션 풀을 사용하지 못하는 경우까지 발생할 수 있다.

### 예외 처리가 추가된 ```deleteAll()```

```java
public void deleteAll() throws SQLException {
  Connection c = null;
  PreparedStatement ps = null;

  try {
    // 예외가 발생할 가능성이 있는 코드를 모두 try 블록으로 묶어준다.
    c = dataSource.getConnection();
    ps = c.preparedStatement("delete from users");
    ps.executeUpdate();
  } catch (SQLException e) {
    throw e;
  } finally {
    // null 체크를 해주는 이유는 NullPointerException 방지 위함
    if (ps != null) {
      try {
        ps.close();
      } catch (SQLException e) {
      }
    }
    if (c != null) {
      try {
        c.close();
      } catch (SQLException e) {
      }
    }
  }
}
```

### 예외 처리가 추가된 ```getCount()```

```java
public int getCount() throws SQLException {
  Connection c = null;
  PreparedStatement ps = null;
  ResultSet rs = null;

  try {
    c = dataSource.getConnection();
    ps = c.preparedStatement("select count(*) from users");

    rs = ps.executeQeury();
    rs.next();
    return rs.getInt(1);
  } catch (SQLException e) {
    throw e;
  } finally {
    if (rs != null) {
      try {
        rs.close();
      } catch (SQLException e) {
      }
    }
    if (ps != null) {
      try {
        ps.close();
      } catch (SQLException e) {
      }
    }
    if (c != null) {
      try {
        c.close();
      } catch (SQLException e) {
      }
    }
  }
}
```

## 3.2 변하는 것과 변하지 않는 것
위와 같이 복잡한 구성의 코드는 실수하기 매우 좋은 코드이다.
- 어떤 기업에서 인터넷 서비스 시스템은 일주일만에 서버를 재시작해주어야 했다. 이 원인은 제대로 DAO 메소드에서 DB커넥션을 반환하지 않았기 때문이다.

### 3.2.2 분리와 재사용을 위한 디자인 패턴 적용

```java
public void deleteAll() throws SQLException {
  Connection c = null;
  PreparedStatement ps = null;

  try {
    // 예외가 발생할 가능성이 있는 코드를 모두 try 블록으로 묶어준다.
    c = dataSource.getConnection();
    ps = makeStatement(c);
  } catch (SQLException e) {
    throw e;
  } finally {
    // null 체크를 해주는 이유는 NullPointerException 방지 위함
    if (ps != null) {
      try {
        ps.close();
      } catch (SQLException e) {
      }
    }
    if (c != null) {
      try {
        c.close();
      } catch (SQLException e) {
      }
    }
  }
}
```

```java
private PreparedStatement makeStatement(Connection c) throws SQLException {
  PreapredStatement ps;
  ps = c.preparedStatement("delete from users");
  return ps;
}
```

### 템플릿 메소드 패턴의 적용

```java
public class UserDaoDeleteAll extends UserDao {
  private PreparedStatement makeStatement(Connection c) throws SQLException {
    PreapredStatement ps;
    ps = c.preparedStatement("delete from users");
    return ps;
  }
}
```

### 전략 패턴 적용

```java
public interface StatementStrategy {
  PreparedStatement makePreparedStatement(Connection c) throws SQLException;
}
```

```java
public class DeleteAllStatement implements StatementStrategy {
  @Override
  private PreparedStatement makeStatement(Connection c) throws SQLException {
    PreapredStatement ps;
    ps = c.preparedStatement("delete from users");
    return ps;
  }
}
```

```java
public void deleteAll() throws SQLException {
  Connection c = null;
  PreparedStatement ps = null;

  try {
    // 예외가 발생할 가능성이 있는 코드를 모두 try 블록으로 묶어준다.
    c = dataSource.getConnection();

    StatementStrategy strategy = new DeleteAllStatement();
    ps = strategy.makePreparedStatement(c);

    ps.executeUpdate();
  } catch (SQLException e) {
    throw e;
  } finally {
    // null 체크를 해주는 이유는 NullPointerException 방지 위함
    if (ps != null) {
      try {
        ps.close();
      } catch (SQLException e) {
      }
    }
    if (c != null) {
      try {
        c.close();
      } catch (SQLException e) {
      }
    }
  }
}
```

- 더 발전된 전략 패턴

```java
public void jdbcContextWithStatementStrategy(StatementStrategy stmt) throws SQLException {
  Connection c = null;
  PreparedStatement ps = null;

  try {
    c = dateSource.getConnection();

    ps = stmt.makePreparedStatement(c);

    ps.executeUpdate();
  }
  // ...
}
```

클라이언트 책임을 담당할 deleteAll() 메소드

```java
public void deleteAll() throws SQLException {
  StatementStrategy st = new DeleteAllStatement();
  jdbcContextWithStatementStrategy(st);
}
```

> 마이크로 DI
> 클래스 단위가 아닌 메소드 단위로 DI를 적용하는 방법

## 3.3 JDBC 전략 패턴의 최적화

### 로컬 클래스

```java
public void add(User user) throws SQLException {
  // add() 메소드 내부에 선언된 로컬 클래스
  class AddSatement implements StatementStrategy {
    User user;

    public AddStatement(User user) {
      this.user = user;
    }

    public PreparedStatement makePreparedStatement(Connection c) throws SQLException{
      PreparedStatement ps = c.preparedStatement("insert into users(id, name, password)
        values(?, ?, ?)");
        ps.setString(1, user.getId());
        ps.setString(2, user.getName());
        ps.setString(3, user.getPassword());

        return ps;
    }
  }

  StatementStrategy st = new AddStatement(user);
  jdbcContextWithStatementStrategy(st);
}
```

### 익명 클래스

```java
// 익명 내부 캘르스는 구현하는 인터페이스를 생성자처럼 이용해서 오브젝트를 만든다.
StatementStrategy st = new StatementStrategy() {
  public PreparedStatement makePreparedStatement(Connection c) throws SQLException {
    PreparedStatement ps = c.preparedStatement("insert into users(id, name, password)
      values(?, ?, ?)");
      ps.setString(1, user.getId());
      ps.setString(2, user.getName());
      ps.setString(3, user.getPassword());

      return ps;
  }
}
```
