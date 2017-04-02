---
title: Falcon框架中Routing的设计及源码分析
category: Python
---

## 一、Falcon框架介绍
Falcon是一个非常快并且小巧的Python Web框架，适合开发微服务、后端API及高性能的框架。Falcon推崇使用RESTful形式的API接口。官方网站：http://falconframework.org/

Falcon的路由实现比较有特色，这篇文章主要分析它的实现方式。

## 二、Falcon添加路由方法
编写一个Falcon应用服务器是非常简单的，一个官方的教程如下：

```python
import falcon
class ThingsResource(object):
    def on_get(self, req, resp):
        """Handles GET requests"""
        resp.status = falcon.HTTP_200  # This is the default status
        resp.body = ('\nTwo things awe me most, the starry sky '
                     'above me and the moral law within me.\n'
                     '\n'
                     '    ~ Immanuel Kant\n\n')
# falcon.API instances are callable WSGI apps
app = falcon.API()
# Resources are represented by long-lived class instances
things = ThingsResource()
# things will handle all requests to the '/things' URL path
app.add_route('/things', things)
```

app变量是一个WSGI应用的入口，通过方法`add_route`添加了一个`/things`URL与资源的对应关系，即访问`/things`，Falcon会调用`ThingsResource`类的`on_get`方法。

如何保存和查找这个对应关系，最简便的方式是使用一个字典，如`{<url>:<method>}`，当一个请求到来了，提出其中的路径，在字典中查找到当前路径对应的方法。

但是，多级路径`/things`、`/things/one`存储一起存在冗余，也不能支持URL变量甚至复杂的正则表达式定制的URL，如希望`/things/{id}`匹配请求路径`/things/1`并提取其中的数值1进行后续处理。

看看Falcon支持的几种URL类型：

```python
#   /foo/{thing1}
#   /foo/all
#   /foo/{thing1}.{ext}
#   /foo/{thing2}.detail.{ext}

#   注意： 同级且不同变量名字的URL在Falcon中是不允许的，例如：
#   /foo/{thing1}
#   /foo/{thing2}
#   或者这样：
#   /foo/{thing1}.{ext}
#   /foo/{thing2}.{ext}

```

`{thing1}`中的URL值将被Falcon赋值给`thing1`变量并传替到方法的参数中。

在Falcon内部，其实是使用树型结构保存URL映射。所有添加的路由URL将组织成一棵棵的树，每个树的节点是URL的每个片段，如`foo`或者`all`，只有树的叶子对应到资源方法对象。

下面逐步分析Falcon如何实现URL树以及URL的匹配查找。

## 三、CompiledRouter类

实际上Falcon框架允许使用自定义的router类，只要定制的类实现`add_route`和`find`这两个方法。`add_route`接收三个参数（uri_template, method_map, resource），分别是URL路径字符串、映射的方法、资源类，而`find`方法接收一个URL字符串，返回一个4元组(resouce, method_map, params, uri_template)。

在实例化falcon.API时传入定制的router类即可。如：

```python
fancy = FancyRouter()
app = falcon.API(router=fancy)
```

Falcon默认的router类为`falcon.router.CompiledRouter`，它通过预编译的python代码实现快速的URL匹配，而不是每一次查询都分析路由树。

分析`CompiledRouter`类之前先看一下`CompiledRouterNode`类，它代表树的节点。

```python
class CompiledRouterNode(object):
    _regex_vars = re.compile('{([-_a-zA-Z0-9]+)}')

    def __init__(self, raw_segment,
                 method_map=None, resource=None, uri_template=None):
        self.children = []      # 代表子树
        self.raw_segment = raw_segment  # 通过'/'分离URL后的片段
        self.method_map = method_map
        self.resource = resource
        self.uri_template = uri_template
        self.is_var = False         # 变量型节点，如{thing1}
        self.is_complex = False     # 复杂变量型节点，如{thing1}.detail
        self.var_name = None        # 保存变量名
        seg = raw_segment.replace('.', '\\.')
        # 精简部分代码
        # 此部分用于判断节点类型
        # ...
    def matches(self, segment):
        return segment == self.raw_segment
```
对添加到项目中的路由URL，Falcon会分解为一个个字符串片段，并为每个片段建一个`CompiledRouterNode`类节点，初始化同时标记节点类型(简单或复杂)。这些节点会组装成一棵棵的树。

