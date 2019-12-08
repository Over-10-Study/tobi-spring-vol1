### 스프링 테스트 적용

- 애플리케이션 컨텍스트가 만들어질 때는 모든 싱글톤 빈 오브젝트를 초기화한다.
- 애플리케이션 컨텍스트가 초기화 될 때, 어떤 빈은 독자적으로 많은 리소스를 할당하거나 독립적인 스레드를 띄운다.
- 빈이 할당한 리소스를 정리해주지 않으면 다음 테스트에 영향을 줄 수 있다.
- 애플리케이션 컨텍스트처럼 생성에 많은 시간과 자원이 소모되는 경우에는 테스트 전체가 공유하는 오브젝트를 만든다.
- 애플리케이션 컨텍스트는 초기화되고 나면 내부의 상태가 바뀌는 일은 거의 없기 때문에 테스트의 일관성있는 실행 결과를 보장한다.
- 하지만, 매번 테스트 클래스마다 오브젝트를 새로 만들어야한다.
- **스프링이 직접 제공하는 애플리케이션 컨텍스트 테스트 지원기능을 사용하자.**

**@RunWith : JUnit** 프레임워크의 테스트 실행 방법을 확장할 때 사용한다.

**@ContextConfiguration :** 애플리케이션 컨텍스트의 설정 파일

### 애플리케이션 컨텍스트는 테스트(테스트 클래스) 개수에 상관없이 한 번만 만들어진다.

- 테스트 클래스마다 다른 설정파일 사용 가능

### @Autowired

1. 테스트 컨텍스트는 변수 타입과 일치하는 빈을 찾는다.
2. 생성자나 수정자 메소드가 없어도 주입이 가능하다.
3. 별도의 DI 설정 없이 필드의 타입정보를 이용해 빈을 자동으로 가져온다(자동와이어링)

### DI를 사용해야 하는 이유

1. 소프트웨어 개발에서 절대로 바뀌지 않는 것은 없다.
   - DI를 사용하면, 변경이 필요한 상황이 닥쳤을 때 수정에 들어가는 시간과 비용의 부담을 줄여준다.
2. 인터페이스를 두고 DI를 적용하면 기능 추가의 유용하다.
   - AOP를 배워보자
3. 테스트는 실행가능하고 빠르게 동작하도록 작성해야한다.
   - 작은 단위의 대상에 대해 독립적으로 만들어지고 실행되게 하는데 중요한 역할

**@DirtiesContext** : 테스트 컨텍스트는 애노테이션이 붙은 테스트 클래스에 대해서 애플리케이션 컨텍스트 공유를 허용하지 않는다.

### 테스트 방법 선택

1. 스프링 컨테이너 없이 테스트할 수 있는 방법을 찾는다.
2. 의존관계에 있는 오브젝트 테스트시 DI방식을 이용한다.
3. 예외적인 의존관계를 강제해야하는 경우에는 @DirtiesContext 사용한다.

### 버그 테스트의 장점

1. 미처 검증하지 못했던 부분에 대한 테스트를 보완한다.
2. 버그로 인해 발생할 수 있는 다른 오류를 함께 발견할 수 있다.
3. 기술적인 문제를 해결하는 데 도움이 된다. (?) → 무슨 말인지 모르겠다

### 동등분할

- 같은 결과를 내는 값의 범위를 구분하여 모든 경우에 대한 테스트

### 경계값 분석

- 동등분할 범위의 경계에서 경계의 근처에 있는 값을 이용해 테스트

### 템플릿이란

- 변경이 거의 일어나지 않으며 일정한 패턴으로 유지되는 부분을 독립시켜서 활용하는 방법
- 코드량이 많이지다보면 빼먹거나 실수하는 경우가 생긴다. 테스트로도 벅차다

### 해보자

```java
Connection c = null;
PreparedStatement ps = null;

try {
	c = dataSource.getConnection();
	ps = c.prepareStatement("delete from users");
	ps.excuteUpdate();
} catch (SQLException e) {
	throw e;
} finally {
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
```

### 메소드 추출 방식

```java
public void deleteAll() throws SQLException {
	Connection c = null;
	PreparedStatement ps = null;
	
	try {
		c = dataSource.getConnection();
		ps = makeStatement(c); // 변하는 부분을 추출
		ps.excuteUpdate();
	} catch (SQLException e) {
		...
	}
}

private PreparedStatement makeStatement(Connection c) throws SQLException {
	PreparedStatement ps;
	ps = c.prepareStatement("delete from users");
	return ps;
}
```

- 아무 이득이 없다.

### 템플릿 메소드 패턴 적용

```java
public class UserDaoDeleteAll extends UserDao {

	protected PreparedStatement makeStatement(Connection c) throws SQLException {
		PreparedStatement ps = c.prepareStatement("delete from users");
		return ps;
	}
}
```

- **상속**을 통해 기능을 확장해서 사용
- 상속나왔다 쓰지말자
- 사용하는 jdbc가 많아질수록 서브클래스의 수가 많아진다.
- 확장구조가 이미 클래스를 설계하는 시점되어 버린다.

### 전략패턴의 적용

```java
public void deleteAll() throws SQLException {
	...
	try {
		c = dataSource.getConnection();

		StatementStrategy strategy = new DeleteAllStatement();
		ps = strategy.makePreparedStatement(c);

		ps.executeUpdate();
	} catch (SQLException e) {
	...
}
```

- DeleteAllStatement를 알고있다. OCP위반

# ...
