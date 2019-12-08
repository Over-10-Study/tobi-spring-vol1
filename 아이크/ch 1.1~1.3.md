### **DAO**

DB를 사용해 데이터를 조회하거나 조작하는 오브젝트

### **JavaBean**

- 데이터를 표현하는 것을 목적으로 하는 자바 클래스
- JavaBean규격서에 따라 작성된 자바 클래스

### JavaBean의 규격

1. 클래스는 패키지화 하여야 한다.
2. 멤버변수는 프로퍼티(Property)라 칭한다.
3. 클래스는 필요에 따라 직렬화가 가능하다.
4. 프로퍼티의 접근자는 private이다.
5. 프로퍼티마다 getter/setter 가 존재해야 하며, 그 이름은 각각 get/set으로 시작해야 한다.
6. 위의 프로퍼티 getter/setter 메서드의 접근자는 public이어야 한다.
7. 외부에서 프로퍼티에 접근은 메서드를 통해서 접근한다.
8. 프로퍼티는 반드시 읽기/쓰기가 가능해야 하지만, 읽기 전용인 경우 getter만 정의할 수 있다.
9. getter의 경우 파라미터가 존재하지 않아야 하고, setter의 경우 한 개 이상의 파라미터가 존재한다.
10. 프로퍼티의 형이 boolean일 경우 get 메서드 대신 is메서드를 사용해도 된다.

    public class Bean_ClassName  {
    	private String name;    // 값을 저장하는 속성 정의(필드)
    	public Bean_ClassName() { }    // 기본 생성자
    
    	public String getName() {    // 필드 값을 읽어오는 메소드 
    		return name; 
    	}
    	public void setName(String name) {    // 필드 값을 저장하는 메소드
    		this.name = name;
    	}
    }

### **JDBC**

1. DB 연결을 하는 Connenction을 가져온다.
2. SQL을 담은 Statement(PreparedStatement)를 만든다.
3. Statement를 실행한다.
4. 조회의 경우 SQL 쿼리의 실행 결과를 ResultSet으로 받는다.
5. Connection, Statement, ResultSet 등의 resource는 작업을 마친 후 닫아준다.
6. JDBC API가 만들어 내는 예외를 처리한다.

개발자가 객체를 설계할 때 가장 염두에 둬야 할 사항은 미래의 변화를 어떻게 대비할 것인가이다.

변경이 일어날 때 필요한 작업을 최소화하고 다른 곳에 문제를 일으키지 않기 위해서 분리와 확장을 고려하여 설계한다.

리펙토링을 하는 이유 : 미래의 변화에 좀 더 손쉽게 대응할 수 있기 때문에

### **템플릿 메소드 패턴**

여러 클래스에서 공통되는 사항은 상위 추상클래스에서 구현하고, 각각의 상세부분은 하위 클래스에서 구현하는 패턴

### **팩토리 메소드 패턴**

객체를 만들어 내는 부분을 서브 클래스에 위임하는 패턴

### **팩토리 메소드**

오브젝트를 생성하는 기능을 가지는 메소드

상속을 통한 확장

    public abstract class UserDao {
    	public void add(User user) throws ClassNotFoundException, SQLException {
    		Connection c = getConnection();
    		...
    	}
    
    	public void get(String id) throws ClassNotFoundException, SQLException {
    		Connection c = getConnection();
    		...
    	}
    
    	**public abstract Connection getConnection() throws ClassNotFoundException, SQLException;**
    }

    public class NUserDao extends UserDao {
    	public Connection getConnection() throws ClassNotFoundException, SQLException {
    		// N 사 DB Connection 생성 코드
    	}
    }

    public class DUserDao extends UserDao {
    	public Connection getConnection() throws ClassNotFoundException, SQLException {
    		// D 사 DB Connection 생성 코드
    	}
    }

상속관계의 단점

슈퍼클래스 내부의 변경이 생기면 모든 서브클래스에 영향이 생길지 모른다.

그래서 슈퍼클래스가 변하지 않도록 제약이 필요하다.

클래스의 분리

독립된 메소드를 만들어 분리하고 메소드 내부에서 분리한 클래스를 사용한다.

