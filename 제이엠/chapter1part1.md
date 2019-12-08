#관심사의 분리
관심이 같은 것끼리 하나의 객체 안으로 또는 친한 객체로 모이게 하고, 관심이 다른 것은 가능한 따로 떨어뜨려 놓는다

#예시
UserDao의 세가지 관심:
1. DB와의 연결을 위한 커넥션 생성 --> DB Connector로 분리
2. SQL query 생성 및 실행 
3. resource 닫기

#UserDao의 중복제거 프로세스
1. 메소드 추출 --> 메소드로 getConnection이라는 관심을 한  곳에 집중
2. 상속구조로 분리하기 또는 확장하기 --> 상위클래스인 UserDao는 Sql 작성, 파라미터 바인딩, 쿼리 실행, 검색정보 전달이란느 관심에 집중하고 하위클래스인 NUserDao,DUserDao는 DB연결방법에 대해 집중
3. 클래스 추출
4. Interface로 느슨하게 의존관계 만들기

~~~java
//1.메소드 추출 예시
public class UserDao {
	public void add(User user) throws ClassNotFoundException, SQLException {
		Connection c = getConnection();

		PreparedStatement ps = c.prepareStatement(
			"insert into users(id, name, password) values(?,?,?)");
		ps.setString(1, user.getId());
		ps.setString(2, user.getName());
		ps.setString(3, user.getPassword());

		ps.executeUpdate();

		ps.close();
		c.close();
	}


	public User get(String id) throws ClassNotFoundException, SQLException {
		Connection c = getConnection();
		PreparedStatement ps = c
				.prepareStatement("select * from users where id = ?");
		ps.setString(1, id);

		ResultSet rs = ps.executeQuery();
		rs.next();
		User user = new User();
		user.setId(rs.getString("id"));
		user.setName(rs.getString("name"));
		user.setPassword(rs.getString("password"));

		rs.close();
		ps.close();
		c.close();

		return user;
	}


	private Connection getConnection() throws ClassNotFoundException,
			SQLException {
		Class.forName("com.mysql.jdbc.Driver");
		Connection c = DriverManager.getConnection("jdbc:mysql://localhost/springbook?characterEncoding=UTF-8", "spring",
				"book");
		return c;
	}
~~~

#2. 확장으로 분리하기
공통된 부분인 getConnection()를 추상메소드로 분리하고 자식클래스들에게 구현하게 하여 확장에 대비한다.
~~~java
public abstract class UserDao {
	public void add(User user) throws ClassNotFoundException, SQLException {
		Connection c = getConnection();

		PreparedStatement ps = c.prepareStatement(
			"insert into users(id, name, password) values(?,?,?)");
		ps.setString(1, user.getId());
		ps.setString(2, user.getName());
		ps.setString(3, user.getPassword());

		ps.executeUpdate();

		ps.close();
		c.close();
	}


	public User get(String id) throws ClassNotFoundException, SQLException {
		Connection c = getConnection();
		PreparedStatement ps = c
				.prepareStatement("select * from users where id = ?");
		ps.setString(1, id);

		ResultSet rs = ps.executeQuery();
		rs.next();
		User user = new User();
		user.setId(rs.getString("id"));
		user.setName(rs.getString("name"));
		user.setPassword(rs.getString("password"));

		rs.close();
		ps.close();
		c.close();

		return user;
	}

