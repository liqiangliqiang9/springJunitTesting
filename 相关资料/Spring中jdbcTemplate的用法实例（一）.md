## 一、首先配置JdbcTemplate；
>要使用Jdbctemplate 对象来完成jdbc 操作。通常情况下，有三种种方式得到JdbcTemplate 对象。
>>* 第一种方式：我们可以在自己定义的DAO 实现类中注入一个DataSource 引用来完 成JdbcTemplate 的实例化。也就是它是从外部“注入” DataSource 到DAO 中，然后 自己实例化JdbcTemplate，然后将DataSource 设置到JdbcTemplate 对象中。
>>* 第二种方式： 在 Spring 的 IoC 容器中配置一个 JdbcTemplate 的 bean，将 DataSource 注入进来，然后再把JdbcTemplate 注入到自定义DAO 中。
>>* 第三种方式: Spring 提供了org.springframework.jdbc.core.support.JdbcDaoSupport 类 ， 这 个 类 中 定 义 了 JdbcTemplate 属性，也定义了DataSource 属性，当设置DataSource 属性的时候，会创 建jdbcTemplate 的实例，所以我们自己编写的DAO 只需要继承JdbcDaoSupport 类， 然后注入DataSource 即可。提倡采用第三种方法。虽然下面的用法中采用了前两种方法
---------------------
 ### 配置方法有3种：
#### 方法1
* Java代码 
```java
public class UserServiceImpl implements UserService {  
  
    private JdbcTemplate jdbcTemplate;  
      
    public JdbcTemplate getJdbcTemplate() {  
        return jdbcTemplate;  
    }  
  
                //注入方法1     
    public void setJdbcTemplate(JdbcTemplate jdbcTemplate) {  
        this.jdbcTemplate = jdbcTemplate;  
    }  
  
               //其它方法这里省略……  
}
```
* spring配置文件为：
Xml代码
```xml
<bean id="jdbcTemplate" class="org.springframework.jdbc.core.JdbcTemplate">  
        <property name = "dataSource" ref="dataSource">  
</bean>  
<bean id="userService" class="com.hxzy.account.jdbcTemplate.UserServiceImpl">  
     <property name="jdbcTemplate" ref="jdbcTemplate"/>  
</bean>  
``` 
#### 方法2、
* Java代码
```java
public class UserServiceImpl implements UserService {  
  
        private JdbcTemplate jdbcTemplate;  
          
        //注入方法2  
        public void setDataSource(DataSource dataSource) {  
                   this.jdbcTemplate = new JdbcTemplate(dataSource);  
        }  
       
       //其它方法省略……  
}  
``` 
* spring配置文件为：
Xml代码
```xml
<bean id="userService" class="com.hxzy.account.jdbcTemplate.UserServiceImpl">  
       <property name="dataSource" ref="dataSource"/>  
</bean>
```  
 
#### 方法3：继承JdbcDaoSupport，其内部有个JdbcTemplate ，需要注入DataSource 属性来实例化。
* Java代码
```java
public class UserDaoImpl extends JdbcDaoSupport implements UserDao {  
  
    @Override  
    public void save(User user) {  
        String sql = null;  
        this.getJdbcTemplate().update(sql);  
    }  
        //其它方法省略……  
}
```
* spring配置文件：
Xml代码
```xml
<bean id="userDao" class="com.hxzy.account.jdbcTemplate.UserDaoImpl">  
           <property name="dataSource" ref="dataSource"/>  
</bean>  
 ```
