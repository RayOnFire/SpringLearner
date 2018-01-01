# Java Web开发一：Servlet，Listener和Filter

## 前言

Java Web开发应该从Servlet开始，Servlet是作为MVC开发模型的C——Controller存在的，它的作用很直观，处理不同HTTP Method，比如GET\POST等等，它通过`doGet`，`doPost`等方法接受一个HTTP请求对象和一个HTTP响应对象，可以从请求中读取信息，然后修改响应对象再返回给客户端。

## Filter

###计数器

这里以一个计数器介绍Filter的作用：

```Java
@WebFilter(filterName = "HitCounterFilter", servletNames = "WelcomeController")
public class HitCounterFilter implements Filter {

    private FilterConfig filterConfig;

    @Override
    public void init(FilterConfig filterConfig) throws ServletException {
        this.filterConfig = filterConfig;
    }

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
        if (filterConfig == null)
            return;
        // 获取counter，使用AtomicInteger保证线程安全
        AtomicInteger counter = (AtomicInteger) filterConfig.
                getServletContext().
                getAttribute("hitCounter");
      	// 如果没有counter在context内，创建一个放入context内，保存在context的attribute不会
        // 随着request销毁而消失
        if (counter == null) {
            counter = new AtomicInteger(0);
          filterConfig.getServletContext().setAttribute("hitCounter", counter);
        }
        // 在console打印log
        StringWriter sw = new StringWriter();
        PrintWriter writer = new PrintWriter(sw);
        writer.println();
        writer.println("===============");
        writer.println("The number of hits is: " +
                counter.incrementAndGet());
        writer.println("===============");
        writer.flush();
        filterConfig.getServletContext().
                log(sw.getBuffer().toString());
        // 继续Filter chain，如果不要下面这行，可以拦截请求不被servlet处理。
        chain.doFilter(request, response);

        // 这里servlet已经处理过request和response对象，可以修改response
        PrintWriter out = response.getWriter();
        CharArrayWriter caw = new CharArrayWriter();
        caw.write("You are visitor number " + counter.get());
        out.write(caw.toString());
    }

    @Override
    public void destroy() {
        this.filterConfig = null;
    }
}
```


###Filter Order

使用annotation-based的filter无法规定顺序，如果有多个filter应用在同一servlet或URL pattern，应该修改`web.xml`配置：

```xml
<web-app>
    <filter-mapping>
        <filter-name>FilterA</filter-name>
        <url-pattern/>
    </filter-mapping>
    <filter-mapping>
        <filter-name>FilterB</filter-name>
        <url-pattern/>
    </filter-mapping>
</web-app>
```
from [StackOverflow](https://stackoverflow.com/questions/6560969/how-to-define-servlet-filter-order-of-execution-using-annotations-in-war)
###Filter中获得response content

在Filter中怎么获得servlet处理后的response内容呢？这个问题稍显麻烦，我们应该给`chain.doFilter`一个`HttpServletResponseWrapper`，这个wrapper还需要我们做一些改造才能使用。

```Java
public class HttpServletResponseCopier extends HttpServletResponseWrapper {

    private ServletOutputStream outputStream;
    private PrintWriter writer;
    private ServletOutputStreamCopier copier;

    public HttpServletResponseCopier(HttpServletResponse response) throws IOException {
        super(response);
    }

    @Override
    public ServletOutputStream getOutputStream() throws IOException {
        if (writer != null) {
            throw new IllegalStateException("getWriter() has already been called on this response.");
        }
        if (outputStream == null) {
            outputStream = getResponse().getOutputStream();
            copier = new ServletOutputStreamCopier(outputStream);
        }
        return copier;
    }

    @Override
    public PrintWriter getWriter() throws IOException {
        if (outputStream != null) {
            throw new IllegalStateException("getOutputStream() has already been called on this response.");
        }
        if (writer == null) {
            copier = new ServletOutputStreamCopier(getResponse().getOutputStream());
            writer = new PrintWriter(new OutputStreamWriter(copier, getResponse().getCharacterEncoding()), true);
        }
        return writer;
    }

    @Override
    public void flushBuffer() throws IOException {
        if (writer != null) {
            writer.flush();
        } else if (outputStream != null) {
            copier.flush();
        }
    }

    public byte[] getCopy() {
        if (copier != null) {
            return copier.getCopy();
        } else {
            return new byte[0];
        }
    }
}
```

其中要用到一个我们自己extend的`ServletOutputStream`:

```Java
public class ServletOutputStreamCopier extends ServletOutputStream {

    private OutputStream outputStream;
    private ByteArrayOutputStream copy;

    public ServletOutputStreamCopier(OutputStream outputStream) {
        this.outputStream = outputStream;
        this.copy = new ByteArrayOutputStream(1024);
    }

    @Override
    public void write(int b) throws IOException {
        outputStream.write(b);
        copy.write(b);
    }

    public byte[] getCopy() {
        return copy.toByteArray();
    }

}
```

使用：

```Java
HttpServletResponseCopier responseCopier = new HttpServletResponseCopier((HttpServletResponse) response);

try {
    chain.doFilter(request, responseCopier);
    responseCopier.flushBuffer();
} finally {
    byte[] copy = responseCopier.getCopy();
    System.out.println(new String(copy, response.getCharacterEncoding()));
}
```

from [StackOverflow](https://stackoverflow.com/questions/8933054/how-to-read-and-copy-the-http-servlet-response-output-stream-content-for-logging)
