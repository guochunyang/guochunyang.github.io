---
layout: post
title: 从博客园迁移博客到jekyll
category : C++ CAS
tagline: "Supporting tagline"
tags : [C++, Linux]
---

上周花了两天时间，把博客园的博客迁移到了jekyll。  
下面讲述一下迁移的过程。

###导出博客园的文章

博客园官方设置的后台，有一个导出RSS文章备份的功能，有了它，我们就不必写爬虫去抓取自己的文章

###解析XML
导出的XML文件中，每篇文章都是一个item结点，我参考了[用 ElementTree 在 Python 中解析 XML](http://pycoders-weekly-chinese.readthedocs.org/en/latest/issue6/processing-xml-in-python-with-element-tree.html)这篇文章

我们依次遍历所有item结点，取出文章的标题、正文和生成时间。这里注意要把时间转换一下格式。
我这里采用的是`datetime.strptime(item.text, "%a, %d %b %Y %H:%M:%S %Z")`

###解析正文
重头戏来了，博客园导出的文章其实是一些html，我的工作就是将这些html还原成原来的文本格式，最好再加上一些markdown语法。

我们观察下面的代码：

```html
<p>然后这样使用：</p>

<div style="border-bottom: #cccccc 1px solid; border-left: #cccccc 1px solid; padding-bottom: 5px; background-color: #f5f5f5; padding-left: 5px; padding-right: 5px; border-top: #cccccc 1px solid; border-right: #cccccc 1px solid; padding-top: 5px" class="cnblogs_code">
  <pre>@count_time(<span style="color: #800000">&quot;</span><span style="color: #800000">foobar</span><span style="color: #800000">&quot;</span><span style="color: #000000">)
</span><span style="color: #0000ff">def</span><span style="color: #000000"> test():
    sleep(</span>0.5<span style="color: #000000">)

@count_time(</span><span style="color: #800000">&quot;</span><span style="color: #800000">测试消息</span><span style="color: #800000">&quot;</span><span style="color: #000000">)
</span><span style="color: #0000ff">def</span><span style="color: #000000"> test2():
    sleep(</span>0.7<span style="color: #000000">)

</span><span style="color: #0000ff">if</span> <span style="color: #800080">__name__</span> == <span style="color: #800000">'</span><span style="color: #800000">__main__</span><span style="color: #800000">'</span><span style="color: #000000">:
    test()
    test2()</span></pre>
</div>

<p>结果如下：</p>
```

经过仔细观察，我得出以下的结论：

	1.每个段落使用的都是p标签
	2.之前写文章加上颜色的部分，都用font进行了修饰
	3.所有的代码都用div pre进行修饰，为了保持代码的格式，这对Python更为重要，只需要将span标签去掉即可。
	4.html代码中原本的html，都进行了转义，改成了&nbsp; &lt;等符号，这部分需要还原
	5.有个p标签内部使用了整段的加粗，这些应该当做子标题处理


所以下面的工作主要就是正则表达式了，经过各种实验，我采用了以下的正则：
	
	1.'<div.*>\s*?<pre>(.*?[\s\S]*?)</pre>\s*?</div>' 解析代码
	2.'<span.*?>(.*?[\s\S]*?)</span>' 过滤span代码
	3.'<p>(.*?[\s\S]*?)</p>' 解析p标签，然后根绝是否满足 '<strong><font.*?>(.*?[\s\S]*?)</font></strong>' 或者 '<font.*?><strong>(.*?[\s\S]*?)</strong></font>'决定是否进行整段加粗，处理成子标题
	4.<a\shref="(.*?)"\starget="_blank">(.*?)</a>处理链接

还有其他的可以根据实际效果去矫正。

###代码

完整的代码在[cnblogs_to_jekyll](https://github.com/guochunyang/cnblogs_to_jekyll)
