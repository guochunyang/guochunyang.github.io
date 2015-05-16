---
layout: post
title: 使用tornado的gen模块改善程序性能
category : C++
tagline: "Supporting tagline"
tags : [C++, Python, Socket, Tornado]
---
之前在公司的一个模块，需要从另一处url取得数据，我使用了Python的一个很著名的lib，叫做requests。但是这样做极大的降低了程序的性能，因为tornado是单线程的，它使用了所谓的reactor模式，底层使用epoll监听每个tcp连接，上层再经过封装，接受HTTP请求。所以，tornad实际上是单线程的。
  在实际的场景中，经常采用nginx反向代理的模式，然后服务器开启多个tornado进程，接受nginx发送过来的请求。
  刚才的问题主要是，因为requests是阻塞的，所以当我发出一个post请求，整个tornado进程就阻塞了，此时该进程不能接受任何的其他请求。
  想想我们的服务器总共才十几个tornado进程，可能要应对上千的并发量，所以阻塞一个进程对我们是巨大的损失。
  tornado内置了异步的模块，例如AsyncHttpClient，它的使用如下：
  

```Python
class MainHandler(tornado.web.RequestHandler):
    @tornado.web.asynchronous
    def get(self):
        http = tornado.httpclient.AsyncHTTPClient()
        http.fetch("http://friendfeed-api.com/v2/feed/bret",
                   callback=self.on_response)

    def on_response(self, response):
        if response.error: raise tornado.web.HTTPError(500)
        json = tornado.escape.json_decode(response.body)
        self.write("Fetched " + str(len(json["entries"])) + " entries "
                   "from the FriendFeed API")
        self.finish()
```
		

那么，tornado的gen是怎么回事？ 可以看到，上面的代码中使用了回调函数，但是回调函数有一个致命的问题，如果逻辑非常复杂，那么我们的程序可能嵌套多层回调，造成所谓的“回调地狱”。


事实上，tornado的gen模块，就是为了改善这一问题。例如：




```Python
class GenAsyncHandler(RequestHandler):
    @gen.coroutine
    def get(self):
        http_client = AsyncHTTPClient()
        response = yield http_client.fetch("http://example.com")
        do_something_with_response(response)
        self.render("template.html")
```
		

从上面可以看出，gen模块的最大作用，就是将异步代码的编写进行改进，使其看起来像同步。


上面的代码执行时，遇到yield后面的阻塞调用则暂停，然后去执行其他请求，等该数据返回时，再继续处理这里。


这样就防止了一个IO操作阻塞整个进程。


在实际应用中，对于可能阻塞的操作（例如查询量较大的数据库查询），最好使用异步。

			