这些树包含在`CompiledRouter`类的`_roots`列表变量中，看一下`CompiledRouter`类。

```python
class CompiledRouter(object):

    def __init__(self):
        self._roots = []    # 保存各个路由树的列表
        self._find = self._compile()    # 实现实际的find函数，下面细说

        # 下面四个是编译生成python代码时用到的变量
        self._code_lines = None
        self._src = None
        self._expressions = None
        self._return_values = None
```

`CompiledRouter`类的`add_route`方法：

```python
def add_route(self, uri_template, method_map, resource):
    # 此处精简部分代码
    # ...

    # path保存URL分离后的片段字符串
    path = uri_template.strip('/').split('/')

    # 一个内部函数insert，遍历树，并建立树叶节点。nodes为保存'CompiledRouterNode'类的列表
    def insert(nodes, path_index=0):
        for node in nodes:
            segment = path[path_index]
            if node.matches(segment):
                path_index += 1
                if path_index == len(path):
                    # NOTE(kgriffs): Override previous node
                    node.method_map = method_map
                    node.resource = resource
                    node.uri_template = uri_template
                else:
                    insert(node.children, path_index)

                return

        # NOTE(richardolsson): If we got this far, the node doesn't already
        # exist and needs to be created. This builds a new branch of the
        # routing tree recursively until it reaches the new node leaf.
        new_node = CompiledRouterNode(path[path_index])
        nodes.append(new_node)
        if path_index == len(path) - 1:
            new_node.method_map = method_map
            new_node.resource = resource
            new_node.uri_template = uri_template
        else:
            insert(new_node.children, path_index + 1)

    insert(self._roots)
    # 此处会进行find方法的预编码
    self._find = self._compile()
```

`add_route`方法会分析所给的URL字符, 通过递归调用`insert`方法，找到需要新增的节点，如果是树叶节点（即URL最后一部分），则设置`uri_template, method_map, resource`，最后的`self._find = self._compile()`会进行查找方法的预编译。既是说，每添加一个URL路由，就进行一次查找方法的预编译。

看看这个预编译的魔法是什么。前面已经说了，Falcon框架的router类要实现两个方法`add_route`和`find`，`find`方法查找给予的URL并返回最终调用的资源类和方法。

```python
def find(self, uri):
    path = uri.lstrip('/').split('/')
    params = {}
    node = self._find(path, self._return_values, self._expressions, params)
    if node is not None:
        return node.resource, node.method_map, params, node.uri_template
    else:
        return None
```
`find`方法实际调用的是`_find`方法，传入了四个参数：

- `path`: URL分解后的列表。
- `self._return_values`: 一个列表，预编译代码后会保存一系列node，列表是实例的属性，最终的查找方法即是从此列表中返回node。
- `self._expressions`: 一个列表，保存的是正则表达式。
- `params`: 这不是实例的属性，`_find`方法执行后得到的URL变量值会存到`params`字典。

`self._find`方法是`self._compile()`所返回的函数，实际的预编译魔法即在此函数当中。

`self._compile`方法会遍历`self._roots`里的所有node，生成当前路由树应执行的快速查找方法`self._find`。预编译后生成的代码类似这样：

```python
def find(path, return_values, expressions, params):
    path_len = len(path)
    if path_len > 0 and path[0] == "books":
        if path_len > 1:
            params["book_id"] = path[1]
            return return_values[1]
        return return_values[0]
    if path_len > 0 and path[0] == "authors"
        if path_len > 1:
            params["author_id"] = path[1]
            if path_len > 2:
                match = expressions[0].search(path[2])
                if match is not None:
                    params.update(match.groupdict())
                    return return_values[4]
            return return_values[3]
        return return_values[2]
```
源代码可以查看https://github.com/falconry/falcon/blob/master/falcon/routing/compiled.py

通过内建`compile`方法编译字符串代码生成code对象再调用`exec`方法得到最终的`_find`函数，很有意思吧。

Done
