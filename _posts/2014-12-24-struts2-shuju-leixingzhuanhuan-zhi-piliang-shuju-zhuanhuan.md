---
layout: post
title: Struts2数据类型转换之批量数据转换
category : C++
tagline: "Supporting tagline"
tags : [C++, Python, Socket, Tornado]
---
前面我们实现了从字符串到User对象的转换。如果表单中有多个User数据，我们可以批量转换。
  我们把input.jsp修改为：
  

```html
<h1>使用分号隔开username password</h1>
    
    <form action="userAction2.action">
    
        <input type="text" name="user"> <br>
        <input type="text" name="user"> <br>
        <input type="text" name="user"> <br>
        <input type="text" name="user"> <br>
        <input type="text" name="user"> <br>
        
        
        <input type="submit" name="submit">
    </form>
```
		



然后新建action，UserAction2：




```Python
package com.test.action;

import java.util.List;

import com.opensymphony.xwork2.ActionSupport;

public class UserAction2 extends ActionSupport
{
    private List<String> user;

    public List<String> getUser()
    {
        return user;
    }

    public void setUser(List<String> user)
    {
        this.user = user;
    }
    
    @Override
    public String execute() throws Exception
    {
        return SUCCESS;
    }
}
```
		

 


下面我们就要进行转换，此时我们需要的是将表单上一堆字符串，转化成一个String集合。


编写转换器，UserConverter3：




```Python
package com.test.converter;

import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.StringTokenizer;

import org.apache.struts2.util.StrutsTypeConverter;

import com.test.bean.User;

public class UserConverter3 extends StrutsTypeConverter
{
    @Override
    public Object convertFromString(Map context, String[] values, Class toClass)
    {
        List<User> users = new ArrayList<User>();
        
        for(String value : values)
        {
            StringTokenizer st = new StringTokenizer(value, ";");
            String username = st.nextToken();
            String password = st.nextToken();
            
            User user = new User();
            user.setUsername(username);
            user.setPassword(password);
            
            users.add(user);
        }
        
        return users;
    }
    
    @Override
    public String convertToString(Map context, Object o)
    {
        @SuppressWarnings("unchecked")
        List<User> list = (List<User>)o;
        
        StringBuffer sbuf = new StringBuffer();
        
        for(User user : list)
        {
            sbuf.append("username: " + user.getUsername() + ", password: " + user.getPassword() + "\n");
        }
        
        return sbuf.toString();
    }
}
```
		



然后建立类型转换的配置文件和修改struts.xml。


启动服务器，是可以正常工作的。

			