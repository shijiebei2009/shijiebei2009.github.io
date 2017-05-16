title: "Struts2基础知识汇总"
date: 2015-06-18 15:41:01
tags: [Struts2]
categories: Programming Notes

---

***力求用最简洁的文字表述最全面的知识，本Blog不适合零基础人员***
![logo][1]
###Struts2简介
Struts2是由传统的Struts1、WebWork两个经典的MVC框架发展起来，如下图所示，无论从Struts2设计的角度还是在实际项目中的易用性来看，Struts2都是一个非常优秀的MVC框架，当然目前还有另外一个非常优秀的MVC框架——SpringMVC，以后再对它进行介绍。
![Struts2][2]

###实现Action
Struts2的Action类是一个普通的POJO（通常应该包含一个无参数的execute方法），Struts2直接使用Action来封装HTTP请求参数，因此，Action类里还应该包含与请求参数对应的实例变量，并且为这些实例变量提供对应的setter和getter方法。注意其实实例变量是可以省略的，因为Struts2是通过对应的setter和getter方法来处理请求参数的，而不是通过实例变量名来处理请求参数的。
####Action访问Servlet API
```java
ActionContext ctx = ActionContext.getContext();//相当于JSP中内置的request
ctx.getApplication();//返回Map对象，该对象模拟了ServletContext实例，相当于JSP中内置的application
ctx.getSession();//返回Map对象，该对象模拟了HttpSession实例，相当于JSP中内置的session
```
####Action直接访问Servlet API

Struts2提供了几个接口供我们直接访问ServletAPI
* `ServletContextAware`: 实现该接口的Action可以直接访问Web应用的ServletContext实例
* `ServletRequestAware`: 实现该接口的Action可以直接访问用户请求的HttpServletRequest实例
* `ServletResponseAware`: 实现该接口的Action可以直接访问服务器响应的HttpServletResponse实例

####使用ServletActionContext访问Servlet API
Struts2还提供了一个ServletActionContext工具类，这个类包含如下几个静态方法
* `static PageContext getPageContext()`: 取得Web应用的PageContext对象
* `static HttpServletRequest getRequest()`: 取得Web应用的HttpServletRequest对象
* `static HttpServletResponse getResponse()`: 取得Web应用的HttpServletResponse对象
* `static ServletContext getServletContext()`: 取得Web应用的ServletContext对象

###Action包和命名空间
在struts.xml中配置Action，如果没有指定namespace属性，那么该包使用默认的命名空间，如果namespace="/"表示指定根命名空间。默认命名空间里的Action可以处理任何命名空间下的Action请求，但根命名空间下的Action只处理根命名空间下的Action请求，这是根命名空间和默认命名空间的区别。
####配置默认Action
```xml
<package name="..." extends="struts-default">
    <default-action-ref name="simpleViewAction"/>
    //当用户请求找不到对应的Action时，系统默认的Action将处理用户请求
    <action name="simpleViewAction" class="...">
        <result.../>
    </action>
</package>
```
将默认Action配置在默认命名空间里就可以让该Action处理所有用户请求，因为默认命名空间的Action可以处理任何命名空间的请求。

###Struts2内建的支持结果类型
* chain: Action链式处理的结果类型
* dispatcher: 用于指定使用JSP作为视图的结果类型，默认值
* freemarker: 用于指定使用FreeMarker模板作为视图的结果类型
* httpheader: 用于控制特殊的HTTP行为的结果类型
* redirect: 用于直接跳转到其他URL的结果类型
* redirectAction: 用于直接跳转到其他Action的结果类型
* stream: 用于向浏览器返回一个InputStream
* velocity: 用于指定使用Velocity模板作为视图的结果类型
* xslt: 用于与XML/XSLT整合的结果类型
* plainText: 用于显示某个页面的原始代码的结果类型

***Notes***
1. dispatcher结果类型是将请求forward（转发）到指定的JSP资源；而redirect结果类型，是将请求redirect（重定向）到指定的视图资源，重定向会丢失所有的请求参数、请求属性——当然也丢失了Action的处理结果。
2. 使用redirect类型的结果时，不能重定向到`/WEB-INF/`路径下任何资源，因为重定向相当于重新发送请求，而Web应用的`/WEB-INF/`路径下资源是受保护资源。
3. 使用redirectAction结果类型时，系统将重新生成一个新请求，只是该请求的URL是一个Action，因此前一个Action处理结果、请求参数、请求属性都会丢失。

###Struts2异常处理
####Struts2开启异常映射
默认的在struts-default.xml中已经开启了异常映射，代码如下
```xml
<interceptors>
    <!--配置异常处理的拦截器-->
    <interceptor name="exception" class="com.opensymphony.xwork.interceptor.ExceptionMapping.Interceptor"/>
    <interceptor-stakc name="defaultStack">
        ...
        <!--加入默认的拦截器栈-->
        <interceptor-ref name="exception"/>
        ...
    </interceptor-stakc>
</interceptors>
```
####Struts2输出异常信息
`<s:property value="exception"/>`: 输出异常对象本身
`<s:property value="exceptionStck"/>`: 输出异常堆栈信息
`<s:property value="exception.message"/>`: 输出异常的message消息
####Convention插件与“约定”支持
有Ruby on Rails开发经验的朋友知道Rails有一条重要原则：约定优于配置。Rails开发者只需要按约定开发ActiveRecord/ActiveController即可，无需进行配置。Struts2的Convention插件借鉴了Rails的创意。
#####Action的搜索和映射约定
使用Convention插件，将Struts2项目下的struts2-convention-plugin-2.3.16.3.jar复制到项目的WEB-INF/lib下即可。
对于Convention插件而言，它会自动搜索位于action、actions、struts、struts2包下的所有Java类，Convention插件会把如下两种Java类当成Action处理。
* 所有实现了com.opensymphony.xwork2.Action的Java类
* 所有类名以Action结尾的Java类

