# Java Web开发四：Servlet实现用户系统

## 前言

用户系统是可以整合数据库操作和servlet展示的小项目。

为了贯彻模块化开发的思想，这个项目只用作用户系统实现，挂载在`/auth`下，可以运行在单独的`Tomcat`容器下，也可以与其他app运行在同一个`Tomcat`下。

## 创建数据库

### 创建用户

```sql
create user 'servletuser'@'localhost' identified by 'pwd';
grant all privileges on * . * to 'servletuser'@'localhost';
flush privileges;
```

以上SQL创建用户并给与它全部权限。~~从删库到跑路？~~

## 创建SQL语句

为了SQL语句方便管理和review，我们将其作为一个`properties`文件放至`src/main/resources`文件夹内：

```properties
# sql.properties
getUserById=SELECT id, password FROM user WHERE username = ?
getRolesByUserId=SELECT role_name FROM role r WHERE r.user_id = ? LEFT JOIN user u ON r.user_id = u.id
```

获取这个文件可以用以下方法：

```Java
File sqlFile = new File(getClass().getClassLoader().getResource("sql.properties").getFile());
Properties properties = new Properties();
properties.load(new FileInputStream(sqlFile));
```

## DAO

对于`User`，我们不希望每次都去请求数据库拿数据，然后再创建一个`User`对象，我们可以用一个`ConcurrentHashMap`来存储曾经获取过的对象。

### 同步问题

既然用到了`ConcurrentHashMap`，我们要保证其并行性，需要用到它的原子操作。

这样做是不行的：

```Java
if (map.containsKey(username)) {
    return map.get(username);
} else {
  // get User from database
}
```

在`containsKey`操作结束后有可能别的线程会删除这个key，导致下一步的`get`返回null。

应该这样：

```Java
User savedUser = map.getOrDefault(username, null);
if (savedUser != null) {
    return savedUser;
} else {
  // get User from database;
}
```

同时在插入map的时候应该用：

```Java
map.putIfAbsent(username, uncachedUser);
```

User类如下：

```Java
public class User {

    private static ConcurrentHashMap<String, User> map = new ConcurrentHashMap<>();
    private static Properties sqlStatements = new Properties();
    private static InitialContext context;
    private static DataSource dataSource;

    private Long id;
    private String username;
    private String password;

    private List<Role> roles;

    public User(Long id, String username, String password) {
        this.id = id;
        this.username = username;
        this.password = password;
    }
  
	// GETTERS AND SETTERS

    public static void setSQL(Properties properties) {
        sqlStatements = properties;
    }

    public static Optional<User> get(String username) {
        Optional<User> userOptional = Optional.empty();
        if (context == null) {
            try {
                context = new InitialContext();
                dataSource = (DataSource) context.lookup("java:/comp/env/jdbc/TestDB");
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
        if (sqlStatements != null) {
            User savedUser = map.getOrDefault(username, null);
            if (savedUser != null) {
                userOptional = Optional.of(savedUser);
            } else {
                try {
                    Connection connection = dataSource.getConnection();
                    PreparedStatement statement = connection.prepareStatement(sqlStatements.getProperty("getUserById"));
                    statement.setString(1, username);
                    ResultSet resultSet = statement.executeQuery();
                    if (resultSet.next()) {
                        Long userId = resultSet.getLong(1);
                        String password = resultSet.getString(2);
                        User unsavedUser = new User(userId, username, password);
                        statement = connection.prepareStatement(sqlStatements.getProperty("getRolesByUserId"));
                        statement.setLong(1, userId);
                        resultSet = statement.executeQuery();
                        List<Role> roles = new ArrayList<>();
                        String roleName;
                        while (resultSet.next()) {
                            roleName = resultSet.getString(1);
                            roles.add(Role.valueOf(roleName));
                        }
                        unsavedUser.setRoles(roles);
                        map.putIfAbsent(username, unsavedUser);
                        userOptional = Optional.of(unsavedUser);
                    }
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        }
        return userOptional;
    }
}
```

值得注意的是`get`方法返回一个`Optional`对象，是Java 8新加入的类，返回`Optional`可以提醒调用者注意检查`null`值，预防Java中最常见的错误——`NullPointerException`。

## 错误处理

