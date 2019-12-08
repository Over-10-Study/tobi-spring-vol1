p.256 new Objecy[] {id}는 왜 쓰는가 가변인자를 안쓰고?

```java
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

//callback인 StatementStrategy stmt는 두가지 외부 정보가 필요하다
//Connection과 User다 이 둘은 각각 다른 곳에서 주어진다
//Connection은 template에서 만들어져 제공되고
//User는 내부로 클래스로 정의햅시 때문에 Scoping 규칙으로 Client에게서 받는다
```
## 클라이언트 템플릿 콜백
- 클라이언트의 역할은 템플릿 안에서 실행될 콜백 오브젝트를 만들고, 콜백이 참조할 정보 제공
- 탬플릿은 정해진 작업 흐름을 따라 작업을 진행하다 내부에서 생성한 참조정보로 콜백 호출
- 콜백은 클라이언트 메소드에 있는 정보(User)와 템플릿이 제공한 정보(Connection)를 이용해 작동

## 템플릿/콜백 특징:
- 매번 메소드 단위로 사용할 오브젝트를 새롭게 전달받는 것이 특징
- 콜백 오브젝트가 내부클래스로서 자신을 생성한 클라리언트 메소드 내의 종보를 직접 참조하는 것
- 클라리언트와 콜백이 강하게 결합된다는 면에서 일반 DI와 다르다

```java
//self 
public void deleteAll() throws SQLException {
		this.jdbcContext.workWithStatementStrategy(
			new StatementStrategy() {
				public PreparedStatement makePreparedStatement(Connection c)
						throws SQLException {
					return c.prepareStatement("delete from users");
				}
			}
		);
    }
//jdbc template
public void deleteAll() throws SQLEXception {
    this.jdbcTemplate.update(
        new PreparedStatementCreater() {
            public PreaparedStatement createPreparedStatement(Connection con) throws SQLException {
                return con.prepareStatement("delete from users");
            }
        }
    )
}

//self
PreaparedStatement ps = c.prepareStatement("insert into users(id, name, password) values(?,?,?)")
ps.setString(1, user.getId());
ps.setString(2, user.getName();
ps.setString(3, user.getPassword());

//jdbc template
this. jdbcTemplate.update("inser into users(id, name, password) values (?,?,?)", user.getId(), user.getName(), user.getPassword());

//self
public int getCount() throws SQLException  {
		Connection c = dataSource.getConnection();
	
		PreparedStatement ps = c.prepareStatement("select count(*) from users");

		ResultSet rs = ps.executeQuery();
		rs.next();
		int count = rs.getInt(1);

		rs.close();
		ps.close();
		c.close();
	
		return count;
    }

//jdbc Template
public int getCount() {
    return this.jdbcTemplate.query(new PreparedStatementCreate() {
        public PreparedStatement createPreparedStatement(Connection cont) throws SQLException {
            return con.prepareStatement("select count(*) from users");
        }
    }, new ResultSetExtractor<Integer> {
        public Integer extractData(ResultSet rs) throws SQLExcpetion, DataAccessEception {
            rs.next();
            return rs.getInt();
        }
    })
}

//self
public User get(String id) throws SQLException {
		Connection c = this.dataSource.getConnection();
		PreparedStatement ps = c
				.prepareStatement("select * from users where id = ?");
		ps.setString(1, id);
		
		ResultSet rs = ps.executeQuery();

		User user = null;
		if (rs.next()) {
			user = new User();
			user.setId(rs.getString("id"));
			user.setName(rs.getString("name"));
			user.setPassword(rs.getString("password"));
		}

		rs.close();
		ps.close();
		c.close();
		
		if (user == null) throw new EmptyResultDataAccessException(1);

		return user;
    }

public User get(String id) {
    return this.jdbcTempate.queryForObject("select * from users where id = ?", new Object[] {id}, 
    new RowMapper(ResultSet rs, int rowNum) throws SQLException) {
        User user = new User();
        user.setId(rs.getString("id"));
        user.setName(rs.getString("name"));
        user.setPassword(rs.getString("password"))
        return user;
    }
}

```

##예외처리
예외를 처리할 때 반드시 지켜야 할 핵심 원칙은 한 가지다. 모든 예외는 적절하게 복구되든지 아니면 작업을 중단시키고 운영자 또는 개발자에게 분명하게 통보돼야 한다.

컴파일에러 == 체크 에러 == 명시적으로 써주어야 하는 에러
런타임에러 == 언체크 에러 == 명시적으로 써도 되고 안써도되는 에러


```java
@Test
public void sqlExceptionTranslate() {
	dao.deleteAll();

	try {
		dao.add(user1);
		dao.add(user1);
	}
	catch(DuplicateKeyException ex) {
		SQLException sqlEx = (SQLExcpetion)ex.getRootCause();
		SQLExceptionTranslate set = new SQLErrorCodeExceptionTranslator(this.dataSource);
		assertThat(set.translate(null, null, sqlEx)),
		is(DuplicateKeyException.class)
	}
}
```

