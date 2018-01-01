# Java Web开发APPENDIX I WEB.XML

## 前言

`web.xml`是一个`deployment descriptor`，即它告诉Servlet容器该怎么部署你的Web App。在Servlet 3.0引入annotation-based部署方式之后，这个文件不是必须的，但在实际使用中它作为一个declarative部署方式仍有很大作用。

## DTD

### context-param\init-param

`context-param`这个标签可以定义context的初始参数。

```xml
<context-param>
	<param-name>foo</param-name>
  	<param-value>bar</param-value>
</context-param>
```

在servlet中这样获取:

```Java
req.getServletContext().getInitParameter("foo")
```

`init-param`只能在`<servlet>`内定义:

```xml
<servlet>
    <servlet-name>IndexController</servlet-name>
    <servlet-class>com.ray.servlet.IndexController</servlet-class>
    <init-param>
        <param-name>foo</param-name>
        <param-value>bar</param-value>
    </init-param>
</servlet>
```

在servlet中这样获取：

```Java
getInitParameter("foo")
```

两者区别很明显，一个是整个Web-app-level的初始常量，可以在context初始化后在context内获取。一个是servlet-level的。

### env-entry

`<env-entry>`可以用于定义环境变量，在Java中可以通过JNDI获取：

```Java
InitialContext cxt = new InitialContext();
String ds = (String) cxt.lookup("java:/comp/env/your-env-entry");
```

```xml
<env-entry> 
    <env-entry-name>your-env-entry</env-entry-name> 
    <env-entry-type>java.lang.String</env-entry-type> 
    <env-entry-value>your-env-value</env-entry-value> 
</env-entry>
```

`web.xml`中定义的环境变量只能是以下的type:

```
java.lang.Boolean
java.lang.Byte
java.lang.Character
java.lang.String
java.lang.Short
java.lang.Integer
java.lang.Long
java.lang.Float
java.lang.Double
```

### error-code/error-page/exception-type

```xml
<!ELEMENT error-page ((error-code | exception-type), location)>
```

`<error-page>`可以定义错误或异常时转向的页面。

### mime-mapping/mime-type

```xml
<!ELEMENT mime-mapping (extension, mime-type)>
```

指定不同的后缀名应该用什么MIME-TYPE。

`mime-type`是用于客户端鉴别怎么处理服务器返回的数据的标志。

| 类型            | 描述                                     | 典型示例                                     |
| ------------- | -------------------------------------- | ---------------------------------------- |
| `text`        | 表明文件是普通文本，理论上是可读的语言                    | `text/plain`, `text/html`, `text/css` ,`text/javascript` |
| `image`       | 表明是某种图像。不包括视频，但是动态图（比如动态gif）也使用image类型 | `image/gif`, `image/png`, `image/jpeg`, `image/bmp`, `image/webp` |
| `audio`       | 表明是某种音频文件                              | `audio/midi`, `audio/mpeg, audio/webm, audio/ogg, audio/wav` |
| `video`       | 表明是某种视频文件                              | `video/webm`, `video/ogg`                |
| `application` | 表明是某种二进制数据                             | `application/octet-stream`, `application/pkcs12`, `application/vnd.mspowerpoint`, `application/xhtml+xml`, `application/xml`,  `application/pdf`,`application/json` |

From [mozilla](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Basics_of_HTTP/MIME_types)

服务器返回的数据可以没有什么后缀名，这时指定`mime-type`可以让浏览器以正确的方式展示数据（或者下载数据）。不过如果展示静态内容，使用其他服务器软件会更好，比如`nginx`。

