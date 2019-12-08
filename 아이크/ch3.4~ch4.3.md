# 컨텍스트와 DI

- 컨텍스트 메소드는 PreaparedStatement를 실행하는 기능을 가진 메소드에서 공유할 수 있다.

- jdbcContextWithStatementStrategy()를 userDao 클래스 밖으로 독립시켜보자

```java
  public class JdbcContext { private DataSource dataSource;

    public void setDataSource(DataSource dataSource) { // DataSource 타입 빈을 DI 받을 수 있게 만든다.
    	this.dataSource = dataSource;
    }
  
    public void workWithStatementStrategy(StatementStrategy stmt) throws SQException {
    	Connection c = null;
    	PreparedStatement ps = null;
  
    	try {
    		c = this.dataSource.getConnection();
    		
    		ps = stmt.makePreparedStatement(c);
  
    		ps.executeUpdate();
    	} catch (SQLException e) {
    		throw e;
    	} finally {
    		if (ps != null) { try { ps.close(); } catch (SQLException e) {} }
    		if (c != null) { try { c.close(); } catch (SQLException e) {} } 
    	}
    }
  
  }
  ```

  ```java
  public class UserDao { 
    ... 
    private JdbcContext jdbcContext;
   
    public void setJdbcContext(JdbcContext jdbcContext) { // jdbcContext를 DI 받도록 만든다.
    	this.jdbcContext = jdbcContext;
    }
  
    public void add(final User user) throws SQLException {
    	this.jdbcContext.workWithStatementStrategy( // DI를 받은 JdbcContext의 컨텍스트 메소드를 사용하도록 변경
    		new StatementStrategy() { ... }
    	);
    }
  
    public void deleteAll() throws SQLException {
    	this.jdbcContext.workWithStatementStrategy(
    		new StatementStrategy() { ... }
    	);
    }
  }
  ```

------

- JdbcContext는 독립적인 JDBC 컨텍스트를 제공해줄 뿐이고 구현 방법이 바뀔 가능성이 없다. 따라서 인터페이스를 사용하지 않았다.

### 인터페이스를 사용하지 않고 DI를 적용하는 것은 문제가 있지 않을까?

- 꼭 그럴 필요는 없다.
- 런타임이 아닌 클래스 레벨에서 의존관계가 존재하기 때문에 DI 개념에 충실하지는 않다.
- 하지만 IoC 관점에서 스프링을 이용해 객체를 주입했다는 것은 DI의 기본을 따르고 있다고 볼 수 있다.

### 그렇다면, JdbcContext를 왜 DI 해야하는가

1. JdbcContext는 싱글톤으로 등록돼서 여러 오브젝트에서 공유해 사용되는 것이 이상적이다.
2. DI는 주입받는 쪽과 주입하는 쪽이 둘 다 스프링 빈이여야 한다. JdbcContext 역시 주입을 받아야 스프링 빈으로 등록된다.

### 인터페이스를 사용하지 않는 DI가 언제 괜찮을까

- JdbcContext처럼 다른 구현체가 필요 없을 때
- 단, 이런 클래스를 바로 DI하는 것은 가장 마지막에 고려하자

### 인터페이스를 사용하지 않아도 되는데 빈으로 분리해야하나

- userDao 내부에서 직접 DI를 해보자(수동 DI)

- DAO마다 하나씩만 만든다면 메모리에 부담이 거의 없다.

- 자주 만들어지고 제거되는게 아니기 때문에 GC에도 부담이 없다.

  ```java
  public class UserDao { ... private JdbcContext jdbcContext;
  
    public void setDataSource(DataSource dataSource) {
    	this.jdbcContext = new JdbcContext(); // jdbcContext를 생성
  
    	this.jdbcContext.setDataSource(dataSource); // 의존 오브젝트를 주입(DI)
    
    	this.dataSource = dataSource; // jdbcContext를 적용하지 않은 메소드를 위해 저장
    }
   }
  ```

### 결론 : 싱글톤 vs 관계 노출 X

------

# 템플릿과 콜백

### 템플릿

- 고정된 작업 흐름을 가진 코드를 재사용한다.
- 전략패턴의 컨텍스트

### 콜백

- 템플릿 안에서 호출되는 것을 목적으로 만들어진 오브젝트
- 익명 내부 클래스로 만들어지는 오브젝트

### 템플릿/콜백의 작업 흐름

1. 콜백 생성
2. 템플릿 호출 시 파라미터로 콜백 전달
3. 템플릿이 내부 작업을 진행
4. 생성된 참조정보를 콜백에게 전달
5. 콜백이 클라이언트가서 일하고 옴
6. 콜백이 가져온 정보로 템플릿이 일을 마무리함

### 템플릿/콜백 방식의 단점

- Dao 매소드에서 매번 익명 내부 클래스를 사용하기 때문에 코드를 작성하고 읽기가 불편하다

### 익명 내부클래스를 분리해보자

