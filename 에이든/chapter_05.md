# Chapter 05 - 서비스 추상화

> 자바에는 표준 스펙, 상용제품, 오픈소스를 통틀어서 사용방법과 형식은 다르지만 기능과 목적이 유사한 기술이 존재한다. 심지어는 같은 자바의 표준 기술 중에서도 플랫폼과 컨테스트 차이가 있거나 발전한 역사사 다르기 때문에 목적이 유사한 여러 기술이 공존하기도 한다.

## 5.1 사용자 레벨 관리 기능 추가
* 간단한 비즈니스 로직을 추가해보자.
	* 사용자의 레벨은 BASIC, SILVER, GOLD 세 가지 중 하나다.
	* 사용자가 처음 가입하면 BASIC 레벨이 된다.
	* 가입 후 50회 이상 로그인 하면 BASIC레벨 에서 SILVER 레벨이 된다.
	* SILVER 레벨이면서 30번 이상 추천을 받으면 GOLD레벨이 된다.
	* 사용자 레벨의 변경 작업은 일정한 주기를 가지고 일괄적으로 진행된다. 변경 작업 전에는 조건을 충족하더라도 레벨의 변경이 일어나지 않는다.

### Level Enum
* 문자열(?), 숫자(?) XXX
* 이런 방법은 안전하지 않다. 다른 종류의 정보를 넣는 실수를 해도 컴파일러가 체크해주지 못한다.
* Enum을 사용하자.

### 가장 많은 실수는 SQL문장
* update 쿼리문에서 where 절을 빼먹은 경우. 현재의 로우 내용이 바뀐 것만 확인한 테스트는 의미가 없다.
	* JdbcTemplate 의 update()가 돌려주는 리턴 값을 확인하자.
	* 테스트를 보강해서 원하는 사용자 외의 정보는 변경되지 않았음을 직접 확인한다.

