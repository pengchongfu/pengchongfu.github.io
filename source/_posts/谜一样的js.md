---
title: 谜一样的js
date: 2016-08-27 13:08:45
tags:
---
转眼工作也已经两个多月了，而博客更是三个多月没更新了，因此来随手写一篇博客

首先在浏览器的console里输入`\u5f6d`并按下回车，可以看到报错`Uncaught ReferenceError: 彭 is not defined`，这和在console输入`a`效果是一样的，`Uncaught ReferenceError: a is not defined`

而`\u5f6d`正是汉字“彭”的unicode值。可以看出，在js中，输入一个字符和输入它的unicode值是等价的。因此，若是在console中输入`'\u5f6d'`并回车，console会输出`"彭"`

这就引发了我的思考，能不能通过在console中不输入汉字，而使其输出汉字呢？而这一想法最初的来源是来自于justjavac的[自我介绍](http://justjavac.com/about.html)，一串谜一般的代码输入console之后竟然能够输出`"http://justjavac.com"`

大概讲的就是js里的各种迷之隐式类型转换，同时也可以参考这篇博客[为什么 ++\[\[\]\]\[+\[\]\]+\[+\[\]\] = 10？](http://justjavac.com/javascript/2012/05/24/can-you-explain-why-10.html)

而里边提出了这样一个问题：是否能够拼出中文字符甚至所有字符串？

于是我也开始了折腾，希望能够把我自己的名字给拼出来。首先，'彭'字的unicode值是'\u5f6d'；因此在console中输入`'\u5f6d'`可以得到输出`"彭"`

这样一看，事情就很简单了，只要想办法将这六个字符拼出来并加在一起，可以得到'彭'字，再依法炮制得到另外两个字就OK了

可是现实总是那么残酷，在console中输入`"'\u5f6'+'d'"`的话，会报错`Uncaught SyntaxError: Invalid Unicode escape sequence`，于是我就放弃了折腾。

直到我看到了这个神一般[网站](http://www.jsfuck.com/)，很轻松的就把我的名字给输出来了。

仔细看了之后发现关键在于：之前的操作都是致力于获得字符，而这个网站不仅获得了字符，还能获得函数。例如`eval`函数的表达形式为`[]["filter"]["constructor"]( CODE )()`

而在看了这一篇[博客](http://www.cnblogs.com/ecalf/archive/2012/09/04/unicode.html)之后，更是恍然大悟

> 在字符串中 \u 后面必须接 4个十六进制字符才是合法的语法，所以不得不转义

因此需要这样的代码才能输出我想要的结果：
`eval('"'+'\\u5f6'+'d'+'"')`

于是接下来就简单了

'u'：`[][+[]]+[]`=`undefined`，`+[]`=`0`因此`u`=`([][+[]]+[])[+[]]`

'5'：`+!+[]`=`+!0`=`+true`=`1`，因此`5`=`+!+[]+!+[]+!+[]+!+[]+!+[]`

'f'：`(([][+[]]+[])[+[]])[+!+[]+!+[]+!+[]+!+[]]`

'6'：`+!+[]+!+[]+!+[]+!+[]+!+[]+!+[]`

'd'：`(([][+[]]+[])[+[]])[+!+[]+!+[]]`

因此，还剩下'"'和'\'需要构造出来。

查看jsfuck的[源码](https://github.com/aemkei/jsfuck/blob/master/jsfuck.js)，可以知道'"'可以通过如下方式得到

`("")["fontcolor"]()[12]`，这是因为`("")["fontcolor"]()`的输出结果为`"<font color="undefined"></font>"`

