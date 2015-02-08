---
layout: post
title: Struts2之简单数据类型转换
category : C++
tagline: "Supporting tagline"
tags : [C++, Python, Socket, Tornado]
---
如果我们在网页表单上填写数字，那么在后台我们是否需要手工把字符串转化为数字呢？
   
  我们编写以下的程序：
  1.创建文件login.jsp，核心内容如下：
  

```html
<form action="login.action">
        username: <input type="text" name="username"> <br>
        password: <input type="password" name="password"><br>
        age: <input type="text" name="age"> <br>
        date: <input type="text" name="date"> <br>
        <input type="submit" value="submit">
    </form>
```
		

2.创建文件login_result.jsp核心内容如下：




```C++
username: ${requestScope.username}<br>
    password: ${requestScope.password }<br>
    age: ${requestScope.age }<br>
    date: ${requestScope.date }<br>
```
		

打印出四个变量的内容。


3.在struts.xml配置如下：




```C++
<action name="login" class="com.test.action.LoginAction">
            <result name="success">/login_result.jsp</result>
        </action>
```
		

4.最后我们编写LoginAction类。


因为表单上我们编写了四个内容，所以类中对应四个变量。


 




```Python
package com.test.action;

import java.util.Date;

public class LoginAction
{
    private String username;
    private String password;
    private int age;
    private Date date;
    
    //set and get method
    
    public String execute()
    {
        return "success";
    }
}
```
		

然后我们重新部署程序，发现运行是正常的，说明struts2自动帮我们完成了转换，当然如果我们填写的数据不合法，例如age填写了字符串，那么会抛出异常。


 


事实上，`对于Java内置的数据类型，struts2会自动帮我们完整转化。但是对于我们自定义的类型，我们必须手工进行转化`。

			