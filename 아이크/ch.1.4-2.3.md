# 1.4 제어의 역전(IoC)

지난 UserDaoTest (테스트를 위한 main method)

```
public class UserDaoTest {
		public static void main(String[] args) throws ClassNotFoundException, SQLException {
				ConnectionMaker connectionMaker = new DConnectionMaker();

				UserDao userDao = new UserDao(connectionMaker);
								...
		}
}
```

- 두 가지 일을 하고 있다.

1. UserDao와 ConnectionMaker 구현 클래스의 오브젝트를 만드는 것
2. 만들어진 오브젝트를 연결시키는 것

## 팩토리

- 객체의 생성 방법을 결정하고, 오브젝트를 반환하는 오브젝트

  

  ```
  public class DaoFactory {
  		public UserDao userDao() {
  				ConnectionMaker connectionMaker = new DConnectionMaker();
  				UserDao userDao = new UserDao(connectionMaker);
  		return userDao;
  }
  ```

  }

- UserDao 타입의 오브젝트를 어떻게 만들고, 어떻게 준비시킬지 결정한다.

- UserDaoTest는 userDao 생성에 관해 신경쓰지 않는다.

  public class UserDaoTest {
  		public static void main(String[] args) throws ClassNotFoundException, SQLException {
  				UserDao dao = new DaoFactory().userDao();
  								...
  		}
  }

- UserDao와 ConnectionMaker는 애플리케이션의 컴포넌트 역할이다.

- DaoFactory는 애플리케이션의 구조를 결정하는 오브젝트이다.

------

### 다른 DAO의 생성 기능을 추가한다면?

```
public class DaoFactory {
		public UserDao userDao() {
				return new UserDao(new DConnectionMaker());
		}

		public AccountDao accountDao() {
				return new AccountDao(new DConnectionMaker());
		}

		public MessageDao messageDao() {
				return new MessageDao(new DConnectionMaker());
		}
}
```

- ConnectionMaker를 선정하고 생성하는 부분이 중복된다.

### 중복을 제거해보자!

```
public class DaoFactory {
		public UserDao userDao() {
				return new UserDao(connectionMaker());
		}

		public AccountDao accountDao() {
				return new AccountDao(connectionMaker());
		}

		public MessageDao messageDao() {
				return new MessageDao(connectionMaker());
		}

		public ConnectionMaker connectionMaker() {
				return new DConnectionMaker();
		}
}
```

------

### 일반적인 프로그램의 흐름

1. 시작지점에서 다음에 사용할 오브젝트를 결정하고 생성하며 다음을 호출하는 방식이다
2. 모든 오브젝트가 능동적으로 자신이 사용할 클래스를 결정하고, 언제 어떻게 만들지를 결정한다.
3. 사용하는 쪽에서 제어한다.

## 제어의 역전

- 모든 제어 권한을 자신이 아닌 다른 대상에게 넘긴다.

예를 들어,

1. 서블릿의 실행은 개발자가 제어하지 않는다.
2. 서브 클래스의 정의한 메소드가 슈퍼클래스에 의해 호출된다.
3. 애플리케이션 코드는 프레임워크가 짜놓은 틀에서 수동적으로 동작한다.
4. UserDao가 DaoFactory에게 제어권을 넘겼다.

------

# 1.5 스프링의 IoC / 1.6 스프링 IoC의 용어 정리

### 빈(Bean)

- 스프링이 제어권을 가지고 직접 만들고 관계를 부여하는 오브젝트
- 관리되는 오브젝트
- 어플리케이션 컨텍스트의 모든 오브젝트가 다 빈은 아니다

### 빈 팩토리(Bean Factory)

- 빈의 생성과 관계설정 같은 제어를 담당하는 IoC 오브젝트
- 보통은 빈팩토리를 확장해서 어플리케이션 컨텍스트를 이용한다.

### 애플리케이션 컨텍스트(Application Context)

- IoC 방식을 따라 만들어진 빈 팩토리

- 일단 빈 팩토리와 동일하다고 생각하자

- 직접 정보를 가지고 있지 않지만, 별도의 설정정보를 담고있는 무언가를 가져와 활용한다.

  

  ```
  @Configuration
  public class DaoFactory {
  	@Bean
  	public UserDao userDao() {
  			return new UserDao(connectionMaker());
  	}
  	
  	@Bean
  	public ConnectionMaker connectionMaker() {
  			return new DConnectionMaker();
  	}
  }
  ```

  

- DaoFactory는 DAO 오브젝트를 생성하고 DB 생성 오브젝트와 관계를 맺어준다.

- 반면에, 애플리케이션 컨텍스트는 IoC를 적용해 관리할 모든 오브젝트에 대한 생성과 관계설정을 담당한다.

### 애플리케이션 컨텍스트의 동작방식

- DaoFactory 클래스를 설정정보로 등록해두고, @Bean이 붙은 메소드의 이름을 가져와 빈 목록을 만든다.
- 클라이언트가 애플리케이션 컨텍스트의 getBean() 메소드를 호출하면 빈 목록에서 조회 후 클라이언트에게 전달한다.
- 해당하는 빈이 없으면 생성하는 메서드를 호출해서 오브젝트를 생성해서 전달한다.