특정 클래스와 그 코드에 종속적이면 확장하기 어렵다.

    public class UserDao {
    	private SimpleConnectionMaker simpleConnectionMaker;
    
    	public UserDao() {
    		simpleConnectionMaker = new SimpleConnectionMaker();
    	}
    }

### **인터페이스의 도입**

클래스가 서로 긴밀하게 연결되지 않도록 추상화를 한다.

인터페이스는 어떤 일을 하겠다는 것만 알 수 있고, 어떻게 하는 지는 몰라도 된다.

    public class UserDao {
    	private ConnectionMaker connectionMaker;
    
    	public UserDao() {
    		connectionMaker = new DConnectionMaker();
    	}
    
    	public void add(User user) throws ClassNotFoundException, SQLException {
    		Connection c = connectionMaker.makeConnection();
    		...
    	}
    
    	...
    }

초기에 어떤 생성자를 사용할 것인가에 대해 정할 수 없다. → 독립적으로 확장 가능하지 않다.

해결 방법 : 외부에서 만든 오브젝트를 전달받는다.

    public UserDao(ConnectionMaker connectionMaker) {
    	this.connectionMaker = connectionMaker;
    }

### 객체지향 설계 원칙(SOLID)

1. SRP(The Single Responsibility Principle) : 단일 책임 원칙
    - 모든 클래스는 단 하나의 책임을 가진다. 다시 말하면 클래스를 수정할 이유가 오직 하나여야한다는 뜻이기도 하다.
2. OCP(The Open Closed Principle) : 개방 폐쇄 원칙
    - 확장에 대해서는 개방 되어 있어야 하지만, 수정에 대해서는 폐쇄 되어야 한다.
3. LSP(The Liskov Subsitution Principle) : 리스코프 치환 원칙
    - 자식 클래스는 언제나 자신의 부모클래스를 교체할 수 있다는 원칙이다.
    - 업캐스팅을 해도 아무런 문제가 안되어야 한다.

        Student s = new Student();
        Person p = (Person)s;

4. ISP(The Interface Segregation Principle) : 인터페이스 분리 원칙
    - 클라이언트가 자신이 이용하지 않는 메서드에 의존하지 않아야 한다는 원칙
    - 한 클래스는 자신이 사용하지 않는 인터페이스는 구현하지 말아야 한다.
5. DIP(The Dependency Inversion Principle) : 의존관계 역전 원칙
    - 상위클래스는 하위클래스에 의존해서는 안된다는 법칙
    - 의존 관계를 맺을 때 변화하기 쉬운 것 또는 자주 변화하는 것보다는 변화하기 어려운 것, 거의 변화가 없는 것에 의존하라는 것.

### 응집도와 결합도

- 모듈의 독립성을 판단하는 두 가지 지표(모듈 : 프로그램에서 하나의 기능을 수행하는 단위)
    - **응집도** : 모듈 내부의 기능적인 집중 정도(함수, 데이터 등의 구성 요소들 사이)
    - **결합도** : 모듈과 모듈간의 상호 의존 정도 또는 연관 관계
    - 결합도는 낮을 수록, 응집도는 높을 수록 이상적
- **높은 응집도**
    - 하나의 모듈, 클래스가 하나의 책임 또는 관심사에 집중이 되어 있다.
    - 변화가 일어날 때 해당 모듈에서 변하는 부분이 크다.
    - 응집도가 높은 모듈은 하나의 모듈이 단일 기능 하나만 담당
- **낮은 결합도**
    - 변화에 대응하는 속도가 높아지고, 구성이 깔끔해진다. 확장하기에도 편리하다.
    - 자신의 책임을 담당하는 데만 충실할 수 있다.

### 전략패턴

- 객체들이 할 수 있는 행위 각각에 대해 전략 클래스를 생성하고, 유사한 행위들을 캡슐화하는 인터페이스를 정의하여, 객체의 행위를 전략만 바꿔주면서 유연하게 확장하는 방법
- OCP에 위배되거나 시스템이 커져서 메서드의 중복문제가 발생할 경우를 대비해서 전략패턴을 사용한다.
- 런타임(Runtime)시에 타겟 메소드 변경
