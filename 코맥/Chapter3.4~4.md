## 3.4 컨텍스트와 DI
### 3.4.1 JdbcContext의 분리
전략 패턴의 구조로 보면
- 클라이언트: ```UserDao```의 메소드
- 전략: 익명 클래스로 만들어진 것
- 컨텍스트: ```jdbcContextWithStatementStrategy()``` 메소드

>> 전략 패턴 복습
>> - 컨텍스트: 자신의 기능을 수행하는데 필요한 기능 중에서 변경 가능한 알고리즘(보통 인터페이스로 구현)
>> - 전략: 컨텍스트를 구현한 클래스(실제 기능)
>> - 클라이언트: 컨텍스트를 사용하는 주체

위의 ```jdbcContextWithStatementStrategy()```는 다른 DAO에서도 사용가능하므로, 이를 ```JdbcContext``` 클래스로 분리해보자.

```java
// JDBC 작업 흐름을 분리해서 만든 JdbcContext 클래스

public class JdbcContext {
  private DataSource datasource;

  // DataSource 타입 빈을 DI 받을 수 있게 준비해둔다.
  public void setDataSource(DataSource dataSource) {
    this.dataSource = dataSource;
  }

  public void workWithStatementStrategy(StatementStrategy stmt) throws SQLException {
    Connection c = null;
    PreparedStatement ps = null;

    try {
      c = this.dataSource.getConnection();
      ps = stmt.makePreparedStatement(c);
      ps.executeUpdate();
    } catch (SQLException e) {
      throw e;
    } finally {
      if (ps != null) try { ps.close(); } catch (SQLException e) {} }
      if (c != null) try { c.close(); } catch (SQLException e) {} }
    }
  }
}
```

```java
// jdbcContext를 DI 받아서 사용하도록 만든 UserDao

public class UserDao {
  // ...
  private JdbcContext jdbcContext;

  // JdbcContext를 DI 받도록 만든다.
  public void setJdbcContext(JdbcContext jdbcContext) {
    this.jdbcContext = jdbcContext;
  }

  public void add(final User user) throws SQLException {
    // DI 받은 JdbcContext의 컨텍스트 메소드를 사용하도록 변경한다.
    this.jdbcContext.workWithStatementStrategy(
      new StatementStrategy() { ... }
    );
  }

  public void delete() throws SQLException {
    this.jdbcContext.workWithStatementStrategy(
      new StatementStrategy() { ... }
    );
  }
}
```

- ```JdbcContext``` 클래스는 구현 방법이 바뀔 가능성이 없으므로 구현 클래스로 만들었다.
- ```UserDao``` 클래스는 ```DataSource```빈을 직접 의존하지 않고, ```JdbcContext``` 빈을 통해 사용한다.

### 3.4.2 ```JdbcContext```의 특별한 DI
위에서 살펴본 ```JdbcContext```를 분리하면서 사용했던 DI 방식은
- 이전과 달리 인터페이스를 사용하지 않고 직접 구현 클래스를 사용했다.
- ```UserDao```와 ```JdbcContext```는 클래스 레벨에서 의존관계가 결정된다.
- 의존 오브젝트의 구현 클래스를 변경할 수 없다.

그렇다면, DI에서 인터페이스를 사용하지 않아도 될까?

의존관계 주입(DI)의 개념을 돌이켜 보면
- 인터페이스를 사이에 둬서 클래스 레벨에서는 의존관계가 고정되지 않계 하고,
- 런타임 시에 의존할 오브젝트와의 관계를 다이내믹하게 주입해주는 것이다.

인터페이스를 사용하지 않으면 엄밀히 말해서 온전한 DI라고 볼 수 없다. 하지만 스프링의 DI는 넓게 보면 IoC라는 개념으로 포괄한다.
- IoC는 객체의 생성과 관계설정에 대한 제어권한을 오브젝트에서 제거하고 **외부로 위임**

