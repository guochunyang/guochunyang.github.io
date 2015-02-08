---
layout: post
title: Struts2环境搭建以及helloworld编写
category : C++
tagline: "Supporting tagline"
tags : [C++, Python, Socket, Tornado]
---
Struts是一个经典的Java Web开发框架。现在我们编写一个简易的helloworld界面。
   
  Struts的环境搭建比较简单，加载相应的jar包即可。
  我这里的开发环境是windows7 + Myeclipse10.0
  Struts2采用的是2.2.1.1版本
   
  1.加载Struts2的必备jar包，我这里是6个，还有另外的两个jar文件。
     这六个分别是：
    1.commons_fileupload-1.2.1.jar
    2.commons-io_1.3.2.jar
    3.commons-logging-1.0.4.jar
    4.ognl-3.0.jar
    5.struts2-core-2.2.1.1.jar
    6.xwork-core-2.2.1.1.jar
      此外，还需要freemarker-2.3.16.jar以及javassist-3.7.ga.jar文件
   2.编辑web.xml文件
  内容为：
  

```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app version="2.5" xmlns="http://java.sun.com/xml/ns/javaee"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://java.sun.com/xml/ns/javaee 
    http://java.sun.com/xml/ns/javaee/web-app_2_5.xsd">

    <filter>
        <filter-name>struts2</filter-name>
        <filter-class>org.apache.struts2.dispatcher.ng.filter.StrutsPrepareAndExecuteFilter</filter-class>
    </filter>
    
    <filter-mapping>
        <filter-name>struts2</filter-name>
        <url-pattern>/*</url-pattern>
    </filter-mapping>
</web-app>
```
		



这个xml文档不需要验证，即使上面的网址失效也无妨。


 


3.在src目录下创建struts.xml文件，内容为：




```Python
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE struts PUBLIC
"-//Apache Software Foundation//DTD Struts Configuration 2.1.7//EN"
"http://struts.apache.org/dtds/struts-2.1.7.dtd">

<struts>
    <package name="struts2" extends="struts-default">
    
        <action name="helloworld" class="com.test.action.HelloWorldAction">
            <result name="success">/helloworld.jsp</result>
        </action>
    
    </package>
    
</struts>
```
		



这个xml文档是最关键的配置部分。`而且需要DTD验证，所以必须保证上面的dtd文件是可用`的。


 


4.创建一个package为com.test.action，创建class为HelloWorldAction。


该class的定义为：




```Python
package com.test.action;

import com.opensymphony.xwork2.ActionSupport;

public class HelloWorldAction extends ActionSupport
{
    private static final long serialVersionUID = 1L;

    @Override
    public String execute() throws Exception
    {
        return SUCCESS;
    }
}
```
		

这里比较简单，我们也可以不继承ActionSupport类，`只要这个类具备execute函数即可`。


5.创建一个jsp文件，为helloworld.jsp


内容如下：




```Python
<%@ page language="java" import="java.util.*" pageEncoding="UTF-8"%>
<%
String path = request.getContextPath();
String basePath = request.getScheme()+"://"+request.getServerName()+":"+request.getServerPort()+path+"/";
%>

<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN">
<html>
  <head>
    <base href="<%=basePath%>">
    <title>我的第一个Struts界面</title>
  </head>
  
  <body>
    <h1>Hello World</h1>
  </body>
</html>
```
		



 


现在我们把程序部署到tomcat上，然后启动server，在浏览器中访问：



  <a title="http://localhost:8080/struts2/helloworld" href="http://localhost:8080/struts2/helloworld">http://localhost:8080/struts2/helloworld</a> 注意我的项目名称是struts2



就可以看到加粗后的helloworld。


 


下面分析访问helloworld页面的流程。


1.首先web.xml中，我们为所有的url都配置了一个分配器，所以当我们输入上面的网址时，server接收到的是/helloworld。


2.到了关键的地方，我们看
  

```C++
<action name="helloworld" class="com.test.action.HelloWorldAction">
            <result name="success">/helloworld.jsp</result>
        </action>
```
		



他的意思是，`对于helloworld，我们去执行HelloWorldAction这个类的execute方法。`



###如果返回结果为success，那么执行helloworld.jsp页面


于是我们在浏览器中就看到了helloworld页面。

			