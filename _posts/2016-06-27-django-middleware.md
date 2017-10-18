---
layout: post
title: django中间件
tags:
- django
categories: django
description:  django中间件的介绍
---


[引言]在有些场合，需要对Django处理的每个request都执行某段代码。这类代码可能是在view处理之前修改传入的request，或者记录日志信息以便于调试，等等。
这类功能可以用Django的中间件框架来实现，该框架由切入到Django的request/response处理过程中的钩子集合组成。这个轻量级低层次的plug-in系统，能用于全面的修改Django的输入和输出。

## 什么是中间件
中间件组件是遵循特定API规则的简单Python类。在深入到该API规则的正式细节之前，先看一下下面这个非常简单的例子。

高流量的站点通常需要将Django部署在负载平衡proxy之后。
这种方式将带来一些复杂性，其一就是每个request中的远程IP地址(**request.META["REMOTE_IP"]**)将指向该负载平衡proxy，而不是发起这个request的实际IP。
负载平衡proxy处理这个问题的方法在特殊的 X-Forwarded-For 中设置实际发起请求的IP。

~~~python
class SetRemoteAddrFromForwardedFor(object):
    def process_request(self, request):
        try:
            real_ip = request.META['HTTP_X_FORWARDED_FOR']
        except KeyError:
            pass
        else:
            # HTTP_X_FORWARDED_FOR can be a comma-separated list of IPs.
            # Take just the first one.
            real_ip = real_ip.split(",")[0]
            request.META['REMOTE_ADDR'] = real_ip
~~~
一旦安装了该中间件(参见下一节)，每个request中的 X-Forwarded-For 值都会被自动插入到 request.META['REMOTE_ADDR'] 中。
这样，Django应用就不需要关心自己是否位于负载平衡proxy之后；简单读取 request.META['REMOTE_ADDR'] 的方式在是否有proxy的情形下都将正常工作。
实际上，为针对这个非常常见的情形，Django已将该中间件内置。它位于 django.middleware.http 中, 下一节将给出这个中间件相关的更多细节。


## 安装中间件


如果按顺序阅读本书，应当已经看到涉及到中间件安装的多个示例,因为前面章节的许多例子都需要某些特定的中间件。出于完整性考虑，下面介绍如何安装中间件。

~~~python
MIDDLEWARE_CLASSES = (
    'django.middleware.common.CommonMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.middleware.doc.XViewMiddleware'
)
~~~

Django项目的安装并不强制要求任何中间件，如果你愿意， MIDDLEWARE_CLASSES 可以为空。但我们建议启用 CommonMiddleware ，稍后做出解释。

这里中间件出现的顺序非常重要。
<ul>
<li>在request和view的处理阶段，Django按照 MIDDLEWARE_CLASSES 中出现的顺序来应用中间件;</li>
<li>而在response和异常处理阶段，Django则按逆序来调用它们。</li>
</ul>

也就是说，Django将 MIDDLEWARE_CLASSES 视为view函数外层的顺序包装子：`在request阶段按顺序从上到下穿过，而在response则反过来`。


## 中间件方法

现在，我们已经知道什么是中间件和怎么安装它，下面将介绍中间件类中可以定义的所有方法。

### Initializer: __init__(self)
在中间件类中， __init__() 方法用于执行系统范围的设置。

出于性能的考虑，每个已启用的中间件在每个服务器进程中只初始化 一 次。
也就是说 __init__() 仅在服务进程启动的时候调用，而在针对单个request处理时并不执行。
对一个middleware而言，定义 __init__() 方法的通常原因是检查自身的必要性。
如果 __init__() 抛出异常 django.core.exceptions.MiddlewareNotUsed ,则Django将从middleware栈中移出该middleware。
可以用这个机制来检查middleware依赖的软件是否存在、服务是否运行于调试模式、以及任何其它环境因素。
在中间件中定义 __init__() 方法时，除了标准的 self 参数之外，不应定义任何其它参数。


### Request预处理函数: process_request(self, request)

这个方法的调用时机`在Django接收到request之后，但仍未解析URL以确定应当运行的view之前`。Django向它传入相应的 HttpRequest 对象，以便在方法中修改。
process_request() 应当返回 None 或 HttpResponse 对象.

<li>如果返回 HttpResponse 对象, Django 将不再执行任何其它的中间件(而无视其种类)以及相应的view。 Django将立即返回该 HttpResponse.</li>

### View预处理函数: process_view(self, request, view, args, kwargs)

这个方法的调用时机`在Django执行完request预处理函数并确定待执行的view之后，但在view函数实际执行之前`。

表15-1列出了传入到这个View预处理函数的参数。

如同 process_request() , process_view() 应当返回 None 或 HttpResponse 对象。
<ul>
<li>如果返回 None , Django将继续处理这个 request ,执行后续的中间件， 然后调用相应的view.</li>
<li>如果返回 HttpResponse 对象, Django 将不再执行 任何 其它的中间件(不论种类)以及相应的view. Django将立即返回该 HttpResponse .</li>
</ul>

### Response后处理函数: process_response(self, request, response)

这个方法的调用时机`在Django执行view函数并生成response之后`。这里，该处理器就能修改response的内容；
一个常见的用途是内容压缩，如gzip所请求的HTML页面。

不同可能返回 None 的request和view预处理函数, process_response() 必须 返回 HttpResponse 对象. 这个response对象可以是传入函数的那一个原始对象(通常已被修改)，也可以是全新生成的。


### Exception后处理函数: process_exception(self, request, exception)

这个方法只有`在request处理过程中出了问题并且view函数抛出了一个未捕获的异常时`才会被调用。这个钩子可以用来发送错误通知，将现场相关信息输出到日志文件, 或者甚至尝试从错误中自动恢复。


process_exception() 应当返回 None 或 HttpResponse 对象.
<ul>
<li>如果返回 None , Django将用框架内置的异常处理机制继续处理相应request。</li>
<li>如果返回 HttpResponse 对象, Django 将使用该response对象，而短路框架内置的异常处理机制。</li>
</ul>
