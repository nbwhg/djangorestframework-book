# Requests & Responses

从这一章，我们将真正开始涵盖REST框架的核心。我们先来介绍一些基本的构建块。

## 请求对象

REST框架引入了一个```Request```扩展常规的对象```HttpRequest```，并提供更灵活的请求解析。```Request```对象的核心功能是```request.data```属性，与```request.POST```Web API 类似，但更加有用。

```python
request.POST  # 只处理form表单数据,只适用POST方法.
request.data  # 处理任意方法. 'POST', 'PUT' 和 'PATCH' 方法.
```


## 响应对象

REST框架还引入了一个```Response```对象，该对象是一种```TemplateResponse```类型，它使用未呈现的内容并使用内容协商来确定返回给客户端的正确内容类型。

```
return Response(data)  # 根据用户请求返回的内容.
```

## 状态码

在视图中使用数字HTTP状态代码并不总是明显的阅读，而且如果错误代码出现错误，会很不容易注意到。REST框架为每个状态代码提供更明确的标识符，例如```HTTP_400_BAD_REQUEST```在```status```模块中。这是一个好想法，而不是使用数字标识符。


## Wrapping API views 装饰 API 视图

REST框架提供了两个可用于编写API视图的装饰器。

1. @api_view用于处理基于函数的视图的装饰器。
2. APIView 基于类的视图工作。

这些装饰器提供了一些功能，例如确保```Request```在视图中接收实例，并向```Response```对象添加上下文，以便可以执行内容协商。

这些装饰器还提供了一些功能，例如```405 Method Not Allowed```在适当的时候返回响应，以及处理```ParseError```在```request.data```访问输入错误的格式时发生的任何异常。


## Pulling it all together  整合到一起

好吧，让我们开始使用这些新组件来写几个视图。

我们不再需要我们的```JSONResponse```教程里的```views.py```，所以删除它。完成后，我们可以开始重构我们的views。

```python
from rest_framework import status
from rest_framework.decorators import api_view
from rest_framework.response import Response
from snippets.models import Snippet
from snippets.serializers import SnippetSerializer


@api_view(['GET', 'POST'])
def snippet_list(request):
    """
    列出所有snippets, 或者创建一个新的snippet.
    """
    if request.method == 'GET':
        snippets = Snippet.objects.all()
        serializer = SnippetSerializer(snippets, many=True)
        return Response(serializer.data)

    elif request.method == 'POST':
        serializer = SnippetSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
```


我们的实例视图比前一个示例有所改进。它更简洁一些，如果我们使用Forms API，代码现在感觉非常相似。我们还使用了指定的状态代码，这使得响应的含义更加明显。

以下是views.py模块中单个snippet的视图。

```python
@api_view(['GET', 'PUT', 'DELETE'])
def snippet_detail(request, pk):
    """
    Retrieve, update or delete a code snippet.
    """
    try:
        snippet = Snippet.objects.get(pk=pk)
    except Snippet.DoesNotExist:
        return Response(status=status.HTTP_404_NOT_FOUND)

    if request.method == 'GET':
        serializer = SnippetSerializer(snippet)
        return Response(serializer.data)

    elif request.method == 'PUT':
        serializer = SnippetSerializer(snippet, data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

    elif request.method == 'DELETE':
        snippet.delete()
        return Response(status=status.HTTP_204_NO_CONTENT)
```


这应该都非常熟悉 - 这与使用常规Django视图没有多大区别。

请注意，我们不再明确地将我们的请求或响应绑定到给定的内容类型.```request.data```可以处理传入的json请求，但它也可以处理其他格式。同样，我们使用数据返回响应对象，但允许```REST```框架将响应展示给我们正确的内容类型。


## 为我们的URL添加可选的格式后缀

为了利用我们的响应不再被硬连接到单一内容类型的事实，让我们将格式后缀的支持添加到我们的API端点。使用格式后缀给了我们明确引用给定格式的URL，并且意味着我们的API将能够处理诸如http://example.com/api/items/4.json之类的 URL 。

首先，在两个视图中添加一个```format```格式关键字参数，就像这样。

```
def snippet_list(request, format=None):
```

and

```
def snippet_detail(request, pk, format=None):
```


现在稍微更新snippets/urls.py文件，以附加一组format_suffix_patterns到现有的URL。

```python
from django.conf.urls import url
from rest_framework.urlpatterns import format_suffix_patterns
from snippets import views

urlpatterns = [
    url(r'^snippets/$', views.snippet_list),
    url(r'^snippets/(?P<pk>[0-9]+)$', views.snippet_detail),
]

urlpatterns = format_suffix_patterns(urlpatterns)
```

我们不一定需要添加这些额外的url模式，但是它提供了一种简单、干净的方式来引用特定的格式。


# 它看起来如何？
继续从命令行测试API，就像我们在教程1中所做的那样。这些工作都非常类似，尽管如果我们发送无效请求，我们已经有了一些更好的错误处理。

我们可以像以前一样获取所有snippets的列表。

```python
http http://127.0.0.1:8000/snippets/

HTTP/1.1 200 OK
...
[
  {
    "id": 1,
    "title": "",
    "code": "foo = \"bar\"\n",
    "linenos": false,
    "language": "python",
    "style": "friendly"
  },
  {
    "id": 2,
    "title": "",
    "code": "print \"hello, world\"\n",
    "linenos": false,
    "language": "python",
    "style": "friendly"
  }
]
```

我们可以通过使用```Accept header```来控制返回的响应格式：

```
http http://127.0.0.1:8000/snippets/ Accept:application/json  # Request JSON
http http://127.0.0.1:8000/snippets/ Accept:text/html         # Request HTML
```

或者通过附加格式后缀访问：

```
http http://127.0.0.1:8000/snippets.json  # JSON suffix
http http://127.0.0.1:8000/snippets.api   # Browsable API suffix
```

同样，我们可以使用```Content-Type```标题来控制我们发送的请求的格式。

```
# POST using form data
http --form POST http://127.0.0.1:8000/snippets/ code="print 123"

{
  "id": 3,
  "title": "",
  "code": "print 123",
  "linenos": false,
  "language": "python",
  "style": "friendly"
}

# POST using JSON
http --json POST http://127.0.0.1:8000/snippets/ code="print 456"

{
    "id": 4,
    "title": "",
    "code": "print 456",
    "linenos": false,
    "language": "python",
    "style": "friendly"
}
```


如果您添加```--debug```向上述```http```请求，您将能够在请求标头中看到请求的类型。

现在，通过访问http://127.0.0.1:8000/snippets/，在Web浏览器中打开API 。

## 浏览功能

因为API根据客户端请求选择响应的内容类型，所以默认情况下，当Web浏览器请求该资源时，它将返回HTML格式的资源。这允许API返回完全可以浏览网页的HTML表示。

拥有可浏览网页的API是一个巨大的可用性胜利，并且使开发和使用您的API变得更加容易。它也大大降低了想要检查和使用API​​的其他开发人员的进入门槛。

有关可浏览的API功能以及如何对其进行定制的更多信息，请参阅可浏览的API主题。

## 下一步是什么？

在教程第3部分中，我们将开始使用基于类的视图，并了解通用视图如何减少编写的代码量。