인터페이스를 사용하지 않았는데 ```JdbcContext```클래스를 ```UserDao```와 DI 구조로 만들어야 하는 이유는 두 가지이다.
- ```JdbcContext```는 JDBC 컨텍스트 메소드를 제공해주는 일종의 서비스 오브젝트이므로, 싱글톤으로 등록해서 여러 오브젝트에서 공유하는 것이 이상적이다.
  - ```JdbcContext```는 ```dataSource```라는 인스턴수 변수가 있지만, 이는 읽기 전용이므로 결과적으로 변경되는 상태 정보가 없다고 볼 수 있다.
- ```JdbcContext```가 **DI를 통해 다른 빈에 의존하고** 있기 때문이다.
  - ```JdbcContext```는 ```dataSource``` 프로퍼티를 통해 ```DataSource``` 오브젝트를 주입받도록 되어 있다.
  - DI를 위해서는 **주입되는 오브젝트와 주입받는 오브젝트 모두 스프링 빈으로 등록되어야 한다.**
    - 스프링이 생성하고 관리하는 IoC 대상이어야 DI에 참여할 수 있기 때문

JdbcContext를 DI로 주입하지 않는 방법도 있다. 그러면 위의 두 가지를 모두 ```UserDao``` 클래스가 담당해야한다.
- ```JdbcContext```를 DAO마다 하나씩 가지고 있게 하면 메모리면에서나 속도면에서 크게 성능저하를 보이지는 않으므로, 싱글톤으로 만들지 않아도 큰 문제는 없다.
- ```UserDao```에서 ```dataSource```를 DI받아 ```JdbcContext```에 직접 수정자 메소드로 주입해준다.

```java
// JdbcContext 생성과 DI 작업을 수행하는 setDataSource() 메소드

public class UserDao {
  // ...
  private JdbcContext jdbcContext;

  // 수정자 메소드이면서 JdbcContext에 대한 생성, DI 작업을 동시에 수행한다.
  public void setDataSource(DataSource dataSource) {
    // jdbcContext 생성(IoC)
    this.jdbcContext = new JdbcContext();

    // 의존 오브젝트 주입(DI)
    this.jdbcContext.setDataSource(dataSource);

    // 아직 JdbcContext를 적용하지 않은 메소드를 위해 저장해둔다.
    this.dataSource = dataSource;
  }
}
```

UserDao 메소드는 ```JdbcContext```가 외부에서 만들어졌는지, 내부에서 만들어졌는지 알 필요가 없고 필요에 따라 사용하기만 하면 된다.

## 3.5 템플릿과 콜백
지금까지 살펴본 ```UserDao```, ```StatementStrategy```, ```JdbcContext```는 전략 패턴의 기본 구조에 익명 클래스를 활용한 방식이다. 이러한 방식을 스프링에서는 **템플릿/콜백 패턴** 이라 한다.
- 전략 패턴의 컨텍스트: 템플릿
- 익명 내부 클래스로 만들어지는 오브젝트: 콜백

>> 템플릿
>> - 어떤 목적을 위해 미리 만들어둔 모양이 있는 틀을 가리킨다.
>> - 프로그래밍에서는 고정된 틀 안에 바꿀 수 있는 부분을 넣어서 사용하는 경우를 말한다.
>> - 템플릿 메소드 패턴은 고정된 틀의 로직을 가진 템플릿 메소드를 부모클래스에 두고, 바뀌는 부분을 자식클레서의 메소드에 두는 구조이다.

>> 콜백
>> - 실행되는 것을 목적으로 다른 오브젝트의 메소드에 전달되는 오브젝트를 말한다.
>> - 보통 파라미터로 전달되지만 값을 참조하기 위한 것이 아니라 특정 로직을 담은 메소드를 실행시키기 위함이다.
>> - 자바는 메소드를 파라미터로 전달할 수 없으므로 오브젝트를 전달한다. (functional object)

### 3.5.1 템플릿/콜백의 동작원리
먼저 템플리/콜백의 특징은 다음과 같다.
- 콜백은 보통 단일 메소드의 인터페이스를 사용한다.
  - 템플릿의 작업 흐름 중 특정 기능을 위해 한 번만 호출되는 경우가 일반적이기 때문
  - 콜백은 보통 하나의 메소드를 가진 인터페이스를 구현한 익명 내부 클래스로 만들어진다.