### 비즈니스 로직은 어디에?
* DAO는 데이터를 어떻게 가져오고 조작할지를 다루는 곳이다. 비즈니스 로직은 서비스객체에 두자.
* UserService는 UserDao의 구현 클래스가 바뀌어도 영향받지 않도록 해야한다. 데이터 액세스 로직이 바뀌었다고 비즈니스 로직 코드를 수정하는 일이 있어서는 안된다.
![enter image description here](https://lh3.googleusercontent.com/0eLjMvUOxn97tEjav7AIIBT0BLsKvaDhWoCuhG0hoU-LFd6zOh6PcuG42MGtKExXDwa-szlHgBk)

```java
public void upgradeLevels() {
	List<User> users = userDao.getAll();
	for(User user : users) {
		Boolean changed = null;
		if (user.getLevel() == Level.BASIC && user.getLogin() >= 50) {
			// BASIC 레벨 업그레이드
			user.setLevel(Level.SILVER);
			changed = true;
		} else if (user.getLevle() == Levle.SILVER && user.getRecommend() >= 30) {
			// SILVER 레벨 업그레이드
			user.setLevel(Level.GOLD);
			changed = true; // 레벨 변경 플래그 설정
		} else if (user.getLevel == Levle.GOLD) {
			// GOLD 레벨은 병경이 없다.
			changed = false;
		} else {
			// 일치하는 조건이 없으면 변경 NO!
			changed = false;
		}
		
		if (changed) {
			// 레벨 변경이 있는 경우에만 호출
			userDao.update(user);
		}
	}
}
```
> 조건문도 많이 나오고 코드가 복잡하다.
> 오류를 쉽게 찾아내기 힘들다.
> 훌륭한 개발자는 아무리 간단해 보여도 실수할 수 있음을 알고 있기 때문에 테스트를 직접 만들어 확인한다.
> **테스트는 기준 값인 50, 30 처럼 경계가 되는 값의 전후로 선택을 한다.**

### 처음 가입하면... BASIC 어디에 담을까?
* UserDaoJdbc에 넣을까?
	* Dao는 DB에 정보를 넣고 읽는 방법에만 관심을 가져야지!
	* 비즈니스적인 의미를 지닌 정보를 설정하는 책임을 지면 안되!
* 그럼 UserService에 넣을까?
	* 서비스는 비즈니스 로직을 담당하게 하면 되니까 OK!
	* 그런데, 문제가 있다. 
		* User에 level 필드에 Basic이 기본 값으로?
		* 서비스의 add()를 호출 시 Basic으로 설정?
		* 이건 정하기 나름이다...

### 코드를 개선하자
* 코드에 중복된 부분은 없는가?
* 코드가 무엇을 하는 것인지 이해하기 불편하지 않는가?
* 코드가 자신이 있어야 할 자리에 있는가?
* 앞으로 변경이 일어난다면 어떤 것이 있고, 그 변화에 쉽게 대응 할 수 있게 작성되어 있는가?

### upgradeLevels() 메소드 코드의 문제점
* if/else/if/else... 블록이 읽기 불편하다.
	* 조건 블록이 레벨 개수만큼 반복된다.
	* 상당히 변화에 취약하다.

```java
// 리팩토링
// 각 오브젝트와 메소드가 각각 자기 몫의 책임을 맡아 일을 하는 구조로 만들자.
// 객체지향적인 코드는 다른 오브젝트에서 데이터를 가져와서 작업하지 않는다.
// 데이터를 갖고 있는 다른 오브젝트에게 작업을 해달라고 요청한다.
// 즉, 오브젝트에게 데이터를 요구하지 말고 작업을 요청하라!!!(객체지향 기본 원리)
public class UserService {
	public static final int MIN_LOGCOUNT_FOR_SILVER = 50;
	public static final int MIN_RECOMMEND_FOR_GOLD = 30;

	private UserDao userDao;

	public UserService(UserDao userDao) {
		this.userDao = userDao;
	}

	public void upgradeLevels() {
		final List<User> users = userDao.getAll();
		for (User user : users) {
			if (canUpgradeLevel(user)) {
				upgradeLevel(user);
			}
		}
	}

	public void add(User user) {
		if (user.getLevel() == null) {
			user.setLevel(Level.BASIC);
		}
		userDao.add(user);
	}

	private boolean canUpgradeLevel(User user) {
		final Level currentLevel = user.getLevel();

		switch (currentLevel) {
			case BASIC:
				return (user.getLogin() >= MIN_LOGCOUNT_FOR_SILVER);
			case SILVER:
				return (user.getRecommend() >= MIN_RECOMMEND_FOR_GOLD);
			case GOLD:
				return false;
			default:
				throw new IllegalArgumentException("Unknown Level: " + currentLevel);
		}
	}

	private void upgradeLevel(User user) {
		user.upgradeLevel();
		userDao.update(user);
	}

	public User get(String id){
		return userDao.get(id);
	}

}
```

* User 테스트
	* User 오브젝트는 스프링이 IoC로 관리해주는 오브젝트가 아니다.
	* 생성자를 호출해서 테스트 오브젝트를 만들면 된다.

```java
// 개선한 Service 테스트
public class UserServiceTest {

    @Autowired
    private UserService userService;
    @Autowired
    private UserDao dao;

    private List<User> users;

    @Before
    public void setUp() throws Exception {
        users = Arrays.asList(
		        // 가능한 경계 값을 이용한다.
                new User("bumjin", "박범진", "p1", Level.BASIC, MIN_LOGCOUNT_FOR_SILVER-1, 0),
                new User("joytouch", "강명성", "p1", Level.BASIC, MIN_LOGCOUNT_FOR_SILVER, 0),
                new User("erwins", "신승한", "p1", Level.SILVER, 60, MIN_RECOMMEND_FOR_GOLD-1),
                new User("madnite1", "이상호", "p1", Level.SILVER, 60, MIN_RECOMMEND_FOR_GOLD),
                new User("green", "오민규", "p1", Level.GOLD, 100, Integer.MAX_VALUE)
        );
    }

    @Test
    public void bean() throws Exception {

        assertThat(this.userService, is(notNullValue()));
    }

    @Test
    public void testUpgradeLevels() throws Exception {
        dao.deleteAll();

        for (User user : users) {
            dao.add(user);

        }

        userService.upgradeLevels();

        checkLevel(users.get(0), false);
        checkLevel(users.get(1), true);
        checkLevel(users.get(2), false);
        checkLevel(users.get(3), true);
        checkLevel(users.get(4), false);
    }


    @Test
    public void testAdd() throws Exception {
        dao.deleteAll();

        User userWithLevel = users.get(4);
        User userWithoutLevel = users.get(0);
        userWithoutLevel.setLevel(null);

        userService.add(userWithLevel);
        userService.add(userWithoutLevel);

        User userWithLevelRead = dao.get(userWithLevel.getId());
        User userWithoutLevelRead = dao.get(userWithoutLevel.getId());

        assertThat(userWithLevelRead.getLevel(), is(userWithLevel.getLevel()));
        assertThat(userWithoutLevelRead.getLevel(), is(Level.BASIC));
    }

	// upgraded: 다음 레벨로 업그레이드 될 것인가? 아닌가?
    private void checkLevel(User user, boolean upgraded) {
        User userUpdate = dao.get(user.getId());

        if (upgraded) {
            assertThat(userUpdate.getLevel(), is(user.getLevel().nextLevel())); // 다음 레벨이 무엇인지는 Level에게 물어보면 된다.
        } else {
            assertThat(userUpdate.getLevel(), is(user.getLevel()));
        }
    }
}
```

## 5.2 트랜잭션 서비스 추상화
> 정기 사용자 레벨 관리 작업을 수행하는 도중에 네트워크가 끊기거나 서버에 장애가 생겨서 작업을 완료 할 수 없다면, 
> 그 때까지 변경된 사용자의 레벨은 그대로 둘까?
> 모두 초기상태로 되돌려 놓아야 할까?

### 테스트용 UserService대역
* 테스트 UserService확장 클래스는 어떻게 만들까?
	* 코드를 복사?
	* 필요한 기능을 상속해서 테스트에 필요한 기능을 추가하고, 일부 메서드는 오버라이딩?
		* Production 코드의 접근제어자를 protected로 임의로 바꿔도 되는 것인가?

### 테스트의 실패의 원인? 트랜잭션
* 트랜잭션?
	* 더 이상 나눌 수 없는 단위 작업을 말한다.
	* 작업을 쪼개서 작은 단위로 만들 수 없다는 것은 트랜잭션의 핵심 속성인 원자성을 의미한다.
* 기술 요구사항 대로, 전체가 다 성공하든지 아니면 젠체가 다 실패하든지 해야한다.
* 여러번에 걸쳐서 진행할 수 있는 작업이 아니어야 한다.
* 중간에 예외가 발생해서 완료할 수 없다면 아예 작업이 시작되지 않은 것처럼 초기 상태로 돌려놔야 한다.
	* DB 서버가 아예 죽은거면(???)
* 여러 SQL문을 한 트랜잭션 처리를 한다면...
* 트랜잭션 롤백(Transaction Rollback)
	* DB에서 수행되기 전에 문제가 발생할 경우에는 앞에서 처리한 SQL 작업도 취소시켜야 한다.
* 트랜잭션 커밋(Transaction Commit)
	* SQL 수행 작업이 다 성공적으로 마무리됐다고 DB에 알려줘서 작업을 확정한다.
* 트랜잭션의 경계는 하나의 Connection 이 만들어지고 닫히는 범위 안에 존재한다.
* 하나의 DB커넥션 안에서 만들어지는 트랜잭션을 로컬 트랜잭션(Local Transaction)이라고 한다.

### 트랜잭션 동기화
![enter image description here](https://lh3.googleusercontent.com/a9S6BKKXpVRBnbCoj_FFN023DPn2S2Tlu-fWka9byfx8qPoed-vpKfAj8NYtVdhc7zn_-XDcJnc)
* 멀티 스레드 환경에서도 안전한 트랜잭션 동기화 방법을 구현하는 일이 기술 적으로 간단하지 않다.
* 다행히 스프링은 JdbcTemplate과 더불어 이런 트랜잭션 동기화 기능을 지원하는 간단한 유틸리티 메소드를 제공한다.

### 기술과 환경에 종속되는 트랜잭션 경계설정 코드
* 여러개의 DB를 사용한다면...
	* 별도의 트랜잭션 관리자를 통해 트랜잭션을 관리하는 **글로벌 트랜잭션** 방식을 사용해야 한다.
	* 여러개의 DB가 참여하는 작엄을 하나의 트랜잭션으로 만든다.
	* 글로벌 트랜잭션을 지원하는 JTA(Java Transaction API)를 제공한다.
![enter image description here](https://lh3.googleusercontent.com/J9Aedqq9OJz_qx7Ayzbdw0TaXNQDZRXcGJiawxk3v-0fKaPMzYSMguSsJmdwqVxMVCYzYjOAmpA)
* 하이버 네이트는 Connection을 직접 사용하지 않고 Session이라는 것을 사용하고, 독자적인 트랜잭션 관리 API를 사용한다.
* 이렇다면, 하이버네이트의 Session과 Transaction 오브젝트를 사용하는 트랜잭션 경계설정 코드로 변경 할 수 밖에 없다.![enter image description here](https://lh3.googleusercontent.com/7cybD534rXcrZ3N-ZH0Oft-WPNApqJ436RZaiZodx6JziGmC8F7hxdSVPpuTu5cgsltvaX3gH8Y)
* 다행히도 트랜잭션의 경계설정을 담당하는 코드는 일정한 패턴을 갖는 유사한 구조다.
* 이렇게 여러 기술의 사용 방법에 공통점이 있다면 추상화를 생각해볼 수 있다.
* 추상화란 하위 시스템의 공통점을 뽑아내서 분리시키는 것을 말한다. 그렇게 하면 하위 시스템이 어떤 것인지 알지 못해도, 또는 하위 시스템이 바뀌더라도 일관된 방법으로 접근할 수 있다.
* [추상화의 원칙](http://woowabros.github.io/study/2018/03/05/sdp-sap.html)
* 스프링의 트랜잭션 추상화 계층
![enter image description here](https://lh3.googleusercontent.com/-wgjxStyeuTK3AEDzoNLTw6qtIY7GQ7ROXStWc3Ax9gdiKqP0wlFT7k287ESa33PWj-GFVRyM6E)

### 트랜잭션 기술 설정의 분리
* 어떤 클래스든 스프링의 빈으로 등록할 때 먼저 검토해야 할 것은?
	* 싱글톤으로 만들어져 여러 스레드에서 동시에 사용해도 괜찮은가 하는 점이다.
	* 상태를 갖고 있고, 멀티스레드 환경에서 안전하지 않은 클래스를 빈으로 무작정 등록하면 심각한 문제가 발생한다.

## 5.3 서비스 추상화와 단일 책임 원칙
* 이렇게 기술과 서비스에 대한 추상화 기법을 이용하면 특정 기술환경에 종속되지 않는 포터블한 코드를 만들 수 있다.
![enter image description here](https://lh3.googleusercontent.com/UfskORCB7aRVkh8j_IA6-NGUFt4mQDJaZISfzqgk-9uHu1bnWrNu8K5BJx48rklK8b3W5UPTd-o)
* **단일 책임 원칙**
	* 하나의 모듈은 한 가지 책임을 가져야 한다.
	* 하나의 모듈이 바뀌는 이유는 한 가지여야 한다.
	* 어떤 변경이 필요할 때 수정 대상이 명확해진다.
	* 그래서 적절하게 책임과 관심이 다른 코드를 분리하고, 
	* 서로 영향을 주지 않도록 다양한 추상화 기법을 도입하고,
	* 애플리케이션 로직과 기술/환경을 분리한다.
* 좋은 코드를 설계하고 만들려면 꾸준한 노력이 필요하다.
* 기능이 동작한다고 해서 코드에 쉽게 만족하지 말고 계속 다음고 개선하려는 자세도 필요하다.

## 5.4 메일 서비스 추상화

### 테스트 대역의 종류와 특징
어렵네...

## 5.5 정리
* 비즈니스 로직을 담은 코드는 데이터 액세스 로직을 담은 코드와 분리되는 것이 바람직하다.
* 이를 위해서는 DAO의 기술 변화에 서비스 계층의 코드가 영향을 받지 않도록 인터페이스와 DI를 잘 활용해서 결합도를 낮춰야한다.
* 트랜잭션의 시작과 종료를 지정하는 일을 트랜잭션 경계설정이라고 한다. 트랜잭션 경계설정은 주로 비즈니스 로직 안에서 일어나는 경우가 많다.
