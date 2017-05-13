title: "JSP/Servlet基础知识汇总"
date: 2015-06-17 18:25:37
tags: [JSP]
categories: Programming Notes

---

***力求用最简洁的文字表述最全面的知识，本Blog不适合零基础人员***

###JSP与Servlet
所有的JSP页面最终都会被编译成Servlet执行，而在Servlet类中主要有三个方法，分别是
* init(): 初始化JSP/Servlet的方法
* destroy(): 销毁JSP/Servlet的方法
* service(): 对用户请求生成响应的方法

JSP页面必须放到应用服务器中运行，当第一次访问JSP页面时，该JSP页面会被编译成Servlet，如果JSP没有改动的话，以后访问的都是第一次编译成功的Servlet。

<!-- more -->

###JSP的4种基本语法
####JSP注释
`<%--JSP注释--%>` 这种注释在客户端浏览器中使用查看源代码是无法查看的
`<!--HTML注释-->` 这种注释在客户端浏览器中使用查看源代码可以进行查看
####JSP声明、输出、脚本
`<%! 声明部分 %>` 相当于全局变量或方法，可使用private、public、static等
`<%=count++%>` 输出表达式语法后不能有分号
`<% scriptlet %>` JSP脚本会转换成_jspService方法里的可执行代码，而Java语法不允许在方法里定义方法，所以JSP脚本里不能定义方法；不能使用private、public、static等

###JSP的3个编译指令
* page: 该指令是针对当前页面的指令
* include: 用于指定包含另一个页面
* taglib: 用于定义和访问自定义标签

其中`<%@ include file="scriptlet.jsp" %>`是静态包含指令，该指令包含进来的页面不需要是一个完整的页面，同时会将被包含页面的编译指令包含进来，如果两个页面的编译指令冲突，那么页面就会报错

动态包含如下所示
```jsp
<jsp:include page="{scriptlet.jsp|<%=expressi%>}">
    {<jsp:param name="parameterName" value="parameterValue">}
</jsp:include>
```
该动态指令仅仅会将被包含页面的body内容插入本页面，这时候被包含页面的编译指令不会被插入本页面

###JSP的7个动作指令
* jsp:forward: 执行页面转向，将请求的处理转发到下一个页面，属于服务器端跳转，地址栏不变，执行forward时不会丢失请求参数
* jsp:param: 用于传递参数，必须与其他支持参数的标签一起使用，主要是结合include、forward、plugin指令使用
* jsp:include: 用于动态引入一个JSP页面
* jsp:plugin: 用于下载JavaBean或Applet到客户端执行
* jsp:useBean: 创建一个JavaBean的实例
* jsp:setProperty: 设置JavaBean实例的属性值
* jsp:getProperty: 输出JavaBean实例的属性值

useBean的用法
`<jsp:useBean id="name" class="classname" scope="page|request|session|application"/>`

- page: 该JavaBean实例仅在该页面有效
- request: 该JavaBean实例在本次请求有效
- session: 该JavaBean实例在本次session内有效
- application: 该JavaBean实例在本次应用内一直有效

setProperty语法
`<jsp:setProperty name="BeanName" property="propertyName" value="value">`

getProperty语法
`<jsp:getProperty name="BeanName" property="propertyName">`

其实，当使用setProperty和getProperty时，底层是调用setBeanName()和getBeanName()方法来操作实例的属性的
###JSP的9大内置对象
* application: javax.servlet.ServletContext的实例，该实例代表JSP所属的Web应用本身，可用于JSP页面，或者在Servlet之间交换信息
* config: javax.servlet.ServletConfig的实例，该实例代表JSP的配置信息，该对象使用较少
* exception: java.lang.Throwable的实例，该实例代表其他页面中的异常和错误。只有当页面编译指令page的isErrorPage属性为true时，该对象才可以使用
* out: javax.servlet.jsp.JspWriter的实例，该实例代表JSP页面的输出流，用于输出内容，形成HTML页面
* page: 代表该页面本身，通常没有太大用处。也就是Servlet中的this，其类型就是生成的Servlet类，能用page的地方就能用this
* pageContext: javax.servlet.jsp.PageContext的实例，该对象代表该JSP页面上下文，使用该对象可以访问页面中的共享数据
* request: javax.servlet.http.HttpServletRequest的实例，该对象封装了一次请求，客户端的请求参数都被封装在该对象里
* response: javax.servlet.http.HttpServletResponse的实例，代表服务器对客户端的响应，通常很少使用该对象直接响应，而是使用out对象，除非需要生成非字符响应
* session: javax.servlet.http.HttpSession的实例，该对象代表一次会话。当客户端浏览器与站点建立连接时，会话开始；当客户端关闭浏览器时，会话结束

