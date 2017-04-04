---
title: Requests库URL的编码问题
category: Python
---

## 一 前言
之前在使用requests库作爬虫的请求时，发现一个API不遵守国际规范，URL中的某些参数中的不安全字符不进行编码，他的意图是把json字符串发给后台，使用的是GET方法，并把json字符串作为查询参数，而且这个API还是更新数据的。

实际的请求参数如下：`?params={"id":200767, "num": 3}`，对方将请求参数中的`"`进行了编码`%22`，即实际请求的URL参数为`?params={ %22id%22:200767, %22num%22: 3}`， 而`{},:`等字符都应该编码后传输。

构造这个请求参数的字符串是很简单的，但拼接后URL后使用requests.get(url)请求时，requests会自动对url中不安全字符进行编码，包括'{}'等都被编码成`%7B%7D`，API的服务器将拒绝此次请求。

翻了requests的文档，并没有选项设置不编码URL字符串，所以，只有进requests的源码看看内部都做了什么。

## 二 requests.get分析

requests库最简便的用法是

```python
import requests

res = requests.get(url)
```

`requests.get`方法实际是`requests.api.get`方法：

```python
# api.py
from . import sessions

def request(method, url, **kwargs):
    with sessions.Session() as session:
        return session.request(method=method, url=url, **kwargs)

def get(url, params=None, **kwargs):
    kwargs.setdefault('allow_redirects', True)
    return request('get', url, params=params, **kwargs)

# session.py
from .models import Request, PreparedRequest, DEFAULT_REDIRECT_LIMIT

class Session(SessionRedirectMixin):
    def request(self, method, url, params=None, data=None, headers=None,
                cookies=None, files=None, auth=None, timeout=None,
                allow_redirects=True, proxies=None, hooks=None, stream=None,
                verify=None, cert=None, json=None):
        # Create the Request.
        req = Request(method = method.upper(), url = url, headers = headers,
                      files = files, data = data or {}, json = json,
                       params = params or {}, auth = auth, cookies = cookies, hooks = hooks,
        )
        prep = self.prepare_request(req)
        proxies = proxies or {}
        settings = self.merge_environment_settings(
            prep.url, proxies, stream, verify, cert
        )

        # Send the request.
        send_kwargs = {
            'timeout': timeout,
            'allow_redirects': allow_redirects,
        }
        send_kwargs.update(settings)
        resp = self.send(prep, **send_kwargs)
        return resp
```

可以看到request.api.request是通过构造session模块中的`Session`类，并调用其request方法完成请求的。Session类实现了Context Manage协议，主要是退出时关闭相应连接。

Session类的request方法先根据传入的参数（如URL等）构造一个Request类，并调用self.prepare_request方法预准备，最后调用self.send发送最终的请求。

URL的编码过程便在self.prepare_request方法中， 看看这个方法：

```python
# session.py
def prepare_request(self, request):
    # 此部分代码已忽略
    p = PreparedRequest()
    p.prepare(method=request.method.upper(), url=request.url, files=request.files,
        data=request.data, json=request.json,
        headers=merge_setting(request.headers, self.headers, dict_class=CaseInsensitiveDict),
        params=merge_setting(request.params, self.params), auth=merge_setting(auth, self.auth),
        cookies=merged_cookies,hooks=merge_hooks(request.hooks, self.hooks),
    )
    return p

# model.py
class PreparedRequest(RequestEncodingMixin, RequestHooksMixin):
    def prepare(self, method=None, url=None, headers=None, files=None,
        data=None, params=None, auth=None, cookies=None, hooks=None, json=None):
        """Prepares the entire request with the given parameters."""
        self.prepare_method(method)
        self.prepare_url(url, params)
        self.prepare_headers(headers)
        self.prepare_cookies(cookies)
        self.prepare_body(data, files, json)
        self.prepare_auth(auth, url)
        self.prepare_hooks(hooks)

    def prepare_url(self, url, params):
        # 这里requests库做了很多努力
        url = requote_uri(urlunparse([scheme, netloc, path, None, query, fragment]))
        self.url = url
```

prepare_request方法中又根据Request对象构造一个model模块中PreparedRequest对象，并调用它的prepare方法，在当中进行各种工作，其中prepare_url是我们关心的，不过里面进行了太多的判断，就不把源码贴出来了，想看的在这个地址：https://github.com/kennethreitz/requests/blob/master/requests/models.py#L350-L434

最终我们传进去的URL就被prepare_url函数进行了字符串编码替换，回到Session类，调用`resp = self.send(prep, **send_kwargs)`发送请求，prep即是上述PreparedRequest对象。

看似我们要阻止requests对URL编码是不让其调用prepare_url函数， 但prepare_url函数帮我们做了许多URL的校验工作，其实可以换个思路，在requests对URL编码之后，发送请求之前我们再将不需要的编码字符串反编码回去。

## 三 解决方法

由于URL在PreparedRequest类中转码后传到Session类的send方法中发送最终的请求，只要写一个类继承Session并重写send方法，在里面进行反编码，将PreparedRequest对象的url属性值改了。代码如下：

```python
import urllib
import requests

class NoQuoteSession(requests.Session):
    def send(self, prep, **send_kwargs):
        table = {
            urllib.quote('{'): '{',
            urllib.quote('}'): '}',
            urllib.quote(':'): ':',
            urllib.quote(','): ',',
        }
        for old, new in table.items():
            prep.url = prep.url.replace(old, new)
        return super(NoQuoteSeesion, self).send(prep, **send_kwargs)

s = NoQuoteSeesion()
res = s.get(url)
```

Done