- 콜백 인터페이스의 메소드에는 보통 파라미터가 있다.
  - 이 파라미터는 템플릿의 작업 흐름 중에 만들어지는 컨텍스트 정보를 전달받을 때 사용된다.

<3-7 그림>

- 클라이언트: 템플릿 안에서 실행될 로직을 담은 콜백 오브젝트를 만들고, 콜백이 참조할 정보를 제공한다.
- 템플릿: 콜백 오브젝트를 호출
- 콜백: 클라이언트 메소드에 있는 정보와 템플릿이 제공한 참조 정보를 이용하여 작업 수행 후 그 결과를 템플릿에 전달
- 템플릿: 콜백이 돌려준 정보를 사용해서 작업을 마저 수행한다. 필요에 따라 최종 결과를 클라이언트에 전달

클라이언트가 템플릿 메소드를 호출하면서 콜백 오브젝트를 전달하는 것은 메소드 레벨에서 일어나는 DI이다.
- 메소드 레벨 DI: 템플릿에 인스턴스 변수를 두고 의존 오브젝트를 수정자 메소드로 받아서 사용한다.
- 템플릿/콜백: 매번 메소드 단위로 사용할 오브젝트를 새롭게 전달받는다. 그리고 자신을 생성한 클라이언트 메소드를 직접 참조한다.

템플릿/콜백 방식은 전략 패턴과 DI의 장점을 익명 내부 클래스 사용 전략과 결합한 독특한 활용법이다.

### 3.5.2 펀리한 콜백의 재활용
템플릿/콜백을 사용하면 클라이언트의 메소드가 간결해지고 최소한의 데이터 액세스 로직만 갖는 장점이 있지만, 익명 클래스로 인해 가독성이 떨어진다.

복잡한 익명 내부 클래스의 사용을 최소화하는 방법을 알아보자.

```java
// 익명 내부 클래스를 사용한 클라이언트 코드

public void deleteAll() throws SQLException {
  this.jdbcContext.workWithStatementStrategy(
    // 변하지 않는 콜백 클래스 정의와 오브젝트 생성
    new StatementStrategy() {
      public PreparedStatement makePreparedStatement(Connection c) throws SQLException {
        // 변하는 SQL 문장
        return c.preparedStatement("delete from users");
      }
    }
  )
}
```

위와 같이 콜백을 구현한다면 SQL 문장이 달라질 때마다 새로운 콜백을 만들어야 하므로, 많은 수의 콜백이 생길 수 있다. 이를 중복 제거해보자.

```java
// 변하지 않는 부분을 분리시킨 deleteAll() 메소드

public void deleteAll() throws SQLException {
  // 변하는 SQL 문장
  executeSql("delete from users");
}

private void executeSql(final String query) throws SQLException {
  this.jdbcContext.workWithStatementStrategy(
    new StatementStrategy() {
      public PreparedStatement makePreparedStatement(Connection c) throws SQLException {
        return c.preparedStatement(query);
      }
    }
  )
}
```

위 코드에서 주의할 점은 익명 클래스 내부에서 사용하기 위해 ```query``` 문장을 **final** 로 받은 것이다.

SQL 문장을 사용하는 DAO는 ```UserDao``` 클래스만이 아닐 것이다. 그러므로 재사용성을 높이기 위해 ```executeSql()``` 메소드를 ```JdbcContext``` 클래스로 옮겨보자.

```java
public void deleteAll() throws SQLException {
  this.jdbcContext.executeSql("delete from users");
}
```

중복 코드 제거하기
- 먼저 메소드로 분리하는 시도
- 일부 작업을 필요에 따라 바꾸어 사용해야 한다면 인터페이스를 사이에 두고 전략 패턴 적용하고 DI로 의존관계 관리
- 바뀌는 부분이 한 애플리케이션 안에서 동시에 여러 종류가 만들어질 수 있다면 템플릿/콜백 패턴
  - 가장 전형적인 템플릿/콜백 패턴의 후보는 ```try/catch/finally``` 블록을 사용하는 코드

