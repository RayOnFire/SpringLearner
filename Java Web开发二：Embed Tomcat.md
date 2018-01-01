# Java Web开发二：Embed Tomcat

真的是想不开的人才会用Embed Tomcat吧。

以下记录我曾经用Embed Tomcat带来的问题。

## 前言

想到用Embed Tomcat是因为Spring Boot也是内置一个Embed Tomcat的，有了它，web app可以直接打包为jar，部署应该是最方便的了：传jar到服务器，运行它就是一个web app了。

## 依赖

```groovy
compile group: 'org.apache.tomcat.embed', name: 'tomcat-embed-core', version: '8.5.24'
compile group: 'org.apache.tomcat', name: 'tomcat-jasper', version: '8.5.24'
compile group: 'org.apache.tomcat', name: 'tomcat-dbcp', version: '8.5.24'
```

第一个是必须的，第二个是JSP用的，第三个是Database Connection Pool的支持

## Bootstrap

作为一个普通的应用，我们需要一个入口：

```Java
// Main.java
public class Main {
    public static void main(String[] args) throws Exception {
        // 找到Main.java运行的位置
	    File root;
        try {
            String runningJarPath = Main.class.getProtectionDomain().getCodeSource().getLocation().toURI().getPath().replaceAll("\\\\", "/");
            // 这里IDEA Intellij会将编译后的class文件放至/project/build文件夹下
            int lastIndexOf = runningJarPath.lastIndexOf("/build/");
            if (lastIndexOf < 0) {
                root = new File("");
            } else {
                root = new File(runningJarPath.substring(0, lastIndexOf));
            }
            // application resolved root folder: C:\Users\Ray\Documents\project
            System.out.println("application resolved root folder: " + root.getAbsolutePath());
        } catch (URISyntaxException ex) {
            throw new RuntimeException(ex);
        }
        Tomcat tomcat = new Tomcat();

        Path tempPath = Files.createTempDirectory("tomcat-base-dir");
        tomcat.setBaseDir(tempPath.toString());

        tomcat.setPort(8080);
        // 这里的路径是标准的Java Web Application Structure
        File webContentFolder = new File(root.getAbsolutePath(), "src/main/webapp/");
        if (!webContentFolder.exists()) {
            webContentFolder = Files.createTempDirectory("default-doc-base").toFile();
        }
        // 将web app添加至tomcat，第一个参数是挂载路径，空字符串指根目录（http://localhost:8080/)
        StandardContext ctx = (StandardContext) tomcat.addWebapp("", webContentFolder.getAbsolutePath());

        ctx.setParentClassLoader(Main.class.getClassLoader());

        // configuring app with basedir: C:\Users\Ray\Documents\project\src\main\webapp
        System.out.println("configuring app with basedir: " + webContentFolder.getAbsolutePath());

        // 以下代码是使Servlet 3.0提供的annotation启用的代码
        // 声明了"WEB-INF/classes"的其他位置，这里是"build/classes"(IDEA默认设置)
        File additionWebInfClassesFolder = new File(root.getAbsolutePath(), "build/classes");
        WebResourceRoot resources = new StandardRoot(ctx);

        WebResourceSet resourceSet;
        if (additionWebInfClassesFolder.exists()) {
            resourceSet = new DirResourceSet(resources, "/WEB-INF/classes", additionWebInfClassesFolder.getAbsolutePath(), "/");
            System.out.println("loading WEB-INF resources from as '" + additionWebInfClassesFolder.getAbsolutePath() + "'");
        } else {
            resourceSet = new EmptyResourceSet(resources);
        }
        resources.addPreResources(resourceSet);
        ctx.setResources(resources);
      
        // 启动tomcat
        tomcat.start();
        tomcat.getServer().await();
    }
}
```

到这里还好，只是google了一下怎么启动servlet的annotation。

## JNDI设置

JNDI设置比起deploy版本的Tomcat比较繁琐。

首先，我们要启用JNDI，默认是不启用的：

```Java
tomcat.enableNaming();
```

然后，因为没有server scope的`context.xml`，我们可以自己添加一个`context.xml`，然后需要手动加载这个配置：

```Java
// ctx是上文提到的tomcat.addWebapp返回的StandardContext
ctx.setConfigFile(new File(webContentFolder + "/WEB-INF/context.xml").toURI().toURL());
```

最后，就可以在自己创建的`context.html`添加resources了。

P.S. 既然我们都有`StandardContext`了，我们也可以抛弃XML，手动添加resource:

```Java
ContextResource resource = new ContextResource();
resource.setName("jdbc/datasource");
resource.setType(DataSource.class.getName());
resource.setProperty("driverClassName", "com.mysql.cj.jdbc.Driver");
resource.setProperty("url", "jdbc:mysql://localhost:3306/servlet?characterEncoding=UTF-8&serverTimezone=UTC");
resource.setProperty("username", "root");
resource.setProperty("password", "");
ctx.getNamingResources().addResource(resource);
```

麻烦的是property多的时候要一个一个设置。