Struts2的Convention插件还允许设置如下三个常量
* struts.convention.exclude.packages: 指定不扫描哪些包下的Java类，位于这些包结构下的Java类将不会被自动映射成Action
* struts.convention.package.locators: Convention插件使用该常量指定的包作为搜寻Action的根包。例如actions.base.LoginAction类，按约定原本映射到/base/login；如果将该常量设置为base，则该Action将会映射到/login
* struts.convention.action.packages: Convention插件以该常量指定包作为根包来搜索Action类，Convention插件除了扫描action、actions、struts、struts2四个包的类之外，还会扫描该常量指定的一个或多个包，Convention会视图从中发现Action类

部署Action时，action、actions、struts、struts2包会映射成根（/）命名空间，而这些包下的子包则被映射成对应的命名空间。
例如：edu.shu.action.base.LoginAction 映射到/base/命名空间

Action的那么属性（即该Action处理的URL）根据Action的类名映射，需遵循如下两步规则：
* 如果该Action类名包含Action后缀，将该Acting类名的Action后缀去掉，否则不做任何处理
* 将Action类名的驼峰写法转成中划线写法，所有字母小写

例如edu.shu.action.base.UserLoginAction映射的URL是/base/user-login.action

#####按约定映射Result
Convention默认也为作为逻辑视图和物理视图之间的映射提供了约定，默认情况下，Convention总会到Web应用的WEB-INF/content路径下定位物理资源，定位资源的约定是actionName+resultcode+suffix。当某个逻辑视图找不到对应的视图资源时，Convention会自动视图使用actionName+sufix作为物理视图资源。

|Action的URL|返回的逻辑视图名|结果类型|对应的物理视图|
| --------   | -----:  | :----:  |
|/login|success|Dispatcher|\WEB-INF\content\login-success.jsp
|/login|success|Dispatcher|\WEB-INF\content\login-success.html
|/login|success|Dispatcher|\WEB-INF\content\login.jsp
|/login|success|Dispatcher|\WEB-INF\content\login.html
|/shu/get-book|error|FreeMarker|\WEB-INF\content\shu\get-book-error.ftl
|/shu/get-book|error|FreeMarker|\WEB-INF\content\shu\get-book.ftl
|/shu/get-book|input|Velocity|\WEB-INF\content\shu\get-book-input.vm
|/shu/get|input|Velocity|\WEB-INF\content\shu\get-input.vm

#####Action链的约定
如果希望一个Action处理结束后不是进入视图页面，而是进入另一个Action形成的Action链，则通过Convention插件只需遵守如下三个约定
* 第一个Action返回的逻辑视图字符串没有对应的视图资源
* 第二个Action与第一个Action处于同一个包下
* 第二个Action映射的URL为: firstactionName+resultcode

#####自动重加载映射
在struts.xml中配置如下两个常量即可使Convention插件重加载映射
```xml
<constant name="struts.devMode" value="true">
<constant name="struts.convention.classes.reload" value="true">
```

###Struts2标签库
####Struts2标签库概述
从最大的范围来分，Struts2可以将所有标签分成如下三类
1. UI标签: 主要用于生成HTML元素的标签
1.1. 表单标签: 主要用于生成HTML页面的form元素，以及普通表单元素的标签
1.2. 非表单元素: 主要用于生成页面的树、Tab页等标签
2. 非UI标签: 主要用于数据访问、逻辑控制等的标签
2.1 流程控制标签: 主要包含用于实现分支、循环等流程控制的标签
2.2 数据访问标签: 主要包含用于输出ValueStack中的值、完成国际化等功能的标签
3. Ajax标签: 用于Ajax支持的标签

####OGNL表达式语言
OGNL的顶级对象是Stack Context，Stack Context对象就是一个Map类型的实例，其根对象就是Value Stack。OGNL的Stack Context里除了包括ValueStack这个根之外，还包括parameters、request、session、application、attr等命名对象，但这些命名对象都不是根。Stack Context“根”对象和普通命名对象的区别在于
* 访问Stack Context里的命名对象需要在对象名之前添加#前缀
* 访问OGNL的Stack Context里“根”对象的属性时，可以省略对象名

###Struts2拦截器
####配置默认拦截器
当配置一个包时，可以为其指定默认拦截器。一旦为某个包指定了默认的拦截器，如果该包中的Action没有显式指定拦截器，则默认的拦截器将会起作用。如果一旦为该包中的Action显式应用了某个拦截器，则默认的拦截器不会起作用，如果该Action还需要使用默认拦截器，则必须手动配置该拦截器的引用。
####拦截器的执行顺序
在Action的控制方法执行之前，位于拦截器链前面的拦截器将先发生作用；在Action的控制方法执行之后，位于拦截器链前面的拦截器将后发生作用。

参考资料
【1】轻量级JavaEE企业应用实战-Struts2+Spring4+Hibernate整合开发


  [1]: http://7xig3q.com1.z0.glb.clouddn.com/struts2-logo.png
  [2]: http://7xig3q.com1.z0.glb.clouddn.com/webwork+struts=struts2.png