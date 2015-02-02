---
layout: post
title: Linux非阻塞IO（二）网络编程中非阻塞IO与IO复用模型结合
category : C++
tagline: "Supporting tagline"
tags : [C++, Python, Socket, Tornado]
---
上文描述了最简易的非阻塞IO，采用的是轮询的方式，这节我们使用IO复用模型。
   
  
###阻塞IO
   
  过去我们使用IO复用与阻塞IO结合的时候，`IO复用模型起到的作用是并发监听多个fd`。
  以简单的回射服务器为例，我们只监听了某fd是否可读，一旦fd有数据，我们立刻read，然后将其write给对方。
  `在阻塞IO里面，我们总是认为fd是可写的`。因为即使底层的IO缓冲区已满，稍微等待片刻即可。这与read卡在一个无数据的fd上是两种情况。`所以从这个角度出发，是不需要监听fd的写事件的`。
  总之，在阻塞IO中，收到数据然后write，这二者是同时使用的。
   
  
###非阻塞IO
   
  到了非阻塞IO里面，事情就远远不是这么简单了。此时，IO绝不仅仅是并发监听fd。
  因为在这种情况下，如果某fd的write缓冲区满了，write会立刻返回-1，并且返回EWOULDBLOCK。所以数据的收和发未必可以同时进行，所以对于write操作，我们需要一个buffer来暂存数据。当fd可写时，才可以写入数据。
  对于read一端，同样需要一个buffer，原因是因为，在阻塞IO中，假设双方协定好处理分包问题，对方发过来一个len为4000，然后我们需要调用readn函数确保收到足够的4000字节，这其中由于网络的拥塞，readn内部可能需要调用多次read系统调用。所以这中间需要短暂的等待。
  到了非阻塞IO中，无法再使用readn反复调用read，否则就变成了轮询操作。如果数据收不满咋办？需要暂存起来，放到一个buffer中，由用户手工处理信息。
  综上，`非阻塞IO的read和write端都需要buffer`。
  详细的叙述请参考muduo库作者陈硕的[Muduo 设计与实现之一：Buffer 类的设计](http://www.cnblogs.com/solstice/archive/2011/04/17/2018801.html)
   
  
###Buffer的设计
   
  Buffer肯定要有输入和输出，所以我设计buffer的格式如下：
  <a href="http://images.cnitblog.com/blog/669654/201410/231704139022211.png"><img style="background-image: none; border-right-width: 0px; padding-left: 0px; padding-right: 0px; display: inline; border-top-width: 0px; border-bottom-width: 0px; border-left-width: 0px; padding-top: 0px" title="buffer1" border="0" alt="buffer1" src="http://images.cnitblog.com/blog/669654/201410/231704169499679.png" width="537" height="204" /></a>
  readindex表示要从Buffer中读取数据的起始位置。writeIndex则表示向Buffer中存放数据的起点。`所以writeIndex – readIndex 是Buffer中数据的大小，而end – writeIndex 则表示Buffer中剩余的空间`。
  注意，`每当buffer中的数据读完时，我们便重置指针，readIndex = writeIndex = begin`。
   
  
###使用非阻塞IO编写回射服务器的客户端
   
  这里的逻辑是：用户从stdin输入数据，然后发给sockfd，随后从sockfd接收数据，输出给stdout。
  所以这里我们要监听四次，stdin的读事件，sockfd的读和写事件，stdout的写事件。
  注意这里需要两个buffer，`因为存在两个数据流`：
     stdin  -> Buffer –> sockfd
      sockfd –> Buffer –> stdout
   所以我们用到两个缓冲区：
  一个用于向sockfd发送数据。
  <a href="http://images.cnitblog.com/blog/669654/201410/231704192306549.png"><img style="background-image: none; border-right-width: 0px; padding-left: 0px; padding-right: 0px; display: inline; border-top-width: 0px; border-bottom-width: 0px; border-left-width: 0px; padding-top: 0px" title="buffer2" border="0" alt="buffer2" src="http://images.cnitblog.com/blog/669654/201410/231704220432791.png" width="659" height="247" /></a>
  一个用于从sockfd接收数据。
  <a href="http://images.cnitblog.com/blog/669654/201410/231704241995931.png"><img style="background-image: none; border-right-width: 0px; padding-left: 0px; padding-right: 0px; display: inline; border-top-width: 0px; border-bottom-width: 0px; border-left-width: 0px; padding-top: 0px" title="buffer3" border="0" alt="buffer3" src="http://images.cnitblog.com/blog/669654/201410/231704273553900.png" width="645" height="196" /></a>
   
  后续将持续讲解Non-Blocking IO，直到完成完整的客户端和服务端。
  未完待续。

			