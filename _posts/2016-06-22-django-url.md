---
layout: post
title: django url 备忘
tags:
    - django
categories: django
description:  django的url备忘
---

[引言]利用Django开发网站，可以设计出非常优美的url规则，如果url的匹配规则（包含正则表达式）组织得比较好，view的结构就会比较清晰，比较容易维护。

### 最简单的形式

~~~
from django.conf.urls import patterns, url
urlpatterns = patterns('',
    url(r'^articles/2003/$', 'news.views.special_case_2003'),
    url(r'^articles/(\d{4})/$', 'news.views.year_archive'),
    url(r'^articles/(\d{4})/(\d{2})/$', 'news.views.month_archive'),
    url(r'^articles/(\d{4})/(\d{2})/(\d+)/$', 'news.views.article_detail'),
)
~~~
其中，正则表达式中组匹配出来的结果可以作为positional parameters传递给view.
如果url是www.yourdomain/articles/2005/，则会匹配第二条规则，执行news.views.year_archive('2005').

**注意点**
<ul>
<li>域名部分会被过滤掉</li>
<li>articles的前面不需要添加/，因为前序url的末尾一定会有/ </li>
<li>任何组匹配的变量，都会议字符串的形式传递给view, 虽然通过(\d{4})匹配出了2005，但2005任然会被当做字符串传递给year_archive </li>
</ul>

### 利用named group来传递参数

可以通过以下形式为特定的组指定一个名称.

~~~
from django.conf.urls import patterns, url
urlpatterns = patterns('',
    url(r'^articles/2003/$', 'news.views.special_case_2003'),
    url(r'^articles/(?P<year>\d{4})/$', 'news.views.year_archive'),
    url(r'^articles/(?P<year>\d{4})/(?P<month>\d{2})/$', 'news.views.month_archive'),
    url(r'^articles/(?P<year>\d{4})/(?P<month>\d{2})/(?P<day>\d{2})/$', 'news.views.article_detail'),
)
~~~

这样的话，组的匹配结果会通过keyword parameters的形式传递给view.例如year_archive(year='2005')


利用named group可以为view指定一个默认参数来匹配多条规则。

~~~
# URLconf
from django.conf.urls import patterns, url

urlpatterns = patterns('',
    url(r'^blog/$', 'blog.views.page'),
    url(r'^blog/page(?P<num>\d+)/$', 'blog.views.page'),
)

# View (in blog/views.py)
def page(request, num="1"):
    # Output the appropriate page of blog entries, according to num.
    pass
~~~

### 指定view前缀(提取公因式)

patterns函数的第一个参数即是view的前缀

~~~
from django.conf.urls import patterns, url

urlpatterns = patterns('news.views',
    url(r'^articles/(\d{4})/$', 'year_archive'),
    url(r'^articles/(\d{4})/(\d{2})/$', 'month_archive'),
    url(r'^articles/(\d{4})/(\d{2})/(\d+)/$', 'article_detail'),
)
~~~

### 指定多个view前缀

~~~
from django.conf.urls import patterns, url
urlpatterns = patterns('myapp.views',
    url(r'^$', 'app_index'),
    url(r'^(?P<year>\d{4})/(?P<month>[a-z]{3})/$','month_display'),
)
urlpatterns += patterns('weblog.views',
    url(r'^tag/(?P<tag>\w+)/$', 'tag'),
)
~~~

### include其它匹配模块

~~~
from django.conf.urls import include, patterns, url

urlpatterns = patterns('',
    # ... snip ...
    url(r'^comments/', include('django.contrib.comments.urls')),
    url(r'^community/', include('django_website.aggregator.urls')),
    url(r'^contact/', include('django_website.contact.urls')),
    # ... snip ...
)
~~~

当然也可以直接include其它patterns

~~~
from django.conf.urls import include, patterns, url

extra_patterns = patterns('',
    url(r'^reports/(?P<id>\d+)/$', 'credit.views.report'),
    url(r'^charge/$', 'credit.views.charge'),
)

urlpatterns = patterns('',
    url(r'^$', 'apps.main.views.homepage'),
    url(r'^help/', include('apps.help.urls')),
    url(r'^credit/', include(extra_patterns)),
)
~~~

### 为view函数传递额外参数

~~~
from django.conf.urls import patterns, url

urlpatterns = patterns('blog.views',
    url(r'^blog/(?P<year>\d{4})/$', 'year_archive', {'foo': 'bar'}),
)
~~~

### 直接使用view函数

~~~
from django.conf.urls import patterns, url
from mysite.views import archive, about, contact

urlpatterns = patterns('',
    url(r'^archive/$', archive),
    url(r'^about/$', about),
    url(r'^contact/$', contact),
)
~~~