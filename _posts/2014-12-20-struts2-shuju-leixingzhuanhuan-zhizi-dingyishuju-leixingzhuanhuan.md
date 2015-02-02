---
layout: post
title: Struts2数据类型转换之自定义数据类型转换
category : C++
tagline: "Supporting tagline"
tags : [C++, Python, Socket, Tornado]
---
前面提到，对于Java内置的int String和Date等，struts可以帮助我们完成数据的转化，但是对于自定义类型，必须手动实现转化方法。
  下面我们自己实现一个User对象的转换。
   
  1.首先我们自定义一个Java bean。建一个package com.test.bean，新建class为User：
  

```C++
package com.test.bean;

public class User
{
    private String username;
    private String password;
    
    public String getUsername()
    {
        return username;
    }
    public void setUsername(String username)
    {
        this.username = username;
    }
    public String getPassword()
    {
        return password;
    }
    public void setPassword(String password)
    {
        this.password = password;
    }

}
```
		

2.建立一个input.jsp 和一个output.jsp


input的内容为一个表单：




```html
<h1>使用分号隔开username password</h1>
    
    <form action="userAction.action">
    
        <input type="text" name="user"> <br>
        <input type="submit" name="submit">
    </form>
```
		

output的内容为：




```C++
<s:property value="user"/>
```
		

这里使用了struts自定义的标签，所以顶部第二行加上：




```C++
<%@ taglib prefix="s" uri="/struts-tags" %>
```
		

3.在struts.xml中配置一个路由表：




```C++
<action name="userAction" class="com.test.action.UserAction">
            <result name="success">/output.jsp</result>
        </action>
```
		

4.编写一个UserAction：




```Python
package com.test.action;

import com.opensymphony.xwork2.ActionSupport;
import com.test.bean.User;

public class UserAction extends ActionSupport
{
    private static final long serialVersionUID = 1L;
    
    private User user;
    
    public User getUser()
    {
        return user;
    }

    public void setUser(User user)
    {
        this.user = user;
    }


    @Override
    public String execute() throws Exception
    {
        System.out.println("username : " + user.getUsername());
        System.out.println("password : " + user.getPassword());

        return SUCCESS;
    }
}
```
		

5.下面就开始真正的转换：


新建一个package com.test.converter 新建类UserConverter




```Python
package com.test.converter;

import java.util.Map;
import java.util.StringTokenizer;

import ognl.DefaultTypeConverter;

import com.test.bean.User;

public class UserConverter extends DefaultTypeConverter
{
    @Override
    public Object convertValue(Map context, Object value, Class toType)
    {
        if (User.class == toType) // 把网页上的字符串转化为User对象
        {
            String[] str = (String[]) value;
            String firstValue = str[0];

            StringTokenizer st = new StringTokenizer(firstValue, ";");

            String username = st.nextToken();
            String password = st.nextToken();

            User user = new User();
            user.setUsername(username);
            user.setPassword(password);

            return user;
        }
        else if (String.class == toType)// 将User对象转化为字符串
        {
            User user = (User) value;

            String username = user.getUsername();
            String password = user.getPassword();

            String userInfo = "username: " + username + ", password: "
                    + password;

            return userInfo;
        }

        return null;
    }
}
```
		

可以看到，我们的转化器类继承了ognl提供的DefaultTypeConverter类。


6.在package converter下面，新建一个文件，名字为UserAction-converter.properties


内容只有一行：




```C++
user=com.test.converter.UserConverter
```
		


###它的意思是，如果要转化UserAction中的成员变量user，应该使用com.test.converter.UserConverter这个类


`注意这个文件的命名规则和大小写`。


 


打开浏览器，输入
  

```C++
http://localhost:8080/struts2/input.jsp
```
		



















输入：



  hello;world



可以看到：



  username: hello, password: world 



下面我们分析一下流程：


当我们在jsp页面中输入hello;world时，后台server根据我们的配置文件交给UserAction去处理，但是这个User不是内置类型，所以他通过UserAction-converter.properties这个配置文件，找到UserConverter这个类，将字符串转化为一个User对象。


 


下文采用struts2提供的转化器，简化流程。

			