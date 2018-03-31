# coreapi

模式是一个机器可读文档，描述可用的API端点，它们的URL以及它们支持的操作。

模式可以成为自动生成文档的工具，也可用于驱动可与API交互的动态客户端库。


## 核心API
为了提供模式支持，REST框架使用Core API。

核心API是用于描述API的文档规范。它用于提供可用端点的内部表示形式以及API公开的可能交互。它可以用于服务器端或客户端。

在服务器端使用时，```Core API```允许API支持以各种架构或超媒体格式进行渲染。

在使用客户端时，```Core API```允许动态驱动的客户端库，它们可以与任何公开受支持的模式或超媒体格式的API进行交互。


## 添加一个模式
REST框架支持显式定义的模式视图或自动生成的模式。由于我们使用视图集和路由器，因此我们可以简单地使用自动模式生成。

您需要安装```coreapi``` python包。

```python
$ pip install coreapi
```

我们现在可以通过在我们的URL配置中包含一个自动生成的模式视图来为我们的API包含模式。

```python
from rest_framework.schemas import get_schema_view

schema_view = get_schema_view(title='Pastebin API')

urlpatterns = [
    url(r'^schema/$', schema_view),
    ...
]
```

如果您在浏览器中访问API根，您现在应该可以看到corejson, 表示可以工作.

![](./corejson-format.png)


我们还可以通过在```Accept```标题中指定所需的内容类型来从命令行请求架构。

```python
$ http http://127.0.0.1:8000/schema/ Accept:application/coreapi+json
HTTP/1.0 200 OK
Allow: GET, HEAD, OPTIONS
Content-Type: application/coreapi+json

{
    "_meta": {
        "title": "Pastebin API"
    },
    "_type": "document",
    ...
```

默认的输出风格是使用Core JSON编码。

其他架构格式，例如Open API（以前称为Swagger）也受支持。


## 使用命令行客户端
现在我们的API公开了一个模式端点，我们可以使用一个动态客户端库来与API进行交互。为了演示这一点，我们使用Core API命令行客户端。

命令行客户端可用```coreapi-cli```包：

```python
$ pip install coreapi-cli
```

现在检查它在命令行上是否可用...

```python
$ coreapi
Usage: coreapi [OPTIONS] COMMAND [ARGS]...

  Command line client for interacting with CoreAPI services.

  Visit http://www.coreapi.org for more information.

Options:
  --version  Display the package version number.
  --help     Show this message and exit.

Commands:
...
```

首先，我们将使用命令行客户端加载API模式。

```python
$ coreapi get http://127.0.0.1:8000/schema/
<Pastebin API "http://127.0.0.1:8000/schema/">
    snippets: {
        highlight(id)
        list()
        read(id)
    }
    users: {
        list()
        read(id)
    }
```

我们还没有进行身份验证，所以现在我们只能看到只读端点，与我们如何设置API的权限一致。

让我们尝试使用命令行客户端列出现有snippets：

```python
$ coreapi action snippets list
[
    {
        "url": "http://127.0.0.1:8000/snippets/1/",
        "id": 1,
        "highlight": "http://127.0.0.1:8000/snippets/1/highlight/",
        "owner": "lucy",
        "title": "Example",
        "code": "print('hello, world!')",
        "linenos": true,
        "language": "python",
        "style": "friendly"
    },
    ...
    
```

一些API端点需要命名参数。例如，要获取特定代码段的高亮HTML，我们需要提供一个id。

```python
$ coreapi action snippets highlight --param id=1
<!DOCTYPE html PUBLIC "-//W3C//DTD HTML 4.01//EN" "http://www.w3.org/TR/html4/strict.dtd">

<html>
<head>
  <title>Example</title>
  ...
```    

## 验证我们的客户端

如果我们希望能够创建，编辑和删除snippets，我们需要以有效用户身份进行身份验证。在这种情况下，我们只使用基本身份验证。

确保使用您的实际用户名和密码替换<username>和<password>在下面。

```python
$ coreapi credentials add 127.0.0.1 <username>:<password> --auth basic
Added credentials
127.0.0.1 "Basic <...>"
```

现在，如果我们再次获取模式，我们应该能够看到完整的可用交互。

```python
$ coreapi reload
Pastebin API "http://127.0.0.1:8000/schema/">
    snippets: {
        create(code, [title], [linenos], [language], [style])
        delete(id)
        highlight(id)
        list()
        partial_update(id, [title], [code], [linenos], [language], [style])
        read(id)
        update(id, code, [title], [linenos], [language], [style])
    }
    users: {
        list()
        read(id)
    }
    
```

我们现在可以与这些端点进行交互。例如，要创建一个新的片段：

```python
$ coreapi action snippets create --param title="Example" --param code="print('hello, world')"
{
    "url": "http://127.0.0.1:8000/snippets/7/",
    "id": 7,
    "highlight": "http://127.0.0.1:8000/snippets/7/highlight/",
    "owner": "lucy",
    "title": "Example",
    "code": "print('hello, world')",
    "linenos": false,
    "language": "python",
    "style": "friendly"
}
```

并删除一个片段：

```python
$ coreapi action snippets delete --param id=7
```

除了命令行客户端之外，开发人员还可以使用客户端库与您的API进行交互。Python客户端库是第一个可用的客户端库，计划很快发布Javascript客户端库。

有关自定义模式生成和使用Core API客户端库的更多详细信息，您需要参阅完整的文档。

## 回顾我们的工作
使用极少量的代码，我们现在已经有了一个完整的```pastebin Web API```，它完全可以浏览网页，包含一个模式驱动的客户端库，并且包含身份验证，每对象权限和多个呈现器格式。

我们已经完成了设计过程的每一步，并且看到如果我们需要定制任何可以逐步使用常规Django视图的方法。

您可以在GitHub上查看最终的教程代码，或者在沙箱中试用一个实例。

## 向上和向上
我们已经完成了我们的教程。如果您想更多地参与REST框架项目，可以从以下几个地方开始：

通过审查和提交问题，并提出pull请求，为GitHub贡献力量。
加入REST框架讨论组，并帮助构建社区。

在Twitter上关注作者并点个赞。

现在去做很棒的事情。