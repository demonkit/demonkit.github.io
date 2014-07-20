---
layout: post
title: WSGI学习记录
tags : [python, framework, web, wsgi]
---

#python程序员选择web框架

这篇文章[WSGI ­ Gateway or Glue?](http://osdcpapers.cgpublisher.com/product/pub.84/prod.21/m.1?)提到了WSGI诞生的一些机缘。总的来说我认为就是python的web框架选择太多，同时支持的web服务器也太多。这两者之间的任意一个组合也许都会有特定的设计方式与交互方式。这对于程序的可复用性、移植性都是不好的选择。

于是python社区的一些志愿者们提出了使用同一规范来规范web服务器和web应用程序之间的交互方式，并定义出其交互的接口。这些接口借鉴自java的servlet api。java的servlet api规范化了java的web程序和web服务器之间的交互接口。任何遵从这一规范的web应用程序都可以运行在任意一个遵守这一规范的web服务器之上。于是开发人员就被解放出来，他们不必再关心所选的框架与服务器之间的通信接口，而只专注于业务逻辑领域的开发与测试。鉴于此，python社区提出了WSGI接口(python web server gateway interface)。


#什么是WSGI
如上所述，WSGI是python的web应用程序和web服务器之间交互的通用接口定义。

#WSGI的特点
1. 接口定义简单，易于实现WSGI接口的服务器和web应用程序框架。这也是WSGI定义的一个最基本的目标。
2. 它释放了开发人员不必要的工作，提高了开发人员的工作效率，使其可以更加专注于感兴趣的领域。

#WSGI包含的内容
WSGI定义了三种角色：server、application、middleware。server端实现的是接收HTTP请求，调用application端，将HTTP请求转发到application端进行处理并接收处理结果返回给客户浏览器。application端是真正做业务逻辑处理的地方，它根据server端下发的不同请求做不同的响应。middleware是真正体现WSGI灵活性的地方，middleware相当于一个管道：他一层层递进客户端的请求，过滤并处理自己感兴趣的内容最终交给application，并且在响应返回时，一层层递出application的响应，封装自己对于响应的处理，最终交给server。在[pep3333](http://legacy.python.org/dev/peps/pep-3333/)中提到，如果middleware的实现足够稳定并且功能足够多，将会对python在web开发中起到极大的促进作用。

下图是简要说明这三者之间是如何做信息交互的，图片来自[ibm developerworks](http://pic.yupoo.com/xiha211/CmMaSs5G/medish.jpg)。
![WSGI数据流图](http://pic.yupoo.com/xiha211/CmMaSs5G/medish.jpg)

####application
在pep3333中规定application的实现就是一个可以被调用的对象。这种对象可以是python的一个函数，也可以是实现了`__call__`方法的类的一个实例。这个可调用对象需要两个参数：
1. `environ`，python的最基本的字典对象，其中包含了cgi类型的环境变量参数和WSGI自身要求的必须参数。
2. `start_response`，是由server提供的一个回调函数，用以返回经过application处理的HTTP响应的响应码和响应头。

application对象的返回是一个可迭代对象，最简单的就是一个数组。使用可迭代对象的原因是在python处理的响应内容比较多的时候不会卡住。下面是一个简单的application的实现代码：

```python
def application(environ, start_response):
    status = '200 OK'
    output = "Hello, world!"
    response_headers = [('Content-type', 'text/plain')]
    start_response(status, response_headers)
    return [output] 
```

####server
pep3333中对于server应该实现什么样的方法做了规定。鉴于WSGI的设计借鉴了java servlet api，我们可以把server理解为一个给application提供环境的一个容器（container）。它注入了一个application需要的两个参数。我对于server端的具体实现细节关注的不多。如下是pep3333中提供的一个python实现的server例子。

```python
import os, sys

enc, esc = sys.getfilesystemencoding(), 'surrogateescape'

def unicode_to_wsgi(u):
    # Convert an environment variable to a WSGI "bytes-as-unicode" string
    return u.encode(enc, esc).decode('iso-8859-1')

def wsgi_to_bytes(s):
    return s.encode('iso-8859-1')

def run_with_cgi(application):
    environ = {k: unicode_to_wsgi(v) for k,v in os.environ.items()}
    environ['wsgi.input']        = sys.stdin.buffer
    environ['wsgi.errors']       = sys.stderr
    environ['wsgi.version']      = (1, 0)
    environ['wsgi.multithread']  = False
    environ['wsgi.multiprocess'] = True
    environ['wsgi.run_once']     = True
    if environ.get('HTTPS', 'off') in ('on', '1'):
        environ['wsgi.url_scheme'] = 'https'
    else:
        environ['wsgi.url_scheme'] = 'http'
    headers_set = []
    headers_sent = []

    def write(data):
        out = sys.stdout.buffer
        if not headers_set:
             raise AssertionError("write() before start_response()")
        elif not headers_sent:
             # Before the first output, send the stored headers
             status, response_headers = headers_sent[:] = headers_set
             out.write(wsgi_to_bytes('Status: %s\r\n' % status))
             for header in response_headers:
                 out.write(wsgi_to_bytes('%s: %s\r\n' % header))
             out.write(wsgi_to_bytes('\r\n'))
        out.write(data)
        out.flush()

    def start_response(status, response_headers, exc_info=None):
        if exc_info:
            try:
                if headers_sent:
                    # Re-raise original exception if headers sent
                    raise exc_info[1].with_traceback(exc_info[2])
            finally:
                exc_info = None     # avoid dangling circular ref
        elif headers_set:
            raise AssertionError("Headers already set!")
        headers_set[:] = [status, response_headers]
        # Note: error checking on the headers should happen here,
        # *after* the headers are set.  That way, if an error
        # occurs, start_response can only be re-called with
        # exc_info set.
        return write
    result = application(environ, start_response)
    try:
        for data in result:
            if data:    # don't send headers until body appears
                write(data)
        if not headers_sent:
            write('')   # send headers now if body was empty
    finally:
        if hasattr(result, 'close'):
            result.close()
```

####middleware
参考pep3333和这篇文章[Introducing WSGI: Python's Secret Web Weapon](http://www.xml.com/pub/a/2006/09/27/introducing-wsgi-pythons-secret-web-weapon.html)的说明，middleware可以实现很多功能，例如：
1. session中间件
2. 用户登录认证的中间件
3. url路由功能
4. 程序执行异常的错误拦截
等等。
这里提供一个简单的url路由功能的代码，这个代码并不是很规范，并能体现middleware的强大之处，建议可以查看一下Bottle和Flask的url路由代码。

```python
def router(environ, start_response):
    path = environ['PATH_INFO']
    if 'hello' in path:
        return application(environ, start_response)
    else:
        return app2(environ, start_response)
```

# django 和uWSGI
django已经支持了wsgi，在每一个项目下都会有一个名为wsgi.py的文件，它是自动生成的，用以定义django项目的application。

uWSGI是一个wsgi服务器，它设计的目标是实现一个最小的wsgi服务器，在uwsgi进程运行时必须指定django提供的wsgi.py的路径。

一般还会在wsgi服务器之前搭建一个web服务器，现在比较流行的是nginx，它转发动态的请求到uWSGI服务器并将响应返回给客户端。另外选择nginx的好处还有它是一个很好的反向代理服务器，它对于静态文件的处理性能也能很好。并且django推荐的部署方式也是使用web服务器来处理静态文件。

# 后续
记录一下我理解的cgi、fastcgi、wsgi的区别。
1. cgi，通用网关接口，用以处理动态程序，一般web服务器接收请求，封装需要的请求参数，将cgi请求转发给cgi应用程序。这时一般操作系统会fork一个进程来处理请求，在响应结束后，销毁进程。
2. fastcgi，上述的cgi程序在每一个请求中都会fork一个进程，对于系统资源消耗过大，大量频繁的请求会占用更多的资源。fastcgi在cgi基础上，持久化一系列的常驻进程，每一个请求发过来，fastcgi都会生成一个线程(??)进行处理，大大的减少了fork进程的资源开销。
3. wsgi是专门针对python的web开发提出的一套接口说明。

# 参考
1. http://linluxiang.iteye.com/blog/799163
2. http://wsgi.readthedocs.org/en/latest/index.html
3. http://www.xml.com/pub/a/2006/09/27/introducing-wsgi-pythons-secret-web-weapon.html
4. http://blog.kenshinx.me/blog/wsgi-research/
5. http://zh.wikipedia.org/wiki/Web服务器网关接口