# Java Web开发三：JDBC和DataSource

## 前言

传统的Web App可以说都是database-driven模式的，增删改查是整个App的主要逻辑，所以我们需要有合适的方法连接数据库。

## JDBC

### 依赖

```
compile group: 'mysql', name: 'mysql-connector-java', version: '6.0.6'
```

### 连接

```Java
Class.forName("com.mysql.cj.jdbc.Driver"); //JDBC V4开始不需要这个了
Connection conn = DriverManager.getConnection("jdbc:mysql://localhost:3306/test", "root", "");
Statement statement = conn.createStatement();
statement.execute("INSERT INTO test VALUES (1, 'test')");
conn.commit(); // auto-commit下不需要
ResultSet resultSet = statement.executeQuery("SELECT * FROM test");
while (resultSet.next()) {
    System.out.println(resultSet.getInt(1) + resultSet.getString(2));
}
conn.close();
```

连接很简单，但我们不需要每次用到数据库时都去创建一个连接，基于TCP的连接消耗是很大的，如果有数千请求不断创建连接、断开连接，其消耗在连接上的CPU时间会变得不能接受。所以我们需要连接池。

## DataSource

首先，在`server.xml`或`context.xml`中添加资源：

```XML
<Resource name="jdbc/TestDB" auth="Container" type="javax.sql.DataSource"
          maxTotal="100" maxIdle="30" maxWaitMillis="10000"
          username="root" password="" driverClassName="com.mysql.cj.jdbc.Driver"
          url="jdbc:mysql://localhost:3306/test?characterEncoding=UTF-8&amp;serverTimezone=UTC"/>
```

这里需要注意的一点是`&`在XML中是需要转义成`&amp;`的。

然后在`web.xml`中添加资源引用

```XML
<resource-ref>
    <res-ref-name>jdbc/TestDB</res-ref-name>
    <res-type>javax.sql.DataSource</res-type>
    <res-auth>Container</res-auth>
</resource-ref>
```

最后在servlet中请求资源：

```Java
try {
    InitialContext cxt = new InitialContext();
    DataSource ds = (DataSource) cxt.lookup( "java:/comp/env/jdbc/TestDB" );
    Connection connection = ds.getConnection();
    Statement statement = connection.createStatement();
    statement.execute("INSERT INTO java_test VALUES (1, 'abcd')");
    ResultSet resultSet = statement.executeQuery("SELECT * FROM java_test");
    while (resultSet.next()) {
        System.out.println(resultSet.getInt(1) + " " + resultSet.getString(2));
    }
} catch (Exception e) {
    e.printStackTrace();
}
```

这里`java:/comp/env`的前缀是必须的，后面跟自己设定的资源名称。