### 3.5.3 템플릿/콜백의 응용 - ```Calculator``` 클래스 개선하기

```java
// 파일의 숫자 합을 계산한느 코드의 테스트

public class CalcSumTest {
  @Test
  public void sumOfNumbers() throws IOException {
    Calculator calculator = new Calculator();
    int sum = calculator.calcSum(getClass().getResource("numbers.txt").getPath());
    assertThat(sum, is(10));
  }
}
```

```java
// 처음 만든 Calculator 클래스 코드

public class Calculator {
  public Integer calcSum(String filepath) throws IOException {
    // 한 줄씩 읽기 편하게 BufferedReader로 파일을 가져온다.
    BufferedReader br = new BufferedReader(new FileReader(filepath));
    Integer sum = 0;
    String line = null;
    // 마지막 라인까지 한 줄씩 읽어가면서 숫자를 더한다.
    while((line = br.readLine()) != null) {
      sum += Integer.valueOf(line);
    }

    // 한 번 연 파일은 반드시 닫아준다.
    br.close();
    return sum;
  }
}
```

```java
// try/catch/finally를 적용한 calcSum() 메소드

public Integer calcSum(String filepath) throws IOException {
  BufferedReader br = null;

  try {
    br = new BufferedReader(new FileReader(filepath));
    Intger sum = 0;
    String line = null;
    while((line = br.readLine()) != null) {
      sum += Integer.valueOf(line);
    }
    return sum;
  }
  catch (IOException e) {
    System.out.print(e.getMessage());
    throw e;
  }
  finally {
    // BufferedReader 오브젝트가 생성되기 전에 예외가 발생 할 수 있으므로 null 체크를 먼저함
    if (br != null) {
      try { br.close(); }
      catch (IOException e) { System.out.println(e.getMessage()); }
    }
  }
}
```

- 중복 제거와 템플릿/콜백 설계

```java
// BufferedReader를 전달받는 콜백 인터페이스

public interface BufferedReaderCallback {
  Integer doSomethingWithReader(BufferedReader br) throws IOException;
}
```

```java
// BufferedReaderCallback을 사용하는 템플릿 메소드

public Integer fileReadTemplate(String filepath, BufferedReaderCallback callback)
    throws IOException {
  BufferedReader br = null;

  try {
    br = new BufferedReader(new FileReader(filepath));
    // 콜백 오브젝트 호출.
    // 템플릿에서 만든 컨텍스트 정보인 BufferedReader를 전달해 주고 콜백의 작업 결과를 받아준다.
    int ret = callback.doSomethingWithReader(br);
    return ret;
  }
  catch (IOException e) {
    System.out.print(e.getMessage());
    throw e;
  }
  finally {
    if (br != null) {
      try { br.close(); }
      catch (IOException e) { System.out.println(e.getMessage()); }
    }
  }
}
```

```java
// 템플릿/콜백을 적용한 calcSum() 메소드

public Integer calcSum(String filepath) throws IOException {
  BufferedReaderCallback sumCallback = new BufferedReaderCallback() {
    public Integer doSomethingWithReader(BufferedReader br) throws IOException {
      Integer sum = 0;
      String line = null;
      while ((line = br.readLine()) != null) {
        sum += Integer.valueOf(line);
      }
      return sum;
    }
  };
  return fileReadTemplate(filepath, sumCallback);
}
```

- ```multiply()``` 메소드 추가

```java
// 새로운 테스트 메소드를 추가한 CalcSumTest

public class CalcSumTest {
  Calculator calculator;
  String numFilepath;

  @Before
  public void setUp() {
    this.calculator = new Calculator();
    this.numFilepath = getClass().getResource("numbers.txt").getPath();
  }

  @Test
  public void sumOfnumbers() throws IOException {
    assertThat(calculator.calcSum(this.numFilepath), is(10));
  }

  @Test
  public void multiplyOfNumbers() throws IOException {
    assertThat(calculator.calcMultiply(this.numFilepath), is(24));
  }
}
```

