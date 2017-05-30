---
title: webpack 搭建静态网站
date: 2017-05-29 14:01:15
tags:
- webpack
- vue
- 前端
---

最近需要给朋友搭建一个个人网站，会有比较多的图片用于展示。

由于网站图片和内容可能会需要不时更新，直觉上用现有的个人网站框架例如 WordPress 之类的会方便一下。但是这样一来各种内容、图片都保存在数据库中管理起来不太直接。

而且不管用什么框架前端界面都是需要高度定制的，不可能套用现有主题之类的。既然前端都是需要写的，因此决定先做个静态网站试试看，不好用的话再试试带后端的框架。

本博客其实也是一个静态网站，用的是 hexo。但是考虑到 hexo 是个博客框架，因此似乎不怎么满足我的需求。

尴尬的是我从学前端开始一直就是写的单页应用，完全没用自己搭建过静态网站，因此开始了以下坑爹的探索

### 1. gulp + less + nunjucks

这是公司官网的前端项目所用的框架。

也就是打包工具用 gulp，样式用 less，模板用 nunjucks。

#### gulp

gulp 其实可以看成是一个任务管理器。将你需要执行的任务通过 gulp 一个一个地完成。这些任务其实直接写 node 的脚本其实也是能够完成的，只不过用 gulp 更方便了。

例如打包 less，告诉 gulp 入口文件和出口文件就可以了。

#### less

其实这些 css 的预编译语言学起来都挺快的，选个自己喜欢的就好。。。

#### nunjucks

正如官网所说，nunjucks 类似于 jinja2 的一个 js 实现。整个逻辑也是和 django 中的相似。

通过 gulp 从文件中获取元数据 --> 将元数据传给 nunjucks --> nunjucks 按照模板的规则生成 html --> 保存成文件

整个过程还是很直观的。不过需要自己处理一下 html、css 和 js 之间的引用关系，耦合得比较小

### 2. webpack + less + vue

这是参考了 [reverland](https://github.com/reverland) 大佬的 [博客项目](https://github.com/reverland/blog-next) 以及[这一篇](http://reverland.org/javascript/2016/07/24/exploring-webpack/)关于 webpack 的介绍。

#### webpack

看了之后还是受益匪浅的。这个博客项目可以看成是使用 webpack 实现了一个简化版的 hexo。

- 对 markdown 的处理通过自定义 loader 来实现
- 对博客信息的提取通过自定义 plugin 来实现（类似于 gulp 的任务）
- 博客具体内容的加载通过 webpack 的 codesplit 功能来实现（简单明了不需要写网络请求 api）

#### vue

也就是说其实这里的 vue 起到了 nunjucks 的作用：模板渲染。但是这个渲染过程是在客户端进行的。当然不仅仅是模板渲染，路由管理、状态管理之类的也都是在这里进行处理。

这貌似是一种可行的方法。通过 webpack 的 plugin 完美地取代了 gulp 的功能。只是由于本质上还是一个单页应用，因此 seo 之类的支持起来还是比较困难的。

### 3. nuxt

所以方案 2 貌似是不太可行了。然而作为一个喜新厌旧的人我并不想去学 gulp。也不想去用 nunjucks 的模板，毕竟我用的最多的是 jsx。。。

于是找到了服务器端渲染的方案。

在没有了解过服务器端渲染之前，我一直以为这是一个鸡肋的功能。然而在看了 vue 服务器端渲染框架 nuxt 的文档之后，我才发现这是一个很强大的功能。不仅能够在服务器端将首屏渲染成 html 减小网络请求数以及客户端计算资源，还能够带有 vue 这类 mvvm 框架的优点。最为关键的是 nuxt 还能实现静态化。

并且，我们还能够用 jsx 这种熟悉的方式书写模板。不需要再去学习 nunjucks。与方案 1 比起来除了网页请求里多了 nuxt 自带的关于 vue 的几个库以外，没有任何其他的缺点。因此我决定用 nuxt 写一个简单的博客试试看。

### nuxt blog

最终方案为 webpack + nuxt + css modules + vue

项目地址为 [https://github.com/pengchongfu/nuxt-blog](https://github.com/pengchongfu/nuxt-blog)。

可以一个一个 commit 地看是如何构建起来的。然而构建过程中还是有很多坑的。。

通过这个项目我也真正明白了前端开发的真理：

`编码两分钟，配置两小时`


