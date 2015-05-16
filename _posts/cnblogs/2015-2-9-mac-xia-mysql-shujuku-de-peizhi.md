---
layout: post
title: mac下mysql数据库的配置
category : C++
tagline: "Supporting tagline"
tags : [C++, Python, Socket, Tornado]
---
这里记录一下。
  之前在mac下使用brew install mysql安装，但是安装完成后发现密码不好修改，上网搜了下发现mac下使用命令行安装mysql确实存在很多问题，这一点确实远不如Ubuntu方便。
  网上建议的是去mysql官网下载，这里采用官方版本。
   
  1.去mysql官网下载
  <a title="http://dev.mysql.com/downloads/mysql/" href="http://dev.mysql.com/downloads/mysql/">http://dev.mysql.com/downloads/mysql/</a> 在这个页面下载，然后安装拖动即可。
  如图：
  <a href="http://images.cnitblog.com/blog/669654/201502/092130139334573.png"><img style="border-bottom: 0px; border-left: 0px; display: inline; border-top: 0px; border-right: 0px" title="屏幕快照 2015-02-09 21.22.37" border="0" alt="屏幕快照 2015-02-09 21.22.37" src="http://images.cnitblog.com/blog/669654/201502/092130153702099.png" width="951" height="470" /></a> 
  2.启动mysql
  点击 偏好设置 在最下方可以看到mysql的配置。
  如下图：
  <a href="http://images.cnitblog.com/blog/669654/201502/092130163869156.png"><img style="border-bottom: 0px; border-left: 0px; display: inline; border-top: 0px; border-right: 0px" title="屏幕快照 2015-02-09 21.22.55" border="0" alt="屏幕快照 2015-02-09 21.22.55" src="http://images.cnitblog.com/blog/669654/201502/092130173709754.png" width="639" height="660" /></a> 
   
  以及：
  <a href="http://images.cnitblog.com/blog/669654/201502/092130185898282.png"><img style="border-bottom: 0px; border-left: 0px; display: inline; border-top: 0px; border-right: 0px" title="屏幕快照 2015-02-09 21.23.03" border="0" alt="屏幕快照 2015-02-09 21.23.03" src="http://images.cnitblog.com/blog/669654/201502/092130197769352.png" width="572" height="271" /></a> 
   
  3.在命令行启动mysql
  上面安装完毕之后，我们是无法再命令行启动mysql的。
  可以在shell中输入：
     export PATH=$PATH:/usr/local/mysql/bin
   为了持久有效，我们将其保存在~/.zshrc中。（本人使用的是zsh）
  4.重设密码
  mysql当前版本默认密码本人也不知，有的文章说密码也是root，我试了下不可行。
  我使用的mysql版本是：
     Server version: 5.6.23 MySQL Community Server (GPL)
   mac版本是最新的Yosemite 10.10.2
  这里重设密码采用：
     /usr/local/mysql/bin/mysqladmin -u root password 新密码
   5.可以安装Mysql workbench，也是从官网下载。
  至此mac下mysql的配置就基本完成了。

			