***Note***
1. 由于JSP内置对象都是在_jspService()方法中完成初始化的，因此只能在JSP脚本、JSP输出表达式中使用这些内置对象。千万不要在JSP声明中使用它们！否则，会提示找不到变量。
2. 所有使用out的地方，都可使用输出表达式来代替，而且使用输出表达式更加简洁，`<%=...%>`表达式的本质就是`out.write();`。
3. pageContext可以访问page/request/session/application范围的变量，方法是取得指定范围的name属性，并指定其scope，如下所示
```java
pageContext.getAttribute(String name,[PageContext.PAGE_SCOPE|PageContext.REQUEST_SCOPE|PageContext.SESSION_SCOPE|PageContext.APPLICATION_SCOPE])
```
4. request有两个方法分别是forward和include
```java
request.getRequestDispatcher("/a.jsp").include(request,response)代替<jsp:include>指令

request.getRequestDispatcher("/a.jsp").forward(request,response)代替<jsp:forward>指令
```
5. 不要滥用session，通常只应该把与用户会话状态相关的信息放入session范围内，不要仅仅为了两个页面之间交换信息，就将该信息放入session范围内。如果仅仅为了两个页面交换信息，建议使用request，然后forward请求即可。session机制通常用于保存客户端的状态信息，这些状态信息需要保存到Web服务器的硬盘上，所以要求session里的属性值必须是可序列化的，否则将引发不可序列化的异常。

###Filter
创建Filter必须实现javax.servlet.Filter接口，在该接口中定义了如下三个方法
* void init(FilterConfig config): 用于完成Filter的初始化
* void destroy(): 用于Filter销毁前，完成某些资源的回收
* void doFilter(ServletRequest request, ServletResponse response, FilterChain chain): 实现过滤功能，该方法就是对每个请求及响应增加的额外处理

Filter其实就是增强的Servlet，假设系统包含多个Servlet，这些Servlet都需要进行一些通用处理：比如权限控制、记录日志等，这将导致在这些Servlet的service方法中有部分代码是相同的——为了解决这种代码重复的问题，可以考虑把这些通用处理提取到Filter中完成，这样各Servlet中剩下的只是特定请求相关的处理代码，而通用处理则交给Filter完成。

###使用URL Rewrite实现网站伪静态
下载[地址](http://www.tuckey.org/urlrewrite/)，目前最新版是urlrewritefilter-4.0.3.jar，在下载页面提供了使用方法，不再赘述。

###Listener
常用的Web事件监听器接口有如下几个
* ServletContextListener: 用于监听Web应用的启动和关闭
* ServletContextAttributeListener: 用于监听ServletContext范围（application）内属性的改变
* ServletRequestListener: 用于监听用户请求
* ServletRequestAttributeListener: 用于监听ServletRequest范围（request）内属性的改变
* HttpSessionListener: 用于监听用户session的开始和结束
* HttpSessionAttributeListener: 用于监听HttpSession范围（session）内属性的改变

###JSP2特性
目前Servlet3.1对应于JSP2.3规范，JSP2.3也被统称为JSP2。表达式语言是JSP2的一个重要特性，它并不是一种通用的程序语言，而仅仅是一种数据访问语言，可以方便地访问应用程序数据，避免使用JSP脚本。表达式语言的格式是`${expression}`，如果需要在支持表达式语言的页面中正常输出“\$”符号，则在“\$”符号前加转义字符“\”。
####表达式语言的内置对象
表达式语言包含如下11个内置对象
* pageContext: 代表该页面的pageContext对象，与JSP的pageContext内置对象相同
* pageScope: 用于获取page范围的属性值
* requestScope: 用于获取request范围的属性值
* sessionScope: 用于获取session范围的属性值
* applicationScope: 用于获取application范围的属性值
* param: 用于获取请求的参数值
* paramValues: 用于获取请求的参数值，与param的区别在于，该对象用于获取属性值为数组的属性值
* header: 用于获取请求头的属性值
* headerValues: 用于获取请求头的属性值，与header的区别在于，该对象用于获取属性值为数组的属性值
* initParam: 用于获取请求Web应用的初始化参数
* cookie: 用于获取指定的Cookie值

###Servlet3.0新特性
Servlet3.0规范在javax.servlet.annotation包下提供了如下注解
* @WebServlet: 用于修饰一个Servlet类，用于部署Servlet类
* @WebInitParam: 用于与@WebServlet或@WebFilter一起使用，为Servlet、Filter配置参数
* @WebListener: 用于修饰Listener类，用于部署Listener类
* @WebFilter: 用于修饰Filter类，用于部署Filter类
* @MultipartConfig: 用于修饰Servlet，指定该Servlet将会负责处理multipart/form-data类型的请求
* @ServiceSecurity: 这是一个与JAAS有关的注解，修饰Servlet指定该Servlet的安全与授权控制
* @HttpConstraint: 用于与@ServletSecurity一起使用，用于指定该Servlet的安全与授权控制
* @HttpMethodConstraint: 用于与@ServletSecurity一起使用，用于指定该Servlet的安全与授权控制


参考资料
【1】轻量级JavaEE企业应用实战-Struts2+Spring4+Hibernate整合开发