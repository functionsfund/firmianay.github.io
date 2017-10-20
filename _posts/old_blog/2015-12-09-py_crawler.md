---
layout: post
title: 简单的python爬虫
category: old_blog
tags: programming
keywords: python, programming
description:
---

> 本文写于 2015.12.09

第一次知道爬虫是某个月黑风高的晚上从盘神口中听说的，感觉好好玩的样子。

网络爬虫被广泛用于互联网搜索引擎或其他类似网站，以获取或更新这些网站的内容和检索方式。它们可以自动采集所有其能够访问到的页面内容，以供搜索引擎做进一步处理，而使得用户能更快的检索到他们需要的信息。

我尝试用Python来写人生中第一个爬虫。

urllib2 是Python的一个获取URLs的库，它以urlopen函数的形式提供了一个非常简单的接口。官方文档

我们来分析一下，用HTTP地址来创建一个Request对象，调用urlopen并传入Request对象，返回一个相关请求response对象，这个应答对象就好像一个文件对象，调用.read()，最后输出。
```
import urllib2
req = urllib2.Request('http://www.baidu.com/')
response = urllib2.urlopen(req)
html = response.read()
print html
```
代码可以优化为下面的样子：
```
import urllib2
response = urllib2.urlopen('http://www.baidu.com/')
html = response.read()
print html
```
抓取后，我们可以打开百度首页查看源码做对比，内容完全一样。好了第一步完成，下面就要想办法从中筛选出第二层的链接了。

我们知道，爬虫在发送http请求获取数据时会在头部附加User-Agent信息，有的网站就通过记录和分析User-Agent信息来挖掘和封锁爬虫。

当然了我们可以通过把爬虫伪装成浏览器，让我们来配置一下User-Agent头域。可以通过Tamper Deta插件得到我们浏览器的User-Agent,当然也可以自己虚构一个。

整个过程就像下面这样：
```
import urllib2
url = 'http://www.baidu.com/'
headers = {'User-Agent':"Mozilla/5.0 (X11; U; Linux i686) Gecko/20071127 Firefox/2.0.0.11"}
req = urllib2.Request(url,headers=headers)
response = urllib2.urlopen(req)
html = response.read()
print html
```
这样我们就有点瞒天过海的感觉了。
