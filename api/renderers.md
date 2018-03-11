## 渲染Renderers

> 在TemplateResponse 实例返回给客户端之前，它必须被渲染。渲染的过程采用模板和上下文变量的中间表示形式，并最终将它转换为可以发送给客户端的字节流。  
—— Django官方文档(TemplateResponse and SimpleTemplateResponse)

点击此处，[查看官方文档](https://docs.djangoproject.com/en/2.0/ref/template-response/)

REST framework包含很多的内置渲染器类。允许你使用各种各样的媒体类型返回响应。还可以设置自定义的渲染器类，来灵活的设计你自己的媒体类型。

#### 渲染器是如何确定的

视图中有效的渲染器总是被设置为一个包含类的列表。当进入一个```View```逻辑的时候，REST framework会对传进来的请求执行内容协商，并且最终会确定一个最合适的渲染器，以满足Request。

内容协商的最基本的过程是检查HTTP请求头的```Accept```，用来确定哪种Media类型是响应最期望的。可选的，URL的格式后缀被用于显示请求特定的表示。比如，URL```http://example.com/api/users_count.json```可能只会返回JSON数据。

更多信息查看[内容协商文档](./cnegotiation.md)

#### 设置渲染器

默认情况下渲染器的设置可能是通过```DEFAULT_RENDERER_CLASSES```设置的全局的。比如，下面的设置将会使用```JSON```作为主要的媒体类型并且还包括可HTML浏览API。

```python
REST_FRAMEWORK = {
    'DEFAULT_RENDERER_CLASSES': (
        'rest_framework.renderers.JSONRenderer',
        'rest_framework.renderers.BrowsableAPIRenderer',
    )
}
```

你还可以为基于```APIView```的单个视图或者视图集合(```viewset```)设置渲染器：

```python
from django.contrib.auth.models import User
from rest_framework.renderers import JSONRenderer
from rest_framework.response import Response
from rest_framework.views import APIView

class UserCountView(APIView):
    """
    A view that returns the count of active users in JSON.
    """
    renderer_classes = (JSONRenderer, )

    def get(self, request, format=None):
        user_count = User.objects.filter(active=True).count()
        content = {'user_count': user_count}
        return Response(content)
```

或者为基于```@api_view```装饰器的函数视图设置渲染器：

```python
@api_view(['GET'])
@renderer_classes((JSONRenderer,))
def user_count_view(request, format=None):
    """
    A view that returns the count of active users in JSON.
    """
    user_count = User.objects.filter(active=True).count()
    content = {'user_count': user_count}
    return Response(content)
```

#### 渲染器类的排序

当为您的API指定渲染器类的时候，要考虑你想要分配给每种Media类型的优先级，这一点非常重要。当客户端没有指定一个可以接受的表现形式，比如发送一个```Accept: */*```HTTP请求头，或者在请求头中不包括任何```Accept```，那么REST framework将会选择列表中的第一个渲染器来用于响应数据。

比如你的API服务器支持```JSON```的响应和HTML可浏览的API，你可能想要指定```JSONRenderer```为你默认的渲染器，这样那些没有指定```Accept```HTTP头的客户端将会接收到```JSON```响应。

如果你的API服务器，包含可以根据请求来确定提供常规网页和API响应的视图，你可能会考虑设置```TemplateHTMLRenderer```作为你的默认的渲染器。

---

<br />
<br />
<br />
<br />

### API 参考

关于渲染器的源码位置： ```rest_framework.renderers```

#### ```JSONRenderer```渲染器

使用UTF-8编码，将请求数据渲染为```JSON```。

注意，默认情况下包含Unicode字符串，并且用紧凑的格式来呈现，没有多余的空白符：

```python
{"unicode black star":"★","value":999}
```

客户端可能会包含额外的Media类型参数```indent```，这表示返回的```JSON```数据会被缩进。比如```Accept: application/json; indent=4```：

```javascript
{
    "unicode black star": "★",
    "value": 999
}
```

默认的编码风格使用```UNICODE_JSON```和```COMPACT_JSON```设置来修改。

在```rest_framework.settings.DEFAULTS``` 中有如下设置：

```python
DEFAULTS = {
    ……省略……

    # Encoding
    'UNICODE_JSON': True,
    'COMPACT_JSON': True,
    'STRICT_JSON': True,
    'COERCE_DECIMAL_TO_STRING': True,
    'UPLOADED_FILES_USE_URL': True,

    # Browseable API
    'HTML_SELECT_CUTOFF': 1000,
    'HTML_SELECT_CUTOFF_TEXT': "More than {count} items...",

    ……省略……
}
```

```.media_type```为 ```application/json```

URL ```.format```后缀为： ```'.json'```

```.charset```: ```None```

#### ```TemplateHTMLRenderer```渲染器

使用Django标准的模板渲染将数据渲染到HTML中。不像其他渲染器，数据被传递给```Response```不需要序列化。另外，与其他渲染器不同，在创建```Response```的时候你需要包含```template_name```参数。

渲染器```TemplateHTMLRenderer```将会创建一个```RequestContext```， 使用```response.data```作为内容，并且确定一个模板名称用于渲染一个内容。

模板名称是按照下面的顺序来确定的：

1. 显式的传递给```Response```对象一个```template_name```参数。
2. 在类里面明确的设置```.template_name```属性。
3. 返回```view.get_template_names()```的结果。

下面是一个使用```TemplateHTMLRenderer```渲染器的示例：

```python
class UserDetail(generics.RetrieveAPIView):
    """
    A view that returns a templated HTML representation of a given user.
    """
    queryset = User.objects.all()
    renderer_classes = (TemplateHTMLRenderer,)

    def get(self, request, *args, **kwargs):
        self.object = self.get_object()
        return Response({'user': self.object}, template_name='user_detail.html')
```

如果你要使用```TemplateHTMLRenderer```，你可以使用REST framework返回一个常规的HTML 页面，或者从一个单一的Endpoint同时返回HTML和API响应数据。

如果你使用```TemplateHTMLRenderer```和其他的渲染器来构建你的Web站点，你应该考虑将```TemplateHTMLRenderer```放在```renderer_classes```列表的第一个，这样即使客户端的HTTP Request Header中的```ACCEPT```不正确，它也会优先使用```TemplateHTMLRenderer```。

参考链接，[HTML & Forms Topic Page](http://www.django-rest-framework.org/topics/html-and-forms/)来查看更多的案例使用。

```.media_type```为 ```text/html```

URL ```.format```后缀为： ```'.html'```

```.charset```: ```utf-8```

#### ```StaticHTMLRenderer```渲染器

一个简单的渲染器，它会返回一个```预渲染```的HTML。不像其他的渲染器，传递给```Response```对象的数据应该是一个表示要返回的字符串。

下面是一个使用```StaticHTMLRenderer```的例子：

```python
@api_view(('GET',))
@renderer_classes((StaticHTMLRenderer,))
def simple_html_view(request):
    data = '<html><body><h1>Hello, world</h1></body></html>'
    return Response(data)
```

如果你要使用```StaticHTMLRenderer```，你可以使用REST framework返回一个常规的HTML 页面，或者从一个单一的Endpoint同时返回HTML和API响应数据。

```.media_type```为 ```text/html```

URL ```.format```后缀为： ```'.html'```

```.charset```: ```utf-8```

注意： ```StaticHTMLRenderer```渲染器类是```TemplateHTMLRenderer```渲染器类的子类。

#### ```BrowsableAPIRenderer```渲染器

为可视化的API将数据渲染到HTML中。

![](../images/render1.png)

该渲染器，会确定哪个渲染器具有最高的级别，然后使用它在HTML页面中显示API的响应样式。

#### ```AdminRenderer```渲染器

#### ```HTMLFormRenderer```渲染器

#### ```MultiPartRenderer```渲染器

---

<br />
<br />
<br />
<br />

### 自定义渲染器

#### 例子

#### 设置字符集

---

<br />
<br />
<br />
<br />

### 高级渲染器用法

#### 通过媒体类型改变行为

#### Underspecifying the media type

#### 设计你的媒体类型

#### HTML 错误视图

---

<br />
<br />
<br />
<br />

### 第三方包

#### YAML

#### XML

#### JSONP

#### MessagePack

#### CSV

#### UltraJSON

#### 驼峰JSON

#### Pandas (CSV, Excel, PNG)

#### LaTeX

---

参考链接：

- https://docs.djangoproject.com/en/2.0/ref/template-response/