```java
// 곱을 계산하는 콜백을 가진 calcMultiply() 메소드

public Integer calcMultiply(String filepath) throws IOException {
  BufferedReaderCallback multiplyCallback = new BufferedReaderCallback() {
    pulbic Integer doSomethingWithReader(BufferedReader br) throws IOException {
      Integer multiply = 1;
      String line = null;
      while ((line = br.readLine()) != null) {
        multiply *= Integer.valueOf(line);
      }
      return multiply;
    }
  };
  return fileReadTemplate(filepath, multiplyCallback);
}
```

- 템플릿/콜백의 재설계
위에서 만든 ```calcSum()```과 ```calcMultiply()``` 메소드는 유사한 점이 매우 많다. 템플릿과 콜백을 찾아낼 때는
- 변하는 코드의 경계를 찾고 그 경계를 사이에 두고 주고받는 일정한 정보가 있는지 확인

```java
// 라인별 작업을 정의한 콜백 인터페이스

public interface LineCallback {
  Integer doSomethingWithReader(String line, Integer value);
}
```

```java
// LineCallback을 사용하는 템플릿

// initVal: 계산 결과를 저장할 변수의 초기값
public Integer lineReadTemplate(String filepath, LineCallback callback, int initVal) throws IOException {
  BufferedReader br = null;
  try {
    br = new BufferedReader(new FileReader(filepath));
    Integer res = initVal;
    String line = null;
    // 파일의 각 라인을 루프를 돌면서 가져오는 것도 템플릿이 담당한다.
    while ((line = br.readLine()) != null) {
      // res: 콜백이 계산한 값을 저장해뒀다가 다음 라인 계산에 다시 사용한다.
      // line: 각 라인의 내용을 가지고 계산하는 작업만 콜백에게 맡긴다.
      res = callback.doSomethingWithReader(line, res);
    }
    return res;
  }
  catch(IOException) { ... }
  finally { ... }
}
```

```java
// lineReadTemplate()을 사용하도록 수정한 calSum(), calcMultiply() 메소드

public Integer calcSum(String filepath) throws IOException {
  LineCallback sumCallback = new LineCallback() {
    public Integer doSomethingWithReader(String line, Integer value) {
      return value + Integer.valueOf(line);
    }
  };
  return lineReadTemplate(filepath, sumCallback, 0);
}

public Integer calcMultiply(String filepath) throws IOException {
  LineCallback multiplyCallback = new LineCallback() {
    public Integer doSomethingWithReader(String line, Integer value) {
      return value * Integer.valueOf(line);
    }
  };
  return lineReadTemplate(filepath, multiplyCallback, 1);
}
```

- 제네릭스를 이용한 콜백 인터페이스
위의 ```calcSum()``` 메소드를 숫자 뿐아니라 문자열을 합쳐주는 기능을 하고 싶다면 제네릭스를 사용해서 쉽게 확장할 수 있다.

```java
// 타입 파라미터를 적용한 LineCallback

public interface LineCallback<T> {
  T doSomethingWithReader(String line, T value);
}
```

```java
// 타입 파라미터를 추가해서 제네릭 모소드로 만든 lineReadTemplate()

public <T> T lineReadTemplate(String filepath, LineCallback<T> callback, T initVal) throws IOException {
  BufferedReader br = null;
  try {
    br = new BufferedReader(new FileReader(filepath));
    T res = initVal;
    String line = null;
    while ((line = br.readLine()) != null) {
      res = callback.doSomethingWithReader(line, res);
    }
    return res;
  }
  catch (IOException e) { ... }
  finally { ... }
}
```

## 3.6 스프링의 JdbcTemplate
- 스프링은 거의 모둔 종류의 JDBC 코드에 사용 가능한 템플릿과 콜백을 제공한다.
- 자주 사용되는 패턴을 가진 콜백은 다시 템플릿에 결합시켜서 간단한 메소드 호출만으로 사용이 가능하다.
- 스프링이 제공하는 JDBC 코드용 기본 템플릿은 ```JdbcTemplate``` 이다.