```java
public void deleteAll() throws SQLException {
	this.jdbcContext.workWithStatementStrategy(
		new StatementStrategy() { // 변하지 않는 콜백 클래스 정의, 오브젝트 생성
			public PreparedStatement makePreparedStatement(Connection c) throws SQLException {
				return c.preparedStatement("delete from users"); // 변하는 문장
			}
		}
	);
}

public void deleteAll() throws SQLException {
	executeSql("delete from users");
}

private void executeSql(final String query) throws SQLException {
	this.jdbcContext.workWithStatementStrategy(
		new StatementStrategy() { // 변하지 않는 콜백 클래스 정의, 오브젝트 생성
			public PreparedStatement makePreparedStatement(Connection c) throws SQLException {
				return c.preparedStatement(query); // 변하는 문장
			}
		}
	);
}
```

### executeSql을 userDao만 사용하기는 아깝다! 옮기자!

```java
public class JdbcContext {
	...
	public void executeSql(final String query) throws SQLException {
		workWithStatementStrategy(
			new StatementStrategy() {
				public PreparedStatement makePreparedStatement(Connection c) throws SQLException {
					return c.preparedStatement(query);
				}
			}
		}
	}
}
```

### 스프링을 사용하는 개발자라면 스프링이 제공하는 템플릿/콜백 기능을 잘 사용할 수 있어야 한다.

테스트... 주저리주저리

### 제네릭을 이용하면 템플릿/콜백 구조를 범용적으로 구현할 수 있다.

------

# 스프링의 JdbcTemplate

### 지금까지 만든 JdbcContext는 버리고 JdbcTemplate으로 바꿔보자

```java
public class userDao {
	...
	private JdbcTemplate jdbcTemplate;

	public void setDataSource(DataSource dataSource) {
		this.jdbcTemplate = new JdbcTemplate(dataSource);

		this.dataSource = dataSource;
	}
}
```

### update()

JdbcTemplate의 내장 콜백을 사용해보자

```java
public void deleteAll() throws SQLException {
	this.jdbcTemplate.update("delete from users");
}
```

### queryForInt()

Integer 타입의 결과를 받아오는 SQL 문장 전달

```java
public int getCount() {
	return this.jdbcTemplate.queryForInt("select count(*) from users");
}
```

### queryForObject()

- ResultSet? RowMapper?
- ResultSet은 전달받은 정보 전체
- RowMapper는 ResultSet의 정보 하나

```java
public User get(String id) {
    	return this.jdbcTemplate.queryForObject("select * from users where id = ?",
    		new Object[] {id},
    		new RowMapper<User>() {
    			public User mapRow(ResultSet rs, int rowNum) throws SQLException {
    				User user = new User();
    				user.setId(rs.getString("id"));
    				user.setName(rs.getString("name"));
    				user.setPassword(rs.getString("password"));
    				return user;
    			}
    		});
    }
```

queryForObject()를 이용할때 조회 결과가 없다면?

- 기존의 EmptyResultDataAccessException 발생

### query()

```java
public List<User> getAll() {
	return this.jdbcTemplate.query("select * from users order by id",
		new RowMapper<User>() {
			public User mapRow(ResultSet rs, int rowNum) throws SQLException {
				User user = new User();
				user.setId(rs.getString("id"));
				user.setName(rs.getString("name"));
				user.setPassword(rs.getString("password"));
				return user;
			}
		});
}
```

------

### 정리

는 에이든이 해주겠지...

------

### 예외를 처리할 때 반드시 지켜야 할 핵심 원칙

- 모든 예외는 적절하게 복구되든지 작업을 중단시키고 통보돼야 한다.

error / exception과 체크 예외 / runtime과 언체크 예외

### 최근에 새로 등장하는 자바 표준 스펙의 API들을 예상 가능한 예외상황을 다루는 예외를 체크 예외로 만들지 않는 경향이 있기도 하다.

------

# 예외처리 방법

1. 예외 복구
   - 예외를 어떤식으로든 복구할 가능성이 있는 경우 사용
2. 예외처리 회피
   - log를 남기고 다시 throw를 던진다.
   - 다른 레벨에서 처리한다.
3. 예외 전환
   - 적절한 예외로 전환해서 던진다.

```java
public void add(User user) throws DuplicateUserIdException, SQLException {
    	try {
    		// JDBC를 이용해 user 정보를 DB에 추가하는 코드 또는
    		// 그런 기능을가진 다른 SQLException을 던지는 메소드를 호출하는 코드
    	} catch (SQLException e) {
    		if (e.getErrorCode() == MysqlErrorNumbers.ER_DUP_ENTRY)
    			throw DuplicateUserIdException();
    		else
    			throw e;
    	}
    }
```

### 어차피 복구하지 못할 예외라면 런타임 예외로 포장해서 던져버리고, 로그를 남기는 식으로 처리하는게 바람직하다.

### 애플리케이션 예외

- 시스템 또는 외부의 예외상황이 원인이 아닌 로직에 의한 예외

```java
try {
    	BigDecimal balance = account.withdraw(amount);
    	...
    } catch (InsufficientBalanceException e) { // 체크 예외
    	BigDecimal availFunds = e.getAvailFunds();
    	...
    }
```

### SQLException

- 99%의 SQLException은 복구 불가
- 시스템 예외라면 애플리케이션 레벨에서 복구 불가
- 언체크 / 런타임 예외로 전환해줘야 한다.
- JdbcTemplate은 SQLException을 DataAccessException로 포장

### JDBC의 한계

1. 비표준 SQL
2. 호환성없는 SQLException의 DB 에러정보

### ...

```

```