	abstract protected Connection getConnection() throws ClassNotFoundException, SQLException ; //이 부분을 하위클래스에서 정의
~~~
# 상위하위클래스그로 나누기 == template method
기본적인 로직의 흐름(커넥션 가져오기 . SQL 생성. 실행， 반환)을 만들고， 그 기능의 일부를 추상 메소드나 오버라이딩이 가능한 protected 메소드 등으로만든 뒤 서브클래스에서 이런 메소드를 펼요에 맞게 구현해서 사용하도록 하는 방법을 디자인 패턴에서 템플릿 메소드 패턴template method pattern이라고 한다.

미스터코를 깨워라:

a. 미스터코 방에 들어간다(상위클래스)
b. 미스터코에게 물을 뿌린다(하위클래스)

a. 미스터코 방에 들어간다(상위클래스)
b. 라면을 끓인다(하위클래스)

#factory method pattern vs template method pattern
https://softwareengineering.stackexchange.com/questions/340099/factory-method-is-a-specialization-of-template-method-how

Template Method

-Relies on inheritance.
-Defines the steps of an algorithm, and leaves the task of implementing them to subclasses.

Factory Method

-Relies on inheritance.
-A superclass defines an interface to create an object. Subclasses decide which concrete class to instantiate.

"factory method pattern is a specilization of template method
pattern"

내 생각: template method, factory method 비슷한데 후자는 상위클래스에서 정의한 객체를 반환한다!

hook method: 선택적 override
abstract method: 무조건 override

#상속의 문제점:
1. 자바는 다중 상속을 허용하지 않는다. 따라서 단지 커넥션이라는 관심사를 분리하기 위해 상속구조를 만든다면 다른 목적의 상속구조가 어려워진다.
2. 상하위 클래스의 관계는 밀접하다 == 분리가 제대로 안된다. 서브클래스가 상위클래스의 메소드를 호출 할 수 있기 때문이다. 변경되면 어쩔텐가? 
3. UsderDao는 NUserDao, DUserDao에게만 확장한다
만약 messageDao, cartDao가 생긴다면 getConnection()이 무수히 생긴다!

#SimpleConnectionMaker의 문제점
1. method 이름이 다를 수 있음
2. 디비 커넥션을 제공하는 클래스를 어떤 것인지를 UserDao가 구체적으로 알고 있어야한다 (알고 있으면 확장이 어렵다)
UserDao에 SimpleConnectionMaker라는 클래스 타입의 인스턴스 변수까지 정의해놓고 있으니 n사용 UserDao를 다시 만들어야 한다
 너무 많이 알고있따: 어떤 클래스가 쓰일지, 메소드 이름이 먼지

 => 추상화해라: 공통적인 성격을 뽑아내어 이를 따로 분리하는 작업...interface는 추상화
 인터퍼이스는 어떤 일을 하겠다는 기능만 정해놓음. 구현방법은 모름

 #new NConnectionMaker()
 바로 UserDao가 어떤 ConnectionMaker구현 클래스의 오브젝트를 이용하게 할지를 결정을 하는 것이다. 특정 구현 클래스의 사이의 관계를 설정

 #의존관계
 알고있다==의존관계==코드내에 클래스 네임이 있다
 ~~~java
 public class UserDao{
     public UserDao() {
         this.connectionMaker = new DConnectionMaker()
     }
 }
 ~~~

 #오브젝트 사이에 관계를 맺어야 한다!
 클래스 사이에 관계를 만들어진 것은 아니고 단지 오브젝트 사이에 다이내믹한 관계가 만들어지는 것이다. 클래스 사이의 관계는 코드에 다른 클래스 이름이 나타나기 때문에 만들어지는 것이다. 하지만 오브젝트 사이의 관계를 그렇지 않다. 코드에서는 특정 클래스를 전혀 알지 못하더라도 해당 클래스가 구현한 인터페이스를 사용했다면, 그 클래스이 오브젝트를 인터페이스 타입으로 받아서 사용 할 수 있다. 다형성 때문이다

 #개방폐쇄 원칙
 클래스나 모듈은 확장에는 열려 있어야 하고 변경에는 닫혀 있어야 한다

 #응집도와 결합도
 응집도가 높다는 건 하나의 모듈, 클래스가 하나의 책임 또는 관심사에만 집중되어 있다는 뜻
 변경이 일어날때 집중된 곳에서 많은 변화가 일어난다. 얇게 넓게 퍼진거 vs 두껍게 집중적으로

 결합도는 느슨한 연결
 결합도는 하나의 오브젝트라 변경이 일어날 때에 관계를 맺고 있는 다른 오브젝트에게 변화를 요구하는 정도
 객체안에서 직접 생성하는 것은 결합도가 높다
