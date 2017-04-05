---
layout: post
title: "django两个漏洞的补丁分析"
categories: notes
tags: django CVE
---

### CVE-2017-7234 

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
 -        head, part = os.path.split(part)
 -        if part in (os.curdir, os.pardir):
 -            # Strip '.' and '..' in path.
 -            continue
 -        newpath = os.path.join(newpath, part).replace('\\', '/')
 -    if newpath and path != newpath:
 -        return HttpResponseRedirect(newpath)
 -    fullpath = os.path.join(document_root, newpath)
 +    path = posixpath.normpath(unquote(path)).lstrip('/')
 +    fullpath = safe_join(document_root, path)
      if os.path.isdir(fullpath):
          if show_indexes:
 -            return directory_index(newpath, fullpath)
 +            return directory_index(path, fullpath)
          raise Http404(_("Directory indexes are not allowed here."))
      if not os.path.exists(fullpath):
          raise Http404(_('"%(path)s" does not exist') % {'path': fullpath})
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

`http://xxx`是可以绕过的。

因此POC就是找到一个静态目录，然后在该目录下输入

```
http://vulnerable.site/static/http://evil.com
```

将跳转到恶意网站。

### CVE-2017-7233

bugtracker：https://code.djangoproject.com/ticket/27912 (没有提供什么有效的信息)

补丁代码

```
+# Copied from urllib.parse.urlparse() but uses fixed urlsplit() function.
 +def _urlparse(url, scheme='', allow_fragments=True):
 +    """Parse a URL into 6 components:
 +    <scheme>://<netloc>/<path>;<params>?<query>#<fragment>
 +    Return a 6-tuple: (scheme, netloc, path, params, query, fragment).
 +    Note that we don't break the components up in smaller bits
 +    (e.g. netloc is a single string) and we don't expand % escapes."""
 +    if _coerce_args:
 +        url, scheme, _coerce_result = _coerce_args(url, scheme)
 +    splitresult = _urlsplit(url, scheme, allow_fragments)
 +    scheme, netloc, url, query, fragment = splitresult
 +    if scheme in uses_params and ';' in url:
 +        url, params = _splitparams(url)
 +    else:
 +        params = ''
 +    result = ParseResult(scheme, netloc, url, params, query, fragment)
 +    return _coerce_result(result) if _coerce_args else result
 +
 +
 +# Copied from urllib.parse.urlsplit() with
 +# https://github.com/python/cpython/pull/661 applied.
 +def _urlsplit(url, scheme='', allow_fragments=True):
 +    """Parse a URL into 5 components:
 +    <scheme>://<netloc>/<path>?<query>#<fragment>
 +    Return a 5-tuple: (scheme, netloc, path, query, fragment).
 +    Note that we don't break the components up in smaller bits
 +    (e.g. netloc is a single string) and we don't expand % escapes."""
 +    if _coerce_args:
 +        url, scheme, _coerce_result = _coerce_args(url, scheme)
 +    allow_fragments = bool(allow_fragments)
 +    netloc = query = fragment = ''
 +    i = url.find(':')
 +    if i > 0:
 +        for c in url[:i]:
 +            if c not in scheme_chars:
 +                break
 +        else:
 +            scheme, url = url[:i].lower(), url[i + 1:]
 +
 +    if url[:2] == '//':
 +        netloc, url = _splitnetloc(url, 2)
 +        if (('[' in netloc and ']' not in netloc) or
 +                (']' in netloc and '[' not in netloc)):
 +            raise ValueError("Invalid IPv6 URL")
 +    if allow_fragments and '#' in url:
 +        url, fragment = url.split('#', 1)
 +    if '?' in url:
 +        url, query = url.split('?', 1)
 +    v = SplitResult(scheme, netloc, url, query, fragment)
 +    return _coerce_result(v) if _coerce_args else v
 +
 +
  def _is_safe_url(url, host):
      # Chrome considers any URL with more than two slashes to be absolute, but
      # urlparse is not so flexible. Treat any url with three slashes as unsafe.
      if url.startswith('///'):
          return False
 -    url_info = urlparse(url)
 +    url_info = _urlparse(url)
      # Forbid URLs like http:///example.com - with a scheme, but without a hostname.
      # In that URL, example.com is not the hostname but, a path component. However,
      # Chrome will still consider example.com to be the hostname, so we must not
      # allow this syntax.
      if not url_info.netloc and url_info.scheme:
          return False
      # Forbid URLs that start with control characters. Some browsers (like
      # Chrome) ignore quite a few control characters at the start of a
      # URL and might consider the URL as scheme relative.
      if unicodedata.category(url[0])[0] == 'C':
          return False
      return ((not url_info.netloc or url_info.netloc == host) and
             (not url_info.scheme or url_info.scheme in ['http', 'https']))
```

可见应该是原本的urlparse实现并没有正确的解析URL，导致获取到的url元组存在问题。

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

看来是urlparse的实现本来就有问题。更准确的说是urlsplit的实现有问题，这个锅是在python的实现上的。通过补丁注释里的链接，https://github.com/python/cpython/pull/661可知，urlparse的实现问题在2003年就存在，这么多年一致延续下来未曾改正(不过这次确实改掉了，PR提交者看github信息是django的开发人员)，django躺枪了。

具体而言，特定的输入(例如https:999999999)导致`url_info.netloc`和`url_info.scheme`都为空，因此`_is_safe_url`将返回True，隐含着输入的url的scheme为http或https。但是通过特定的构造，我们"可能可以"伪造一个scheme为javascript的url通过校验。

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
固然我们可以伪造javascript伪协议，但是XSS的代码部分只能为数字(`c not in '0123456789' for c in rest`)，这样是无法构造出一个有效的XSS的。而django的公告里写的"Also, if a developer relies on ``is_safe_url()`` to provide safe redirect targets and puts such a URL into a link, they could suffer from an XSS attack."，可能存在被我忽略掉的姿势？又或者开发人员只是做了最保险的警告，而没有做深究？









