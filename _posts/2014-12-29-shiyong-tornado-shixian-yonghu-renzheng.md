---
layout: post
title: 使用tornado实现用户认证
category : C++
tagline: "Supporting tagline"
tags : [C++, Python, Socket, Tornado]
---
关于用户的登录状态，一部分的应用程序是采用session实现的。
  HTTP是一个无状态协议，用户的每次请求都是相互独立的，HTTP本身意识不到用户是否登录。
  很多web框架选择将session存放在cookies中，本节我们也是这样实现：
  

```Python
import tornado.ioloop
import tornado.web

class BaseHandler(tornado.web.RequestHandler):
    def get_current_user(self):
        return self.get_secure_cookie("user")

class MainHandler(BaseHandler):
    def get(self):
        if not self.current_user:
            self.redirect("/login")
            return
        name = tornado.escape.xhtml_escape(self.current_user)
        self.write("Hello, " + name)

class LoginHandler(tornado.web.RequestHandler):
    def get(self):
        self.render("login.html")
    def post(self):
        self.set_secure_cookie("user", self.get_argument("name"))
        self.redirect("/")

application = tornado.web.Application([
    (r"/", MainHandler),
    (r"/login", LoginHandler)
    ], cookie_secret="61oETzKXQAGaYdkL5gEmGeJJFuYh7EQnp2XdTP1o/Vo=")

if __name__ == '__main__':
    application.listen(8888)
    tornado.ioloop.IOLoop.instance().start()
```
		

这里的核心便是LoginHandler类，他的get方法对于HTTP的GET请求，返回一个表单，对于post方法，则认为是用户的登录动作。


登录页面login.html代码如下：




```html
<html>
    <head>
        <title>Login</title>
    </head>
    <body>
        <form action="/login" method="post" accept-charset="utf-8">
            <input type="text" name="name" value="your username"><br>
            <input type="submit" value="submit">
        </form>
    </body>
</html>
```
		

这里处理实际登录的逻辑是，在cookies中存入相应的数据。


这样，`我们检测用户是否登录，只需检测cookies即可，这是BaseHandler的核心逻辑，它重写了父类的get_current_user方法`。


注意MainHandler的逻辑：



  如果用户没有登录，那么跳转到登录页面。




  如果用户登录，那么打印出欢迎的语句。




###采用装饰器


 


这里的检查登录的代码，我们可以使用闭包写一个装饰器，这样可以减少代码的冗余：




```Python
login_url = "login.html"

def require_login():
    def temp(func):
        def wrapped(self, *args, **kargs):
            if not self.current_user:
                self.redirect(login_url)
            return
        return wrapped
    return temp
```
		

这样我们在MainHandler中只需要采用装饰器修饰即可：




```Python
class MainHandler(BaseHandler):
    @require_login()
    def get(self):
        name = tornado.escape.xhtml_escape(self.current_user)
        self.write("Hello, " + name)
```
		

 



###采用框架提供的装饰器


 


MainHandler需要检测用户是否登录，`我们可以采用装饰器@tornado.web.authenticated来帮助我们完成这一目标，而不需要手工写出检测的代码`。




```Python
import tornado.ioloop
import tornado.web

class BaseHandler(tornado.web.RequestHandler):
    def get_current_user(self):
        return self.get_secure_cookie("user")

class MainHandler(BaseHandler):
    @tornado.web.authenticated
    def get(self):
        name = tornado.escape.xhtml_escape(self.current_user)
        self.write("Hello, " + name)

class LoginHandler(tornado.web.RequestHandler):
    def get(self):
        self.render("login.html")
    def post(self):
        self.set_secure_cookie("user", self.get_argument("name"))
        self.redirect("/")

settings = {
        "cookie_secret": "61oETzKXQAGaYdkL5gEmGeJJFuYh7EQnp2XdTP1o",
        "login_url": "/login",
}

application = tornado.web.Application([
    (r"/", MainHandler),
    (r"/login", LoginHandler)
    ], **settings) 

if __name__ == '__main__':
    application.listen(8888)
    tornado.ioloop.IOLoop.instance().start()
```
		

很显然，@tornado.web.authenticated这个装饰器的功能`与我们编写的require_login功能相似`。


另外，在自己编写的装饰器中，`我们将login_url单独做成了变量，保证可配置性，所以这里我们也需要配置login_url选项`。

			