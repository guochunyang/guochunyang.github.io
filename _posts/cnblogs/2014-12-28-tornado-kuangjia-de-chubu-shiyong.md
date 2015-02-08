---
layout: post
title: Tornado框架的初步使用
category : C++
tagline: "Supporting tagline"
tags : [C++, Python, Socket, Tornado]
---
Tornado的搭建很简单，使用pip，或者下载源码均可。
   
  我们先看一个最简单的程序：
  

```Python
import tornado.ioloop
import tornado.web

class MainHandler(tornado.web.RequestHandler):
    def get(self):
        self.write("<h1>Hello World<h1>")

application = tornado.web.Application([(r"/", MainHandler),])

if __name__ == '__main__':
    application.listen(8888)
    tornado.ioloop.IOLoop.instance().start()
```
		

我们运行这个程序，打开浏览器输入：




```C++
http://localhost:8888/
```
		

就可以看到加粗的helloworld。


 


那么这段代码到底什么意思：


我们先看 




```Python
class MainHandler(tornado.web.RequestHandler):
    def get(self):
        self.write("Hello, world")
```
		



这里`定义了一个处理器，里面定义个一个get方法，对应Http协议中的GET请求`。


然后是：




```C++
application = tornado.web.Application([
    (r"/", MainHandler),
])
```
		

这里的含义是：如果用户输入的路径是“/”，也就是根路径，那么将使用我们刚才编写的MainHandler，`如果该请求使用的GET，那么调用MainHandler的get方法，如果是POST请求，则去调用MainHandler中的post方法`。


 


所以我们输入上面的网址，tornado调用了MainHandler中的get方法，返回"<h1>Hello World<h1>"


 


我们再看一个稍微复杂的程序：




```Python
import tornado.ioloop
import tornado.web

class MainHandler(tornado.web.RequestHandler):
    def get(self):
        self.write("<h1>This is Home Page!</h1>")

class StoryHandler(tornado.web.RequestHandler):
    def get(self, story_id):
        self.write("You request the story <h1>" + story_id + "</h1>")

application = tornado.web.Application([
    (r"/", MainHandler),
    (r"/story/([0-9]+)", StoryHandler),
    ])

if __name__ == '__main__':
    application.listen(8888)
    tornado.ioloop.IOLoop.instance().start()
```
		

上面代码的含义是：



  对于/根目录，使用MainHandler，处理GET请求。


  `对于/story/99这种，使用StoryHandler，处理GET请求`。



下面看一个更加复杂的程序：




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

application = tornado.web.Application([
    (r"/", MainHandler),
    (r"/story/([0-9]+)", StoryHandler),
    (r"/message", MessageHandler),
    ])

if __name__ == '__main__':
    application.listen(8888)
    tornado.ioloop.IOLoop.instance().start()
```
		


这里，输入/message这个URL，使用的是MessageHandler，调用其中的get方法，返回一段HTML代码，其中含有一个表单，提交后，`仍使用/message，但是此时采用POST请求提交`。后端Tornado收到这段数据，采用MessageHandler的post方法，处理这段文本，将其回显给用户。
			