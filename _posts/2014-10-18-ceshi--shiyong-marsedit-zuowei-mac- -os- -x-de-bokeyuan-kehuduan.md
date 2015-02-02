---
layout: post
title: 测试-使用MarsEdit作为Mac OS X的博客园客户端
category : C++
tagline: "Supporting tagline"
tags : [C++, Python, Socket, Tornado]
---
测试

<p style="margin: 0px; font-size: 18px; font-family: Menlo; min-height: 21px;"> </p>
<p style="margin: 0px; font-size: 18px; font-family: Menlo; color: #c91b13;">#include <iostream></p>
<p style="margin: 0px; font-size: 18px; font-family: Menlo; color: #c91b13;">#include <functional></p>
<p style="margin: 0px; font-size: 18px; font-family: Menlo; color: #822e0e;">#include <memory></p>
<p style="margin: 0px; font-size: 18px; font-family: Menlo; color: #c32275;">using namespace std;</p>
<p style="margin: 0px; font-size: 18px; font-family: Menlo; min-height: 21px;"> </p>
<p style="margin: 0px; font-size: 18px; font-family: Menlo;">class Test</p>
<p style="margin: 0px; font-size: 18px; font-family: Menlo;">{</p>
<p style="margin: 0px; font-size: 18px; font-family: Menlo; color: #c32275;">public:</p>
<p style="margin: 0px; font-size: 18px; font-family: Menlo;">    Test()</p>
<p style="margin: 0px; font-size: 18px; font-family: Menlo;">    {</p>
<p style="margin: 0px; font-size: 18px; font-family: Menlo;">        cout << "Test " << endl;</p>
<p style="margin: 0px; font-size: 18px; font-family: Menlo;">    }</p>
<p style="margin: 0px; font-size: 18px; font-family: Menlo; min-height: 21px;">    </p>
<p style="margin: 0px; font-size: 18px; font-family: Menlo;">    ~Test()</p>
<p style="margin: 0px; font-size: 18px; font-family: Menlo;">    {</p>
<p style="margin: 0px; font-size: 18px; font-family: Menlo;">        cout << "~Test" << endl;</p>
<p style="margin: 0px; font-size: 18px; font-family: Menlo;">    }</p>
<p style="margin: 0px; font-size: 18px; font-family: Menlo; color: #c32275;">private:</p>
<p style="margin: 0px; font-size: 18px; font-family: Menlo;">    int value_;</p>
<p style="margin: 0px; font-size: 18px; font-family: Menlo;">};</p>
<p style="margin: 0px; font-size: 18px; font-family: Menlo; min-height: 21px;"> </p>
<p style="margin: 0px; font-size: 18px; font-family: Menlo; min-height: 21px;"> </p>
<p style="margin: 0px; font-size: 18px; font-family: Menlo;">int main(int argc, const char * argv[])</p>
<p style="margin: 0px; font-size: 18px; font-family: Menlo;">{</p>
<p style="margin: 0px; font-size: 18px; font-family: Menlo; min-height: 21px;"> </p>
<p style="margin: 0px; font-size: 18px; font-family: Menlo; color: #1d9421;">    // insert code here...</p>
<p style="margin: 0px; font-size: 18px; font-family: Menlo; color: #c91b13;">    std::cout << "Hello, World!\n";</p>
<p style="margin: 0px; font-size: 18px; font-family: Menlo;">    std::shared_ptr<Test> ptr(new Test);</p>
<p style="margin: 0px; font-size: 18px; font-family: Menlo; min-height: 21px;">    </p>
<p style="margin: 0px; font-size: 18px; font-family: Menlo; min-height: 21px;">    </p>
<p style="margin: 0px; font-size: 18px; font-family: Menlo;">    return 0;</p>
<p style="margin: 0px; font-size: 18px; font-family: Menlo;">}</p>
			