## 二、常用方法使用
**【注意：】jdbcTemplate 中的sql均是用“?”做占位符的**
* domain User：
Java代码
```java
public class User {  
    private int id;  
    private String username;  
    private String password;  
    private String sex;  
              
               //setter和getter方法省略……  
}
```
* UserServiceImpl ：
如果采用第三种方式，则下面的用法中将方法中的 jdbcTemplate 换成 this.getJdbcTemplate()即可。
Java代码
```java
     /**   
     * 创建表  
     */   
    public void create(String tableName){ //tb_test1  
        jdbcTemplate.execute("create table "+tableName +" (id integer,user_name varchar2(40),password varchar2(40))");  
    }  
      
    //jdbcTemplate.update适合于insert 、update和delete操作；  
    /**   
     * 第一个参数为执行sql   
     * 第二个参数为参数数据   
     */   
    public void save3(User user) {  
        Assert.isNull(user, "user is not null");  
        jdbcTemplate.update("insert into tb_test1(name,password) values(?,?)",   
                new Object[]{user.getUsername(),user.getPassword()});  
    }  
      
    /**   
     * 第一个参数为执行sql   
     * 第二个参数为参数数据   
     * 第三个参数为参数类型   
     */   
    @Override  
    public void save(User user) {  
        Assert.isNull(user, "user is not null");  
        jdbcTemplate.update(  
                "insert into tb_test1(name,password) values(?,?)",   
                new Object[]{user.getUsername(),user.getPassword()},   
                new int[]{java.sql.Types.VARCHAR,java.sql.Types.VARCHAR}  
                );  
    }  
  
    //避免sql注入  
    public void save2(final User user) {  
        Assert.isNull(user, "user is not null");  
          
        jdbcTemplate.update("insert into tb_test1(name,password) values(?,?)",   
                new PreparedStatementSetter(){  
              
                    @Override  
                    public void setValues(PreparedStatement ps) throws SQLException {  
                        ps.setString(1, user.getUsername());  
                        ps.setString(2, user.getPassword());  
                    }  
        });  
          
    }  
      
    public void save4(User user) {  
        Assert.isNull(user, "user is not null");  
        jdbcTemplate.update("insert into tb_test1(name,password) values(?,?)",   
                             new Object[]{user.getUsername(),user.getPassword()});  
    }  
      
    //返回插入的主键  
    public List save5(final User user) {  
          
        KeyHolder keyHolder = new GeneratedKeyHolder();  
  
        jdbcTemplate.update(new PreparedStatementCreator() {  
                      
                                @Override  
                                public PreparedStatement createPreparedStatement(Connection connection) throws SQLException {  
                                    PreparedStatement ps = connection.prepareStatement("insert into tb_test1(name,password) values(?,?)", new String[] {"id"});  
                                    ps.setString(1, user.getUsername());  
                                    ps.setString(2, user.getPassword());  
                                    return ps;  
                                }  
                            },  
                keyHolder);  
          
        return keyHolder.getKeyList();  
    }  
      
    @Override  
    public void update(final User user) {  
        jdbcTemplate.update(  
                "update tb_test1 set name=？,password=？ where id = ?",   
                new PreparedStatementSetter(){  
                    @Override  
                    public void setValues(PreparedStatement ps) throws SQLException {  
                        ps.setString(1, user.getUsername());  
                        ps.setString(2, user.getPassword());  
                        ps.setInt(3, user.getId());  
                    }  
                }  
        );  
    }  
  
    @Override  
    public void delete(User user) {  
        Assert.isNull(user, "user is not null");  
        jdbcTemplate.update(  
                "delete from tb_test1 where id = ?",   
                new Object[]{user.getId()},   
                new int[]{java.sql.Types.INTEGER});  
    }  
  
    @Deprecated //因为没有查询条件，所以用处不大  
    public int queryForInt1(){  
        return jdbcTemplate.queryForInt("select count(0) from tb_test1");  
    }  
      
    public int queryForInt2(User user){  
        return jdbcTemplate.queryForInt("select count(0) from tb_test1 where username = ?" ,  
                new Object[]{user.getUsername()});  
    }  
      
    //最全的参数3个  
    public int queryForInt3(User user){  
        return jdbcTemplate.queryForInt("select count(0) from tb_test1 where username = ?" ,  
                new Object[]{user.getUsername()},  
                new int[]{java.sql.Types.VARCHAR});  
    }  
      
    //可以返回是一个基本类型的值  
    @Deprecated  //因为没有查询条件，所以用处不大  
    public String queryForObject1(User user) {  
        return (String) jdbcTemplate.queryForObject("select username from tb_test1 where id = 100",  
                                                    String.class);  
    }  
      
    //可以返回值是一个对象  
    @Deprecated //因为没有查询条件，所以用处不大  
    public User queryForObject2(User user) {  
        return (User) jdbcTemplate.queryForObject("select * from tb_test1 where id = 100", User.class); //class是结果数据的java类型  
    }  
      
    @Deprecated //因为没有查询条件，所以用处不大  
    public User queryForObject3(User user) {  
        return (User) jdbcTemplate.queryForObject("select * from tb_test1 where id = 100",   
                    new RowMapper(){  
      
                        @Override  
                        public Object mapRow(ResultSet rs, int rowNum)throws SQLException {  
                            User user  = new User();  
                            user.setId(rs.getInt("id"));  
                            user.setUsername(rs.getString("username"));  
                            user.setPassword(rs.getString("password"));  
                            return user;  
                        }  
                    }  
        );   
    }  
      
    public User queryForObject4(User user) {  
        return (User) jdbcTemplate.queryForObject("select * from tb_test1 where id = ?",   
                                                    new Object[]{user.getId()},  
                                                    User.class); //class是结果数据的java类型  实际上这里是做反射，将查询的结果和User进行对应复制  
    }  
      
    public User queryForObject5(User user) {  
        return (User) jdbcTemplate.queryForObject(  
                "select * from tb_test1 where id = ?",   
                new Object[]{user.getId()},  
                new RowMapper(){  
  
                    @Override  
                    public Object mapRow(ResultSet rs,int rowNum)throws SQLException {  
                        User user  = new User();  
                        user.setId(rs.getInt("id"));  
                        user.setUsername(rs.getString("username"));  
                        user.setPassword(rs.getString("password"));  
                        return user;  
                    }  
              
        }); //class是结果数据的java类型  
    }  
      
    @Override  
    public User queryForObject(User user) {  
        //方法有返回值  
        return (User) jdbcTemplate.queryForObject("select * from tb_test1 where id = ?",  
                new Object[]{user.getId()},  
                new int[]{java.sql.Types.INTEGER},   
                new RowMapper() {  
              
                    @Override  
                    public Object mapRow(ResultSet rs, int rowNum) throws SQLException {  
                        User user  = new User();  
                        user.setId(rs.getInt("id"));  
                        user.setUsername(rs.getString("username"));  
                        user.setPassword(rs.getString("password"));  
                        return user;  
                    }  
                }  
        );  
    }  
  
    @SuppressWarnings("unchecked")  
    public List<User> queryForList1(User user) {  
        return (List<User>) jdbcTemplate.queryForList("select * from tb_test1 where username = ?",   
                            new Object[]{user.getUsername()},  
                            User.class);  
    }  
  
    @SuppressWarnings("unchecked")  
    public List<String> queryForList2(User user) {  
        return (List<String>) jdbcTemplate.queryForList("select username from tb_test1 where sex = ?",   
                            new Object[]{user.getSex()},  
                            String.class);  
    }  
      
    @SuppressWarnings("unchecked")  
    //最全的参数查询  
    public List<User> queryForList3(User user) {  
        return (List<User>) jdbcTemplate.queryForList("select * from tb_test1 where username = ?",  
                            new Object[]{user.getUsername()},  
                            new int[]{java.sql.Types.VARCHAR},  
                            User.class);  
    }  
  
    //通过RowCallbackHandler对Select语句得到的每行记录进行解析，并为其创建一个User数据对象。实现了手动的OR映射。  
    public User queryUserById4(String id){  
        final User user  = new User();  
          
        //该方法返回值为void  
        this.jdbcTemplate.query("select * from tb_test1 where id = ?",   
                new Object[] { id },   
                new RowCallbackHandler() {     
              
                    @Override    
                    public void processRow(ResultSet rs) throws SQLException {     
                        User user  = new User();  
            user.setId(rs.getInt("id"));  
            user.setUsername(rs.getString("username"));  
            user.setPassword(rs.getString("password"));    
                    }     
        });   
          
        return user;     
    }  
      
    @SuppressWarnings("unchecked")  
    @Override  
    public List<User> list(User user) {  
        return jdbcTemplate.query("select * from tb_test1 where username like '%?%'",   
                new Object[]{user.getUsername()},   
                new int[]{java.sql.Types.VARCHAR},   
                new RowMapper(){  
              
                    @Override  
                    public Object mapRow(ResultSet rs, int rowNum) throws SQLException {  
                        User user  = new User();  
                        user.setId(rs.getInt("id"));  
                        user.setUsername(rs.getString("username"));  
                        user.setPassword(rs.getString("password"));  
                        return user;  
                    }  
        });  
    }  
  
    //批量操作    适合于增、删、改操作  
    public int[] batchUpdate(final List users) {  
          
        int[] updateCounts = jdbcTemplate.batchUpdate(  
                "update tb_test1 set username = ?, password = ? where id = ?",  
                new BatchPreparedStatementSetter() {  
                      
                        @Override  
                        public void setValues(PreparedStatement ps, int i) throws SQLException {  
                            ps.setString(1, ((User)users.get(i)).getUsername());  
                            ps.setString(2, ((User)users.get(i)).getPassword());  
                            ps.setLong(3, ((User)users.get(i)).getId());  
                        }  
                          
                        @Override  
                        public int getBatchSize() {  
                            return users.size();  
                        }  
                }   
        );  
          
        return updateCounts;  
    }  
      
    //调用存储过程  
    public void callProcedure(int id){  
        this.jdbcTemplate.update("call SUPPORT.REFRESH_USERS_SUMMARY(?)", new Object[]{Long.valueOf(id)});  
}
```
> 其中，batchUpdate适合于批量增、删、改操作；
         update(…)：使用于增、删、改操作；
         execute（）：执行一个独立的sql语句，包括ddl语句；
         queryForInt ：查询出一个整数值
 
 
摘自：http://lehsyh.iteye.com/blog/1579737