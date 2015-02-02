---
layout: post
title: Tornado框架中视图模板Template的使用
category : C++
tagline: "Supporting tagline"
tags : [C++, Python, Socket, Tornado]
---
上文的程序中有这样一段：
  

```Python
class MessageHandler(tornado.web.RequestHandler):
    def get(self):
        self.write('''
<html>
<head>
        <title>Please Input Message</title>
</head>
<body>
        <form action="/message" method="post">
                <input type="text" name="message"><br>
                <input type="submit" value="submit">
        </form>
</body>
</html>''' 
        )
    def post(self):
        #self.set_header("Content-Type", "text/plain")
        self.write("You wrote <h1>" + self.get_argument("message") + "</h1>")
```
		

当收到GET请求时，返回一段HTML表单。


上面的这种写法，将html写在python代码中，灵活性差，而且view代码与controller代码混合在一块，不符合MVC的原则。


所以我们采用Tornado中的模板。


新建form.html：




```html
<html>
    <head>
    <title>{{title}}</title>
    </head>
    <body>
        <form action="/message" method="post">
            <input type="text" name="message" value="please input.">
            <input type="submit" value="submit">
        </form>
    </body>
</html>
```
		

然后将上面的python代码修改为：


 




```Python
class MessageHandler(tornado.web.RequestHandler):
    def get(self):
        self.render("form.html", title="Input Message")
    def post(self):
        #self.set_header("Content-Type", "text/plain")
        self.write("You wrote <h1>" + self.get_argument("message") + "</h1>")
```
		

这样代码简洁了很多。


 






完整的代码是：




```Python
import tornado.ioloop
import tornado.web

class MainHandler(tornado.web.RequestHandler):
    def get(self):
        self.write("<h1>This is Home Page!</h1>")

class StoryHandler(tornado.web.RequestHandler):
    def get(self, story_id):
        self.write("You request the story <h1>" + story_id + "</h1>")

class MessageHandler(tornado.web.RequestHandler):
    def get(self):
        self.render("form.html", title="Input Message")
    def post(self):
        #self.set_header("Content-Type", "text/plain")
        self.write("You wrote <h1>" + self.get_argument("message") + "</h1>")

application = tornado.web.Application([
    (r"/", MainHandler),
    (r"/story/([0-9]+)", StoryHandler),
    (r"/message", MessageHandler),
    ])

if __name__ == '__main__':
    application.listen(8888)
    tornado.ioloop.IOLoop.instance().start()
```
		
			