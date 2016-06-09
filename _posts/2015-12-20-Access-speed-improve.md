---
title: 博客加载的前端优化
layout: post
categories: 填坑记录
---
这个博客开通之后，一直在调整界面，更新内容，没有关注过访问速度，今天终于得空好好的把访问速度优化一下，查看了一下博客主页，在我家的网络环境下，目录页基本加载速度在5秒左右，文章详情页在7-8秒左右，这样的体验是在是太差了，然后使用了下浏览器抓包工具把加载时间查看了下时间消耗最大的是下面几个：

* 一个谷歌的字体样式文件（ http://fonts.googleapis.com/css?family=Open+Sans:400,700）  200ms
* 数学公式的js文件（https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML）  这个压根打不开，SSL被墙了
* /stylesheets/normalize.css   站内文件400ms左右
* /stylesheets/stylesheet.css   站内文件400ms左右
* /stylesheets/github-light.css 站内文件400ms左右
* /stylesheets/normalize.css 站内文件400ms左右
* /stylesheets/toc.css 站内文件400ms左右
* favicon.ico（这个文件404了） 404文件500ms

看到上面的文件消耗时间，基本优化思路就出来了：

* 不加载谷歌的字体文件，把文件内容放到stylesheet.css里面
* 把所有站内CSS文件放到一个文件stylesheet.css里面，然后把该文件进行压缩
* 把数学公式的js从mathjax的git上下载下来，传到git hub pages上，防止它的cdn无法访问
* 添加favicon.ico文件，防止CSS，把它的耗时降到200ms


经过上述的优化，主页的加载速度控制在1秒内，文章页的加载速度控制在1.5秒。