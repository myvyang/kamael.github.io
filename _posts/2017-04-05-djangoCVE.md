---
layout: post
title: "django两个漏洞的补丁分析"
categories: notes
tags: django CVE
---

### CVE-2017-7234 

URL跳转漏洞。

补丁代码

```
 // https://github.com/django/django/commit/2a9f6ef71b8e23fd267ee2be1be26dde8ab67037

 -    path = posixpath.normpath(unquote(path))
 -    path = path.lstrip('/')
 -    newpath = ''
 -    for part in path.split('/'):
 -        if not part:
 -            # Strip empty path components.
 -            continue
 -        drive, part = os.path.splitdrive(part) # 重点
              ...
 -        newpath = os.path.join(newpath, part).replace('\\', '/')
 -    if newpath and path != newpath:
 -        return HttpResponseRedirect(newpath)
```

看下splitdrive的实现：

```
// https://svn.python.org/projects/python/trunk/Lib/ntpath.py

# Split a path in a drive specification (a drive letter followed by a
# colon) and the path specification.
# It is always true that drivespec + pathspec == p
def splitdrive(p):
    """Split a pathname into drive and path specifiers. Returns a 2-tuple
"(drive,path)";  either part may be empty"""
    if p[1:2] == ':':
        return p[0:2], p[2:]
    return '', p
```

`http://xxx`是可以绕过的，newpath保持原样，直接进入`HttpResponseRedirect`。

`HttpResponseRedirect`的说明：

```
class HttpResponseRedirect
The first argument to the constructor is required – the path to redirect to. This can be a fully qualified URL (e.g. 'https://www.yahoo.com/search/'), an absolute path with no domain (e.g. '/search/'), or even a relative path (e.g. 'search/'). In that last case, the client browser will reconstruct the full URL itself according to the current path. See HttpResponse for other optional constructor arguments. Note that this returns an HTTP status code 302.
```

因此POC就是找到一个静态目录，然后在该目录下输入

```
http://vulnerable.site/static/http://evil.com
```

将跳转到恶意网站。

### CVE-2017-7233

官方说在A标签内可导致XSS。

bugtracker：https://code.djangoproject.com/ticket/27912

试下bugtracker中的例子：

```
In [4]: from urllib2 import urlparse

In [6]: urlparse.urlparse("http:999999999")
Out[6]: ParseResult(scheme='http', netloc='', path='999999999', params='', query='', fragment='')

In [7]: urlparse.urlparse("ftp:999999999")
Out[7]: ParseResult(scheme='', netloc='', path='ftp:999999999', params='', query='', fragment='')

In [8]: urlparse.urlparse("https:999999999")
Out[8]: ParseResult(scheme='', netloc='', path='https:999999999', params='', query='', fragment='')
```

可见应该是原本的urlparse实现并没有正确的解析URL，导致获取到的url元组存在问题。


补丁代码

```
 +# Copied from urllib.parse.urlparse() but uses fixed urlsplit() function.
 +def _urlparse(url, scheme='', allow_fragments=True):
      ...

 +# Copied from urllib.parse.urlsplit() with
 +# https://github.com/python/cpython/pull/661 applied.
 +def _urlsplit(url, scheme='', allow_fragments=True):
      ...

  def _is_safe_url(url, host):
      ...
 -    url_info = urlparse(url)
 +    url_info = _urlparse(url)
      if not url_info.netloc and url_info.scheme:
          # url_info.netloc为空而url_info.scheme不空则返回False
          return False
      if unicodedata.category(url[0])[0] == 'C':
          # 控制字符开头则返回False
          return False
      return ((not url_info.netloc or url_info.netloc == host) and
             (not url_info.scheme or url_info.scheme in ['http', 'https']))
```

补丁替换了使用的urlparse实现，在更新的实现里修复了上面示例中存在的问题。同时将正确的实现提交给了python官方：https://github.com/python/cpython/pull/661 (PR提交者看github信息是django的开发人员)。查看PR里的信息可知，urlparse的这个实现问题在2003年就存在，这么多年一致延续下来未曾改正(不过这次确实改掉了)，django躺枪了。

回到django的代码。特定的输入(例如https:999999999)导致`url_info.netloc`和`url_info.scheme`都为空，因此`_is_safe_url`将返回True。`_is_safe_url`返回True的本意为输入的url的scheme为http或https或空。但是通过特定的构造，我们"可能可以"伪造一个scheme为javascript的url通过校验。

看下原本的urlsplit的代码：

```
def urlsplit(url, scheme='', allow_fragments=True):
  ...
  i = url.find(':')
    if i > 0:
      ...
    else:
      # make sure "url" is not actually a port number (in which case
      # "scheme" is really part of the path)
      rest = url[i+1:]
      if not rest or any(c not in '0123456789' for c in rest):
        # not a port number
        scheme, url = url[:i].lower(), rest
  ...
```

因此我们可以构造"xxx:数字"这样的URL，其中xxx为任意字母。

固然我们可以伪造javascript伪协议，但是XSS的代码部分只能为数字(`c not in '0123456789' for c in rest`)，这样是无法构造出一个有效的XSS的。而django的公告里写的"Also, if a developer relies on ``is_safe_url()`` to provide safe redirect targets and puts such a URL into a link, they could suffer from an XSS attack."，可能存在被我忽略掉的姿势？又或者开发人员只是做了最保险的警告，而没有做深究？









