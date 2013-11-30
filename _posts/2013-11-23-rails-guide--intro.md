---
layout: post
title: Rails Guide - 序言
date: 2013-11-23 16:27:31
disqus: y
---

## 序言

大鱼旅行的每个开发者都被要求是full stack programmer, 从前端的html、css和js到应用层面的套页面、写逻辑，到底层的数据库建表、索引原理、全文搜索、NoSQL，都要求大家有一定的掌握。

当然，这些内容不可能每个工程师都对他们深入理解，我们期望每个工程师都能了解所有内容，然后在某个领域深入理解，成为这个领域的专家。

所以本文也是对这些层面进行一个梳理，让新来的同学可以快速熟悉我们的约定，可以快速上手开始输出自己的价值。

### 编码原则

* 你的代码不是给机器读的，而是给人读的
* 过早优化是万恶之源
* 写代码之前之后，思考下面的问题：可复用性如何？扩展性如何？是不是优雅？
* 应对变化。把变化和不变抽离，思考未来可见范围内可见的变化，针对这些变化去编程。不能把所有的变化都去hard code，也不能过于考虑变化的情况，要适度。

### 重构
重构是不断进行的，并不是到了某个阶段才开始重构，每完成一个feature、修完一个bug都可以思考是不是可以重构。

重构可以从下面几方面考虑：

* 可读性：是不是可以使代码更具有可读性？
* 简洁：实现是不是可以更优美？优美代表的是更简单的解决方案，一般来讲也表示代码行数会减少* 
* 复用：代码块是不是抽取，以便更好的复用？
* 扩展：代码块是不是有利于扩展？针对将来可能发生的需求变化能否优雅的应对？
* 封装：是不是已经最小化了你的接口？是不是某些代码应该隐藏其实现？

更多细节可以看重构的相关章节。

### 开发工具篇
vim被称作是“编辑器之神”，是大鱼工程师的首选。使用vim的理由包括：

* vim有各种插件，支持很多种语言；
* 本质上来讲，vim是可编程的，所以可以定制成满足每个人的口味；
* vim一旦熟悉，手指是有记忆功能的，开发效率会提高；
* vim的学习曲线比较陡峭，但是值得去学习；

当然，如果您使用sublime或者textmate，也是可以的，这个不强求。

大鱼要求的最基础的格式是：取消tab，用两个空格代替；同时缩进是2个空格；

vim中的配置：

    set ts=2
    set sw=2
    set expandtab

推荐的几个vim插件如下：

    The-NERD-tree 文件树      
    supertab 自动补全
    vim-snipmate 代码片段自动补全
    ctrlp.vim 文件搜索
    vim-ragtag 标签关闭    
    vim-surround 字符围绕，比如自动添加标签
    vim-fakeclip mac下和系统共享剪切板 
    nerdcommenter 快速注释代码       
    vim-bufsurf  buf快速浏览
    vim-ruby ruby支持
    vim-rails rails支持 

我个人的vimrc以及vim文件夹存放在github上：
    
    <http://github.com/zamia/vimrc>

### 系统架构篇
一个典型的rails stack是这样子的：

* 应用开发: ruby on rails
* 数据库：mysql （可选方案 mongodb、postgres）
* 缓存：memcached (可选方案 redis)
* 全文搜索：solr

其他也一些常用的技术：

* 前端js框架: jquery 
* 前端css框架: bootstrap
* 异步队列：delayed_job / resque
* geo搜索：mongodb / solr 4.0以上版本

关键gem：

* sunspot: 全文搜索
* unicorn: 稳定的多进程web server
* paperclip: 附件上传
* rmagick: 图片处理
* will_paginate: 分页
* cancan: 权限控制
* nokogirl: xml/html解析器
* hz2py: 汉字简繁体、汉字拼音转换
* delayed_job: 异步执行
* flash_cookie_session: 支持带验证的flash文件上传
* rspec: 测试框架
* factory_girl: 自动测试数据构造
* sprite_factory: 小图片拼接
* draper: decorator设计模式实现

还有其他一些gem，可以根据实际的使用需求再去寻找，一般判断gem的标准是：

* 项目的活跃度：有多少人watch和fork？作者更新频率如何？
* 项目的成熟度：发布了多久？经历了多少个版本号？

