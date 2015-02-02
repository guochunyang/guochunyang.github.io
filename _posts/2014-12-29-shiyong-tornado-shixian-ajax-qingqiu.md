---
layout: post
title: 使用Tornado实现Ajax请求
category : C++
tagline: "Supporting tagline"
tags : [C++, Python, Socket, Tornado]
---
Ajax，指的是网页异步刷新，一般的实现均为js代码向server发POST请求，然后将收到的结果返回在页面上。
   
  这里我编写一个简单的页面，ajax.html
  

```html
<html>
<head>
    <title>测试Ajax</title>
    <meta http-equiv="Content-Type" content="text/html; charset=utf-8" />
    <script src="http://code.jquery.com/jquery-1.9.1.min.js"></script>  

    <style type="text/css">
#result{
    border: 10px;
    font-size: 50px;
    background: #ff0fef;
}


    </style>
</head>
<body>

    <input type="text" id="word" > <br>
    <button id="foo">点击</button>

    <div id="result">

    </div>


<script type="text/javascript">
    $("#foo").click(function()
    {
        var word = $("#word").val(); //获取文本框的输入

        //把word发给后台php程序
        //返回的数据放在data中，返回状态放在status
        $.post("/test",{message:word}, function(data,status){
            if(status == "success")
            {
                $("#result").html(data);
            }
            else
            {
                alert("Ajax 失败");
            }
        });
    });


</script>

</body>
</html>
```
		

注意，从上面的代码可以看出，数据存储在“message”字段中。


所以后台从message中解析数据，我们记得是get_argument方法。


所以后台的python代码为：




```Python
import tornado.ioloop
import tornado.web

class MainHandler(tornado.web.RequestHandler):
    def get(self):
        self.render("ajax.html")

class AjaxHandler(tornado.web.RequestHandler):
    def post(self):
        #self.write("hello world")
        self.write(self.get_argument("message"))

application = tornado.web.Application([
    (r"/", MainHandler),
    (r"/test", AjaxHandler),
    ])

if __name__ == '__main__':
    application.listen(8888)
    tornado.ioloop.IOLoop.instance().start()
```
		
访问首页，就渲染ajax前端页面，而AjaxHandler处理真正的ajax异步请求。 

 


这里总结下流程：



  1.用户访问home页面，tornado使用MainHandler返回其中的ajax.html页面




  2.用户填写信息，点击按钮，因为之前加载js代码，注册了回调函数，所以触发Ajax




  3.js向后台发post请求。




  4.根据请求的URL，tornado使用AjaxHandler处理post请求，将信息原样返回。




  5.js收到数据，调用之前的回调函数，将结果显示在html页面上。

			