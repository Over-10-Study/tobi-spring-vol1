# Chapter 03 - 템플릿
> 예외처리와 안전한 리소스 반환을 보장해주는 DAO 코드를 만들고 이를 객체지향 설계 원리와 디자인 패턴, DI 등을 적용해서 깔끔하고 유연하며 단순한 코드로 만드는 방법을 살펴본다.

## 3.1 다시보는 초난감 DAO
> 예외사항에 대한 처리를 알아보자.

* 정상적인 JDBC코드의 흐름을 따르지 않고 중간에 어떤 이유로든 예외가 발생했을 경우에도 사용한 리소스를 반드시 반환하도록 만들어야 하기 때문이다. 그렇지 않으면 시스템에 심각한 문제를 일으킬 수 있다.
	* DB 풀은 매번 getConnection()으로 가져간 커넥션을 명시적으로 close()해서 돌려줘야지만 다시 풀에 넣었다가 다음 커넥션 요청이 있을 때 재사용할 수 있다.
	* 그런데... 오류가 날 때마다 미처 반환되지 못한 Connection이 계속 쌓이면 어느 순간에 커넥션 풀에 여유가 없어지고 리소스가 모자란다는 심각한 오류를 내며 서버가 중단될 수 있다.
* JDBC 코드에서 어떤 상황에서도 가져온 리소스를 반환하도록 try/catch/finally 구문 사용을 권장한다. ([AutoCloseable](https://docs.oracle.com/javase/7/docs/api/java/lang/AutoCloseable.html) 을 사용해도 될듯...)

## 3.2 변하는 것과 변하지 않는 것
* 복잡한 try/catch/finally 블록이 2중으로 중첩까지 되어 나오는데다, 모든 메소드 마다 반복된다.
* 예외상황 처리 코드는 테스트하기가 매우 어렵고 모든 DAO 메소드에 대해 이런 테스트를 일일이 한다는 건 매우 번거롭다.
```java
Connection c = null;
PreparedStatement ps = null;

try { 
	c = dataSource .getConnection();
	ps = c.prepareStatement("delete from users"); // 변하는 부분 나머지는 변하지 않는다.
	
	ps.executeUpdate();
} catch(SQLException e) {
	throw e
} finally {
	if (ps != null) { try { ps.close(); } catch (SQLException e) {} }
	if (c != null) { try {c .close(); } catch (SQLException e) {} }
}
```
* 방법1: 메소드를 추출한다.
* 방법2: 템플릿 메소드 패턴의 적용
```java
// UserDao 의 서브 클래스
public class UserDaoDeleteAll extends UserDao {
	protected PreparedStatement makeStatement(Connection c) throws SQLException { 
		PreparedStatement ps = c.prepareStatement("delete from users");
		return ps;
	}
}
// DAO 로직마다 상속을 통해 새로운 클래스를 만들어야 한다.
// 확장구조가 이미 를 설계하는 시점에서 고정되어 버린다는 점이다.
```
* 방법3: 전략패턴의 적용
* 방법4: DI 적용을 위한 클라이언트/컨텍스트 분리
```java
// 클라이언트가 컨텍스트를 호출할 때 넘겨줄 전략파라미터 stmt
public void jdbcContextWithStatementStrategy(StatementStrategy stmt) throws SQLException {
	Connection c = null;
	PreparedStatement ps = null;
	try (
		c = dataSource .getConnection();
		ps = stmt.makePreparedStatement(c);
		ps.executeUpdate();
	} catch (SQLException e) {
		throw e; 
	} finally {
		if (ps != null) { try { ps.close(); } catch (SQLException e) {} }
		if (c != null) { try {c .close(); } catch (SQLException e) {} }
	}
}

// 클라이언트 책임을 담당할 deleteAll() 메소드
public void deleteAll() throws SQLException {
	StatementStrategy st = new DeleteAllStatement(); // 선정한 전략 클래스의 오브젝트 생성 
	jdbcContextWithStatementStrategy(st); // 컨텍스트 호출 전략 오브젝트 전달
}
```

## 3.3 전략 패턴 최적화
* 로컬 클래스
```java
public void add(final User user) throws SQLException {
	class AddStatement implements StatementStrategy {
		public PreparedStatement makePreparedStatement(Connection c) throws SQLException {
			PreparedStatement ps = c.prepareStatement(
					"insert into users(id, name , password) values(7,7,7)"
			);
			
			// 로컬 클래스의 코드에서 외부의 로컬 변수에 직접 접근이 가능하다.
			ps.set5tring(1 , user.getld()); 
			ps.set5tring(2, user.getName());
			ps.set5tring(3, user.getPassword());
			
			return ps;
		}
	}
	StatementStrategy st = new StatementStrategy() {

	}; // 생성자 따라미터로 user를 전달하지 않아도된다
	jdbcContextWith5tatement5trategy(st);
}
```

* 익명 클래스
```java
public void add(final User user) throws SQLException {
	// 인터페이스를 생성자처럼 이용하여 오브젝트를 만든다.
	StatementStrategy st = new StatementStrategy() {
		public PreparedStatement makePreparedStatement(Connection c) throws SQLException {
			PreparedStatement ps = c.prepareStatement(
					"insert into users(id, name , password) values(7,7,7)"
			);
			
			// 로컬 클래스의 코드에서 외부의 로컬 변수에 직접 접근이 가능하다.
			ps.set5tring(1 , user.getld()); 
			ps.set5tring(2, user.getName());
			ps.set5tring(3, user.getPassword());
			
			return ps;
		}
	};
	jdbcContextWith5tatement5trategy(st);
}

// jdbcContextWith5tatement5trategy파라미터에 직접 오브젝트 생성한다.
// depth 가 깊어져서 별로인듯...
public void add(final User user) throws SQLException {
	jdbcContextWith5tatement5trategy(
		new StatementStrategy() {
			public PreparedStatement makePreparedStatement(Connection c) throws SQLException {
				PreparedStatement ps = c.prepareStatement(
						"insert into users(id, name , password) values(7,7,7)"
				);
				
				// 로컬 클래스의 코드에서 외부의 로컬 변수에 직접 접근이 가능하다.
				ps.set5tring(1 , user.getld()); 
				ps.set5tring(2, user.getName());
				ps.set5tring(3, user.getPassword());
				
				return ps;
			}
		}
	);
}
```

## 3.4 컨텍스트와 DI

* 스프링의 DI 는 기본적으로 인터페이스를 사이에 두고 의존 클래스를 바꿔서 사용하도록 하는게 목적이다. 꼭 그럴 필요는 없다.
	* 구현방법이 바뀔 가능성이 없다면 인터페이스를 사이에 두지 않고 DI를 적용하는 특별한 구조를 사용해도 된다.
	* 긴밀한 관계를 가지고 강하게 결함되어 있다면
* DI로 만들어야할 이유는
	* 스프링 컨테이너의 싱글톤 레지스트리에서 관리되는 싱글톤 빈이 되야한다면
	* 메소드를 제공해주는 일종의 서비스 오브젝트로서의 의미가 있고
	* 여러 오브젝트에서 공유되어 사용되는것이 이상적이라면
	* DI 를 통해 다른 빈에 의존하고 있다면
	* 다른 빈을 DI 받기 위해서라도 스프링 빈으로 등록되어야 한다.
* 그러나, 이런 방법은 최후에 선택해봐야 한다.

### 코드를 이용하는 수동 DI
![enter image description here](https://lh3.googleusercontent.com/5AdciXK7X9UP6diXETiceBYk7DBMHg9MbucmmZACLKi7twPEKmTpAemAFjuo-Zg030xDd_5OgL8)
* JdbcContext가 외부에서 빈으로 만들어져 주입된 것인지, 내부에서 직접 만들고 초기화한 것인지 구분할 필요도 없고 구분할 수도 없다.
* 한 오브젝트의 수정자 메소드에서 다른 오브젝트를 초기화하고 코드를 이용해 DI 하는 것은 스프링에서도 종종 사용하는 기법이다.

## 3.5 템플릿과 콜백

![enter image description here](https://lh3.googleusercontent.com/_wJPU4cPTnDLxQYubPmXlESUz70DZHNFkCzvdciXySc_IoKUZQLiiTMb6XO2sNy-r8MeDo7_1Ks)
* 템플릿(Template)
	* 전략 패턴의 컨텍스트를 템플릿이라고 부른다.
	* 어떤 목적을 위해 미리 만들어 둔 모양이다.
	* 템플릿 메소드 패턴은
		* 고정된 틀의 로직을 가진 템플릿 메소드를 슈퍼클래스(부모클래스)에 두고,
		* 바뀌는 부분을 서브클래스의 메소드에 두는 구조이다.
* 콜백(Callback)
	* 익명 내부 클래스로 만들어지는 오브젝트를 콜백이라고 부른다.
	* 실행되는 것을 목적으로 다른 오브젝트의 메소드에 전달되는 오브젝트를 말한다.
	* 특정 로직을 담은 메소드를 실행시키기 위해 사용한다.

![enter image description here](https://lh3.googleusercontent.com/w3QKeYTn2Lfo7ycEKZWWPWVQgxwpmks8s6UhuRxOLkjXV1vv1SJGqEmR7qupZdFL4AgPxtKHObg)
* 일반적으로 성격이 다른 코드들은 가능한 한 분리하는 편이 낫다.
* 하지만, 하나의 목적을 위해 서로 긴밀하게 연관되어 동작하는 응집력이 강한 코드들은 한 군데 모여 있는게 유리하다.
* 구체적인 구현과 내부의 전략 패턴, 코드에 의한 DI, 익명 내부 클래스 등의 기술은 최대한 감춰두고,
* 외부에는 꼭 필요한 기능을 제공하는 단순한 메소드만 노출해주는 것이다.



## 3.7 정리
* JDBC와 같은 예외가 발생할 가능성이 있으며 공유 리소스의 반환이 필요한 코드는 반드시 try/catch/finally 블록으로 관리해야 한다.
* 일정한 작업 흐름이 반복되면서 그중 일부 기능만 바뀌는 코드가 존재한다면 전략 패턴을 적용한다.
	* 바뀌지 않는 부분은 컨텍스트로
	* 바뀌는 부분은 전략으로 만들고 인터페이스를 통해 유연하게 전략을 변경할 수 있도록 구성한다.
* 같은 애플리케이션 안에서 여러 가지 종류의 전략을 다이내믹하게 구성하고 사용해야 한다면 컨텍스트를 이용하는 클라이언트 메소드에서 직접 전략을 정의하고 제공하게 만든다.
* 클라이언트 메소드 안에 익명 내부 클래스를 사용해서 전략 오브젝트를 구현하면 코드도 간결해지고 메소드의 정보를 직접 사용할 수 있어서 편리하다.
* 컨텍스트가 하나 이상의 클라이언트 오브젝트에서 사용된다면 클래스를 분리해서 공유하도록 만든다.
* 컨텍스트는 별도의 빈으로 등록해서 DI받거나 클라이언트 클래스에서 직접 생성해서 사용한다. 클래스 내부에서 컨텍스트를 사용할 때 컨텍스트가 의존하는 외부의 오브젝트가 있다면 코드를 이용해서 직접 DI 해줄 수 있다.
* 단일 전략 메소드를 갖는 전략 패턴이면서 익명 내부 클래스를 사용해서 매번 전략을 새로 만들어 사용하고, 컨텍스트 호출과 동시에 전략 DI를 수행하는 방식을 템플릿/콜백 패턴이라고 한다.
* 콜백의 코드에도 일정한 패턴이 반복된다면 콜백을 템플릿에 넣고 재활용하는 것이 편리하다.
* 템플릿과 콜백의 타입이 다양하게 바뀔 수 있다면 제네릭스를 이용한다.
* 스프링은 JDBC 코드 작성을 위해 JdbcTemplate을 기반으로 하는 다양한 템플릿과 콜백을 제공한다.
* 템플릿은 한번 하나 이상의 콜백을 사용할 수도 있고, 하나의 콜백을 여러 번 호출할 수도 있다.
* 템플릿/콜백을 설계할 때는 템플릿과 콜백사이에 주고받는 정보에 관심을 둬야 한다.