| No.  | Attribute & Description                  |
| ---- | ---------------------------------------- |
| 1    | `javax.servlet.error.status_code` This attribute give status code which can be stored and analyzed after storing in a `java.lang.Integer` data type. |
| 2    | `javax.servlet.error.exception_type` This attribute gives information about exception type which can be stored and analyzed after storing in a `java.lang.Class` data type. |
| 3    | `javax.servlet.error.message` This attribute gives information exact error message which can be stored and analyzed after storing in a `java.lang.String` data type. |
| 4    | `javax.servlet.error.request_uri` This attribute gives information about URL calling the servlet and it can be stored and analyzed after storing in a `java.lang.String` data type. |
| 5    | `javax.servlet.error.exception` This attribute gives information about the exception raised, which can be stored and analyzed. |
| 6    | `javax.servlet.error.servlet_name`This attribute gives servlet name which can be stored and analyzed after storing in a `java.lang.String` data type. |

from [tutorialpoint](https://www.tutorialspoint.com/servlets/servlets-exception-handling.htm)

以上是存储在`HttpServletRequest`内的属性，下面在servlet中获取他们：

```xml
<!--web.xml-->
<error-page>
    <location>/error</location>
</error-page>
```

```Java
// ErrorController.java
@WebServlet(name = "ErrorController", urlPatterns = {"/error"})
public class ErrorController extends HttpServlet {
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        Integer statusCode = (Integer) req.getAttribute("javax.servlet.error.status_code");
        String message = (String) req.getAttribute("javax.servlet.error.message");
        PrintWriter writer = resp.getWriter();
        writer.println(statusCode);
        writer.println(message);
    }

    @Override
    protected void doPost(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        doGet(req, resp);
    }
}
```

这里如果你用`sendError`转入错误处理`javax.servlet.error.servlet_name`和`javax.servlet.error.request_uri`会是空字符串，如果需要获得servlet name，要用异常跳出servlet.

## 日志

```Java
@WebFilter(filterName = "ErrorControllerFilter", servletNames = {"ErrorController"})
public class ErrorControllerFilter implements Filter {
    private Logger logger = LoggerFactory.getLogger(getClass());
    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
        HttpServletRequest req = (HttpServletRequest) servletRequest;
        Integer statusCode = (Integer) req.getAttribute("javax.servlet.error.status_code");
        String servletName = (String) req.getAttribute("javax.servlet.error.servlet_name");
        Exception exception = (Exception) req.getAttribute("javax.servlet.error.exception");
        String requestUri = (String) req.getAttribute("javax.servlet.error.request_uri");
        logger.info(requestUri + "-" + servletName + "-" + statusCode + "-" + exception);
        filterChain.doFilter(servletRequest, servletResponse);
    }
}
```

对`<error-page>`应用的servlet启用一个filter作为日志。这里需要注意，filter默认是不会处理错误的转发的。

需要设置：

```xml
<!--web.xml-->
<filter-mapping>
    <filter-name>ErrorControllerFilter</filter-name>
    <servlet-name>ErrorController</servlet-name>
    <dispatcher>ERROR</dispatcher>
</filter-mapping>
```

或者：

```Java
@WebFilter(..., dispatcherTypes = {DispatcherType.ERROR})
```

默认的dispatcher是`REQUEST`，如果需要同时处理两者，则需要把`REQUEST`也写上。

其他dispatcher还有一个`FORWARD`。

from [StackOverflow](https://stackoverflow.com/questions/28027249/web-xml-error-page-not-filtered)

在这一步我们需要改造一下`LoginController`，让其更好配合错误处理：

```Java
@WebServlet(name = "LoginController", urlPatterns = {"/login"})
public class LoginController extends HttpServlet {
    @Override
    protected void doPost(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        String username = req.getParameter("username");
        String password = req.getParameter("password");
        PrintWriter writer = resp.getWriter();
        Object isAuthenticated = req.getSession().getAttribute("authenticated");
        if (isAuthenticated != null && (Boolean)isAuthenticated) {
            resp.setStatus(200);
            writer.println("Already Authenticated!");
        } else {
            if (username == null || password == null) {
                throw new ParameterNotPresentException("Necessary Parameter Not Presented");
            } else {
                if (AuthenticationService.authenticate(req.getServletContext(), username, password)) {
                    req.getSession().setAttribute("authenticated", true);
                    resp.setStatus(200);
                    writer.println("Authenticated!");
                } else {
                    throw new BadCredentialException("Bad Credential");
                }
            }
        }
    }
}
```

这里新加入两个异常`ParameterNotPresentException`和`BadCredentialException`，让其给特定的servlet处理，这里为了简单，都给`ErrorController`处理。