```java
// JdbcTemplate의 초기화를 위한 코드

public class UserDao {
  // ...

  private JdbcTemplate jdbcTemplate;

  public void setDataSource(DataSource dataSource) {
    this.jdbcTemplate = new JdbcTemplate(dataSource);

    this.dataSource = dataSource;
  }
}
```

### 3.6.1 ```update()```

```java
// JdbcTemplate을 적용한 deleteAll() 메소드ㅜ

public void deleteAll() {
  this.jdbcTemplate.update(
    new PreparedStatementCreator() {
      public PreparedStatement createPreparedStatement(Connection con) throws SQLException {
        return con.prepareStatement("delete from users");
      }
    }
  );
}
```

```java
// 내장 콜백을 사용하는 update()로 변경한 deleteAll() 메소드

public void deleteAll() {
  this.jdbcTemplate.update("delete from users");
}
```

```java
// 내장 콜백을 사용하는 update를 사용한 add() 메소드 내부

this.jdbcTemplate.update("insert into users(id, name, password) values(?,?,?)",
  user.getId(), user.getName(), user.getPassword());
```

### 3.6.2 ```queryForInt()```

```java
// JdbcTemplate을 이용해 만든 getCount()

public int getCount() {
  // 첫 번째 콜백. Statement 생성
  return this.jdbcTemplate.query(new PreparedStatementCreator() {
    public PreparedStatement createPreparedStatement(Connection con) throws SQLException {
      return con.prepareStatement("select count(*) from users");
    }
  }, new ResultSetExtractor<Integer>() {   // 두 번째 콜백. ResultSet으로부터 값 추출
    public Integer extractData(Result rs) throws SQLException, DataAccessException {
      rs.next();
      return rs.getInt(1);
    }
  });
}
```

- 원래 ```getCount()``` 메소드에 있던 코드 중에서 변하는 부분만 콜백으로 만들어져 제공된다고 생각하면 된다.
- ```ResultSetExtractor```는 제네릭스 타입 파라미터를 갖는데, ```ResultSet```에서 추출할 수 있는 값의 타입이 다양하기 때문이다.
  - 해당 제네릭스 타입으로 ```query()``` 메소드에 적용된다.
- ```JdbcTemplate```는 위 기능을 한 번에 제공하는 ```queryForInt()``` 메소드를 제공한다.

```java
// queryForInt()를 사용하도록 수정한 getCount()

public int getCount() {
  return this.jdbcTemplate.queryForInt("select count(*) from users");
}
```

- ```JdbcTemplate```는 스프링에서 제공하지만 DI를 할 필요 없이 사용하는 오브젝트에서 직접 생성해주면 된다.

### 3.6.3 ```queryForObject()```
이 메소드는 ```User```와 같은 하나의 오브젝트를 결과로 치환한다.
- ```RowMapper``` 콜백을 사용한다.
  - 이는 모든 템플릿으로부터 ```ResultSet```을 전달받고, 필요한 정보를 추출한다.
  - ```ResultSet```은 로우 하나를 매핑하므로 여러 번 호출해야한다.

```java
// queryForObject()와 RowMapper를 적용한 get() 메소드

public User get(String id) {
  return this.jdbcTemplate.queryForObject("select * from users where id = ?",
      new Object[] {id},   // SQL에 바인딩할 파라미터 값. 가변인자 대신 배열을 사용한다.
      // ResultSet 한 로우의 결과를 오브젝트에 매핑해주는 RowMapper 콜백
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

- ```new Object[] {id}```: 첫 번째 인자의 id값의 ?에 해당하는 값을 전달한다.
- ```RowMapper``` 콜백은 앞의 두 파라미터를 가지고 만들어진다.
- ```queryForObject()```는 SQL을 실행하면 하나의 로우만 얻을 것을 기대한다.
- 위 코드의 문제점은 조회 결과가 없을 때 ```EmptyResultDataAccessException```을 전지지 않는 것이다.

### 3.6.4 ```query()```