### 애플리케이션 컨텍스트의 장점

1. 클라이언트는 구체적인 팩토리 클래스를 알 필요가 없다.
2. 종합 IoC 서비스를 제공해준다.
3. 빈을 검색하는 다양한 방법을 제공한다. 

### 설정정보(Configuration)

- 어플리케이션 컨텍스트가 IoC를 적용하기 위해 사용하는 메타 정보
- 컨테이너에 어떤 기능을 세팅하거나 조정할 때 사용
- IoC 컨테이너에 의해 관리되는 오브젝트를 생성하고 구성할 때 사용

### IoC 컨테이너

- IoC 방식으로 빈을 관리하는...?

------

### 어플리케이션 컨텍스트와 오브젝트 팩토리의 차이

- 동등성과 동일성
- 스프링은 여러 번에 걸쳐 빈을 요청하더라도 동일한 오브젝트를 돌려준다.

### 스프링이 빈을 싱글톤으로 만드는 이유

- 스프링을 사용하는 대상이 서버 환경이기 때문에
- 오브젝트의 지속적인 생성을 제한하여 서버의 성능을 높인다.

### 싱글톤 패턴

```
public class UserDao {
		private static UserDao INSTANCE;
		...
		private UserDao(ConnectionMaker connectionmaker) {
				this.connectionMaker = connectionMaker;
		}

		public static synchronized UserDao getInstance() {
				if(INSTANCE == null) INSTANCE = new UserDao(???);
				return INSTANCE;
		}
}
```

### 싱글톤 패턴의 문제점

1. private 생성자를 갖고 있기 때문에 상속할 수 없다.
2. 테스트하기가 어렵다.
3. 서버환경에서는 하나만 만들어지는 것을 보장하지 못한다.
4. 전역 상태이다(아무 객체나 접근, 수정, 공유 가능)

### 싱글톤 레지스트리

- 위와 같은 단점 때문에 스프링이 직접 싱글톤 형태의 오브젝트를 만들고 관리하는 기능을 제공한다.
- 평범한 자바클래스도 싱글톤으로 만들어져 관리되게 한다.

### 싱글톤과 오브젝트의 상태

- 멀티스레드 환경에서 싱글톤은 여러 스레드가 동시에 접근할 수 있다.
- 그래서 상태정보를 갖지 않는 무상태(Stateless)로 만들어져야 한다.

### 스코프(Scope)

- 빈이 생성되고, 존재하고, 적용되는 범위
- 스프링 빈의 기본 스코프는 싱글톤
- 10장에서 알아보자

------

# 1.7 의존관계 주입(DI)

### DI

- 오브젝트 레퍼런스를 외부로부터 제공받고 다른 오브젝트와 의존관계가 생기는 것
- 의존 오브젝트를 연결해주는 작업
- 의존 오브젝트 : 오브젝트가 생성되고 나서 런타임 시에 의존관계가 생기는 오브젝트

### 의존관계 주입의 조건

1. 클래스 모델이나 코드에는 런타임 시점의 의존관계가 드러나지 않는다(인터페이스에 의존)
2. 런타임 시의 의존관계는 컨테이너나 팩토리 같은 제3의 존재가 결정한다.
3. 오브젝트의 레퍼런스를 외부에서 주입한다.

# 한마디로...

제 3의 존재에 의해, 런타임 시에 외부로부터 오브젝트 레퍼런스를 주입받아 오브젝트 사이에 의존 관계가 만들어지는 것!

의존관계 주입 전

```
public UserDao() {
		connectionMaker = new DConnectionMaker();
}
```

의존관계 주입 후

```
private ConnectionMaker connectionMaker;

public UserDao(ConnectionMaker connectionMaker) {
		this.connectionMaker = connectionMaker();
}
```

### 의존관계 검색

- 자신이 필요로 하는 의존 오브젝트를 능동적으로 찾는다.

```
  public UserDao() {
  		AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(DaoFactory.class);
  		this.connectionMaker = context.getBean("connectionMaker", ConnectionMaker.class);
  }
```
  

- 대개는 의존관계 주입 방식을 사용하는 편이 좋다.

- 스태틱 메소드인 main()에서는 DI를 못하기 때문에 의존관계 검색 방식을 사용해야한다.

- 서버의 메인과 비슷한 역할을 하는 서블릿에서도 의존관계 검색을 해야한다.

### 의존관계 검색과 의존관계 주입의 차이

- 의존관계 검색은 검색하는 오브젝트가 스프링 빈일 필요가 없다.
- 의존관계 주입은 반드시 컨테이너가 만드는 빈을 사용해야 한다.

### DI의 장점

- 관심사의 분리(SoC)를 통해 얻어지는 높은 응집도!

# 스프링을 공부하는 것은 DI를 어떻게 활용할 지 공부하는 것!

------

# XML

- 스프링은 다양한 방법을 통해 DI 의존관계 설정정보를 만들 수 있다.
- 다루기 쉽고, 이해하기 쉽다.
- 빌드가 업다.
- 오브젝트 관계에 대한 변경사항에 빠르게 반영할 수 있다.
