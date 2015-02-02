---
layout: post
title: Struts2数据校验
category : C++
tagline: "Supporting tagline"
tags : [C++, Python, Socket, Tornado]
---
回到之前的LoginAction，我们只是简单的回显了数据，但我们的操作都是假设数据是正确的。但是假设我们在输入age的时候输入了字符串，服务器就会throw异常，而且age也无法接收到正确的值。
  所以我们需要对数据进行校验。
   
  Struts2中的数据不合法分两种情况：
     一是Field级别的错误，例如给age输入字符串，还有Date不按格式输入
      二是Action级别的逻辑错误，例如username不能过长，两次password必须相同等。
   下面以一个注册RegisterAction说明Struts2如何进行数据校验。
   
  我们先编写jsp页面。
  register.jsp的内容如下：
   
  

```Python
<%@ page language="java" import="java.util.*" pageEncoding="UTF-8"%>
<%@ taglib prefix="s" uri="/struts-tags" %>
<%
String path = request.getContextPath();
String basePath = request.getScheme()+"://"+request.getServerName()+":"+request.getServerPort()+path+"/";
%>

<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN">
<html>
  <head>
    <base href="<%=basePath%>">
    
    <title>用户注册</title>

  </head>
  
  <body>
      <h2><font color="blue">用户注册</font></h2>
    <s:actionerror cssStyle="color:red"/>
    ----------------------------------------
    <s:fielderror cssStyle="color:blue"></s:fielderror>
    ----------------------------------------
    <s:form action="registerAction" theme="simple">
        username: <s:textfield name="username" label="username"></s:textfield><br>
        password: <s:password name="password" label="password"></s:password><br>
        repassword: <s:password name="repassword" label="repassword"></s:password><br>
        age: <s:textfield name="age" label="age"></s:textfield><br>
        birthday: <s:textfield name="birthday" label="birthday"></s:textfield><br>
        graduation: <s:textfield name="graduation" label="graduation"></s:textfield><br>
    
    <s:submit value="submit"></s:submit>
    </s:form>
  </body>
</html>
```
		

register_result.jsp内容如下：




```Python
<%@ page language="java" import="java.util.*" pageEncoding="UTF-8"%>
<%@ taglib prefix="s" uri="/struts-tags" %>
<%
String path = request.getContextPath();
String basePath = request.getScheme()+"://"+request.getServerName()+":"+request.getServerPort()+path+"/";
%>

<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN">
<html>
  <head>
    <base href="<%=basePath%>">
    
    <title>用户注册结果</title>

  </head>
  
  <body>
  
    username: <s:property value="username"/><br>
    password: <s:property value="password"/><br>
    age:<s:property value="age"/><br>
    birthday:<s:property value="birthday"/><br>
    graduate:<s:property value="graduation"/>
  </body>
</html>
```
		



 


然后我们编写核心的RegisterAction，




```Python
package com.test.action;

import java.util.Date;

import com.opensymphony.xwork2.ActionSupport;

public class RegisterAction extends ActionSupport
{
    private String username;
    private String password;
    private String repassword;
    private int age;
    private Date birthday;
    private Date graduation;

    //get 和 set方法

    

    @Override
    public String execute() throws Exception
    {
        return SUCCESS;
    }
}
```
		



 


这跟以前我们编写的步骤完全相同，但是这里我们需要进行数据校验。所以我们添加一个validate方法，这个也是对基类的重写。




```C++
@Override
    public void validate()
    {
        if (username == null || username.length() < 4 || username.length() > 8)
        {
            this.addActionError("username invalid!.");
        }

        if (password == null || password.length() < 4 || password.length() > 8)
        {
            this.addActionError("password invalid!!");
        }
        else if (!password.equals(repassword))
        {
            this.addActionError("the passwords not the same");
        }

        if (!birthday.before(graduation))
        {
            this.addActionError("birthday must be earlier than graduation!!");
        }
        
        if(birthday != null && graduation != null && !birthday.before(graduation))
        {
            this.addActionError("birthday must be earlier than graduation!!");
        }
    }
```
		



我们可以看上，上面就是对username等进行逻辑上的校验。


然后我们修改struts.xml，可以去查看结果。


 


还有一种校验，是采用配置文件，我们将上面的validate注释掉，然后在action包下面，新建文件RegisterAction-validation.xml文件，内容为：




```xml
<?xml version="1.0" encoding="UTF-8"?>

<!DOCTYPE validators PUBLIC "-//OpenSymphony Group//XWork Validator 1.0.2//EN" 
"http://struts.apache.org/dtds/xwork-validator-1.0.2.dtd">

<validators>
    <field name="username">
        <field-validator type="requiredstring">
            <message>username can't be blank!</message>
        </field-validator>
        <field-validator type="stringlength">
            <param name="minLength">4</param>
            <param name="maxLength">8</param>
            <message key="username.invalid"></message>
        </field-validator>
    </field>
    
    <field name="password">
        <field-validator type="requiredstring">
            <message>password can't be blank!</message>
        </field-validator>
        <field-validator type="stringlength">
            <param name="minLength">4</param>
            <param name="maxLength">8</param>
            <message>
                length of password should be between ${minLength} and ${maxLength}
            </message>
        </field-validator>
    </field>
    
    <field name="age">
        <field-validator type="required">
            <message>age can't be blank!</message>
        </field-validator>
        <field-validator type="int">
            <param name="min">10</param>
            <param name="max">40</param>
            <message>age should be between ${min} and ${max}</message>
        </field-validator>
        
    </field>
    
    <field name="birthday">
        <field-validator type="required">
            <message>birthday can't be blank!</message>
        </field-validator>
        <field-validator type="date">
            <param name="min">2005-1-1</param>
            <param name="max">2007-12-31</param>
            <message>birthday should be between ${min} and ${max}</message>
        </field-validator>
    </field>
</validators>
```
		



起到的作用是相同的，当然，这种方式相对于第一种比较死板。


 


上面的校验都是Action级别的错误。如果是Field级别的错误，系统默认信息是XXX invalid，我们可以配置信息，显示更具体一些：


在action下新建文件RegisterAction.properties。内容如下：




```C++
invalid.fieldvalue.age \u5E74\u9F84\u5FC5\u987B\u4E3A\u6574\u6570
invalid.fieldvalue.birthday=\u751F\u65E5\u4E0D\u5408\u6CD5
invalid.fieldvalue.graduation=\u6BD5\u4E1A\u65E5\u671F\u4E0D\u5408\u6CD5
```
		


这里的信息是中文。此时我们在age中输入字符串时，显示的就是中文错误提示。
			