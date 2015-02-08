---
layout: post
title: Struts2数据类型转换之自定义数据类型转换（二）
category : C++
tagline: "Supporting tagline"
tags : [C++, Python, Socket, Tornado]
---
对于自定义的类型转换器来说，需要提供三个信息：
  Action的名字、Action中带转换的属性名、该属性对应的转换器类
 其中，action的名字是通过属性文件名来获取的，action中待转换的属性名是通过文件中的key获得的，该属性对应的转换器类是通过该key对应的value获取的。
 例如上节中的UserAction-converter.properties，其文件内容为：
 <pre>user=com.test.converter.UserConverter</pre><pre>这就告诉我们，`对于UserAction这个action，我们可以使用`</pre><pre>`com.test.converter.UserConverter这个类完成user这个属性的转换工作`。</pre><pre> </pre><pre>上节中我们的转换器类使用的是ognl提供的DefaultTypeConverter类，现在我们采用Struts提供的StrutsTypeConverter来简化代码的编写。</pre><pre> </pre><pre>StrutsTypeConverter继承自DefaultTypeConverter，并且`提供了两个抽象方法：converterFromString与convertToString`，分别表示从页面的String转化为后台对象，以及从后台对象转化为页面的字符串。</pre><pre> </pre><pre>我们只需要实现这两个抽象方法即可。</pre><pre>我们新建一个类，UserConverter2：</pre>


```Python
package com.test.converter;

import java.util.Map;
import java.util.StringTokenizer;

import org.apache.struts2.util.StrutsTypeConverter;

import com.test.bean.User;

public class UserConverter2 extends StrutsTypeConverter
{
    @Override
    public Object convertFromString(Map context, String[] values, Class toClass)
    {
        User user = new User();

        String str = values[0];

        StringTokenizer st = new StringTokenizer(str, ";");
        String username = st.nextToken();
        String password = st.nextToken();

        user.setUsername(username);
        user.setPassword(password);

        return user;
    }

    @Override
    public String convertToString(Map context, Object o)
    {
        User user = (User) o;

        String username = user.getUsername();
        String password = user.getPassword();
        String userInfo = "username: " + username + ", password: " + password;

        return userInfo;
    }
}
```
		
然后在UserAction-converter.properties将原来的内容注释掉，修改为：



```C++
user=com.test.converter.UserConverter2
```
		

 

重新打开页面，发现仍然可以正常工作。

 

如果在项目中大量使用User类，我们可以写一个全局的配置文件。在src下新建xwork-conversion.properties，内容为：



```C++
com.test.bean.User=com.test.converter.UserConverter2
```
		
这意味着，凡是需要转换User类型的属性，如果我们没有像上面那样针对某个Action写配置，那么就采用默认的UserConverter2类去实现转换。
			