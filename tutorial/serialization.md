# 序列化

## 介绍


本章将介绍创建一个简单的粘贴WebAPI的程序。在此过程中，它将介绍构成REST框架的各种组件，并让您对所有组件的组合方式有一个全面的了解。

此章教程是相当深入的，so you should probably get a cookie and a cup of your favorite brew before getting started. 如果您只想快速浏览一下，则应该转到快速入门文档。

---
注意：本教程的代码可以在GitHub的tomchristie/rest-framework-tutorial存储库中找到。线上测试版,可以在https://restframework.herokuapp.com/这里找到.

---

## 创建一个新的ENV环境

在我们做任何事情之前，我们都应使用```virtualenv```创建一个新的虚拟环境。这将保证我们的配置与我们正在进行的其他任何项目的配置保持良好的隔离。

```python
virtualenv env
source env/bin/activate
```

现在我们进入了virtualenv环境，我们可以安装我们所需的包。

```python
pip install django
pip install djangorestframework
pip install pygments  # We'll be using this for the code highlighting
```
注意：要随时退出```virtualenv```环境，只需键入```deactivate```。要了解更多信息，请参阅virtualenv文档。

## 开始

让我们创建一个新的项目.

```python
cd ~
django-admin.py startproject tutorial
cd tutorial
```

项目创建成功后,我们可以创建一个应用程序,我们将用它来创建一个简单的WebAPI。

```python
python manage.py startapp snippets
```

我们需要添加我们的新```snippets```应用程序和``rest_framework``应用程序```INSTALLED_APPS```。我们来编辑这个```tutorial/settings.py```文件：

```python
INSTALLED_APPS = (
    ...
    'rest_framework',
    'snippets.apps.SnippetsConfig',
)
```

## 创建model

出于作为教程目的,我们将先创建一个简单的```Snippet``` model,创建```snippets/models.py```文件。注意:良好的编程实践是包括注释的。在官方代码版本库里可以找到注释,此处省略了注释,只需要关注代码即可。


```python
from django.db import models
from pygments.lexers import get_all_lexers
from pygments.styles import get_all_styles

LEXERS = [item for item in get_all_lexers() if item[1]]
LANGUAGE_CHOICES = sorted([(item[1][0], item[0]) for item in LEXERS])
STYLE_CHOICES = sorted((item, item) for item in get_all_styles())


class Snippet(models.Model):
    created = models.DateTimeField(auto_now_add=True)
    title = models.CharField(max_length=100, blank=True, default='')
    code = models.TextField()
    linenos = models.BooleanField(default=False)
    language = models.CharField(choices=LANGUAGE_CHOICES, default='python', max_length=100)
    style = models.CharField(choices=STYLE_CHOICES, default='friendly', max_length=100)

    class Meta:
        ordering = ('created',)
```


我们根据snippet model初始化,并且同步到数据库。

```python
python manage.py makemigrations snippets
python manage.py migrate
```


## 创建一个序列化器类
我们需要开始使用Web API的第一件事就是提供一种将代码片段实例序列化和反序列化为表示形式的方法json。我们可以通过声明与Django的表单非常相似的序列化器来实现这一点。在snippets名为的目录中创建一个文件serializers.py并添加以下内容。

我们在Web API上首先要做的就是提供一种序列化和反序列化的方法，将代码片段实例序列化为json之类的表示。我们可以通过声明与Django的形式非常相似的序列化器来实现这一点。在名为序列化器的代码片段目录中创建一个文件。py并添加以下内容。


我们需要开始使用Web API的第一件事就是提供一种序列化和反序列化的方法，将代码片段实例序列化为```json```,我们可以用与Django的表单非常相似的序列化器来实现这一点. 在```snippets```目录下创建```serializers.py```文件，并添加以下内容

```python
from rest_framework import serializers
from snippets.models import Snippet, LANGUAGE_CHOICES, STYLE_CHOICES


class SnippetSerializer(serializers.Serializer):
    id = serializers.IntegerField(read_only=True)
    title = serializers.CharField(required=False, allow_blank=True, max_length=100)
    code = serializers.CharField(style={'base_template': 'textarea.html'})
    linenos = serializers.BooleanField(required=False)
    language = serializers.ChoiceField(choices=LANGUAGE_CHOICES, default='python')
    style = serializers.ChoiceField(choices=STYLE_CHOICES, default='friendly')

    def create(self, validated_data):
        """
        根据已验证的数据，创建并返回一个新的`Snippet`实例。
        """
        return Snippet.objects.create(**validated_data)

    def update(self, instance, validated_data):
        """
        根据已验证的数据，更新并返回一个已存在的`Snippet`段实例。
        """
        instance.title = validated_data.get('title', instance.title)
        instance.code = validated_data.get('code', instance.code)
        instance.linenos = validated_data.get('linenos', instance.linenos)
        instance.language = validated_data.get('language', instance.language)
        instance.style = validated_data.get('style', instance.style)
        instance.save()
        return instance


```

```
```
这个序列化器类的第一部分定义了序列化/反序列化的字段。该```create()```和```update()```方法定义了在调用```serializer.save()```时如何创建或修改完全成熟的实例。

序列化器类与Django Form类非常相似，并在各个字段中包含类似的验证标志，例如```required```，```max_length```和```default```。

上面的字段标志还可以控制在某些情况下应该如何显示序列化程序，例如在展现为HTML时。```{'base_template': 'textarea.html'}```上面的标志等同于```widget=widgets.Textarea```在Django Form类上使用。这对于控制如何显示可浏览的API特别有用，我们将在本教程后面看到的。

我们实际上也可以通过使用这个```ModelSerializer```类节省一些时间，我们在后面会看到，但现在我们将理解序列化器如何定义的。


## 使用序列化器

在进一步讨论之前，我们将先熟悉使用新的序列化器类。让我们进入Django shell。

```python
python manage.py shell
```

我们导入一些模块, 然后创建一些实例

```python
rom snippets.models import Snippet
from snippets.serializers import SnippetSerializer
from rest_framework.renderers import JSONRenderer
from rest_framework.parsers import JSONParser

snippet = Snippet(code='foo = "bar"\n')
snippet.save()

snippet = Snippet(code='print "hello, world"\n')
snippet.save()
```

现在我们已经有了一些实例。让我们来看看序列化其中一个实例。

```python
serializer = SnippetSerializer(snippet)
serializer.data
# {'id': 2, 'title': u'', 'code': u'print "hello, world"\n', 'linenos': False, 'language': u'python', 'style': u'friendly'}
```

我们已经将模型实例转换为Python原生数据类型。我们将数据转换为json,完成序列化。

```python
content = JSONRenderer().render(serializer.data)
content
# '{"id": 2, "title": "", "code": "print \\"hello, world\\"\\n", "linenos": false, "language": "python", "style": "friendly"}'
```

反序列化也是相似的。首先，我们将一个流解析为Python原生数据类型。

```python
from django.utils.six import BytesIO

stream = BytesIO(content)
data = JSONParser().parse(stream)
```

然后，我们将这些原生数据类型恢复到一个完全填充的对象实例中。

```python
serializer = SnippetSerializer(data=data)
serializer.is_valid()
# True
serializer.validated_data
# OrderedDict([('title', ''), ('code', 'print "hello, world"\n'), ('linenos', False), ('language', 'python'), ('style', 'friendly')])
serializer.save()
# <Snippet: Snippet object>
```

请注意这个API与表单的工作方式是多么相似。当我们开始编写使用我们的序列化器的视图时，相似性应该变得更加明显。

我们还可以序列化queryset，而不是模型实例。为了实现这一操作，我们只需要在序列化器参数中添加```many=True```标记。


```python
serializer = SnippetSerializer(Snippet.objects.all(), many=True)
serializer.data
# [OrderedDict([('id', 1), ('title', u''), ('code', u'foo = "bar"\n'), ('linenos', False), ('language', 'python'), ('style', 'friendly')]), OrderedDict([('id', 2), ('title', u''), ('code', u'print "hello, world"\n'), ('linenos', False), ('language', 'python'), ('style', 'friendly')]), OrderedDict([('id', 3), ('title', u''), ('code', u'print "hello, world"'), ('linenos', False), ('language', 'python'), ('style', 'friendly')])]
```


## 使用ModelSerializers

我们```SnippetSerializer```类复制Snippet model中的大量信息。如果我们能让代码更简洁一点，那就更好了。
就像Django提供```Form```类和```ModelForm```类一样，REST框架包含```Serializer```类和```ModelSerializer```类。

让我们用```ModelSerializer```类重构我们的序列化程序。打开```snippets/serializers.py```文件，并用下面的代码替换。

```python
class SnippetSerializer(serializers.ModelSerializer):
    class Meta:
        model = Snippet
        fields = ('id', 'title', 'code', 'linenos', 'language', 'style')
```

序列化器的一个很好的特性是，您可以通过打印其表示来检查序列化程序实例中的所有字段。打开Django shell python manage.py shell，然后尝试以下操作：

```python
from snippets.serializers import SnippetSerializer
serializer = SnippetSerializer()
print(repr(serializer))
# SnippetSerializer():
#    id = IntegerField(label='ID', read_only=True)
#    title = CharField(allow_blank=True, max_length=100, required=False)
#    code = CharField(style={'base_template': 'textarea.html'})
#    linenos = BooleanField(required=False)
#    language = ChoiceField(choices=[('Clipper', 'FoxPro'), ('Cucumber', 'Gherkin'), ('RobotFramework', 'RobotFramework'), ('abap', 'ABAP'), ('ada', 'Ada')...
#    style = ChoiceField(choices=[('autumn', 'autumn'), ('borland', 'borland'), ('bw', 'bw'), ('colorful', 'colorful')...
```


记住```ModelSerializer```类没有做任何特别的事情，它们只是创建序列化类的一个快捷方式:

1.自动确定的一组字段。

2.简单的默认实现```create()```和```update()```方法。


## 使用序列化器来编写Django的视图
我们来看看如何使用新的Serializer类来编写一些API视图。目前，我们不会使用任何REST框架的其他功能，我们只会将视图编写为常规的Django视图。

编辑```snippets/views.py```文件，并添加以下内容。

```python
from django.http import HttpResponse, JsonResponse
from django.views.decorators.csrf import csrf_exempt
from rest_framework.renderers import JSONRenderer
from rest_framework.parsers import JSONParser
from snippets.models import Snippet
from snippets.serializers import SnippetSerializer
```

我们的API的根本将是一个视图，支持列出所有snippets实例，或新建一个snippets实例。

```python
@csrf_exempt
def snippet_list(request):
    """
    List all code snippets, or create a new snippet.
    """
    if request.method == 'GET':
        snippets = Snippet.objects.all()
        serializer = SnippetSerializer(snippets, many=True)
        return JsonResponse(serializer.data, safe=False)

    elif request.method == 'POST':
        data = JSONParser().parse(request)
        serializer = SnippetSerializer(data=data)
        if serializer.is_valid():
            serializer.save()
            return JsonResponse(serializer.data, status=201)
        return JsonResponse(serializer.errors, status=400)
```

请注意，因为我们希望没有CSRF令牌的客户端访问到这个视图，所以我们需要将视图标记为```csrf_exempt```。
这不是你需要考虑做的事，并且REST框架视图提供更高级的，但它现在就可以满足我们需求。

我们还需要一个与snippets相对应的视图，可以用来查看，更新或删除操作。

```python
@csrf_exempt
def snippet_detail(request, pk):
    """
    Retrieve, update or delete a code snippet.
    """
    try:
        snippet = Snippet.objects.get(pk=pk)
    except Snippet.DoesNotExist:
        return HttpResponse(status=404)

    if request.method == 'GET':
        serializer = SnippetSerializer(snippet)
        return JsonResponse(serializer.data)

    elif request.method == 'PUT':
        data = JSONParser().parse(request)
        serializer = SnippetSerializer(snippet, data=data)
        if serializer.is_valid():
            serializer.save()
            return JsonResponse(serializer.data)
        return JsonResponse(serializer.errors, status=400)

    elif request.method == 'DELETE':
        snippet.delete()
        return HttpResponse(status=204)
```

最后，我们把这些联系起来。创建```snippets/urls.py```文件：

```python
from django.conf.urls import url
from snippets import views

urlpatterns = [
    url(r'^snippets/$', views.snippet_list),
    url(r'^snippets/(?P<pk>[0-9]+)/$', views.snippet_detail),
]
```

我们还需要在```tutorial/urls.py```文件中连接项目的urlconf，引用我们应用程序的URL。

```python
from django.conf.urls import url, include

urlpatterns = [
    url(r'^', include('snippets.urls')),
]
```

值得注意的是，我们目前还没有处理好几个边缘案例。如果我们发送的是畸形的```json```，或者如果一个请求是用视图不处理的方法进行的，那么我们最终会收到500“服务器错误”响应。尽管如此，这还是要做的。



## 测试我们第一个Web API
现在我们启动一个服务。
退出shell

```python
quit()
```

并启动Django的服务器。

```python
python manage.py runserver

Validating models...

0 errors found
Django version 1.11, using settings 'tutorial.settings'
Development server is running at http://127.0.0.1:8000/
Quit the server with CONTROL-C.
```

在另一个终端窗口中，我们可以测试服务器。

我们可以使用curl或httpie来测试我们的API 。Httpie是用Python编写的用户友好的http客户端。让我们安装。

您可以使用pip安装httpie：

```python
pip install httpie
```

最后，我们可以得到所有snippets的列表：

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

或者我们可以通过引用snippet的id来get一个对应的实例：

```python
http http://127.0.0.1:8000/snippets/2/

HTTP/1.1 200 OK
...
{
  "id": 2,
  "title": "",
  "code": "print \"hello, world\"\n",
  "linenos": false,
  "language": "python",
  "style": "friendly"
}
```

同样，您可以通过在Web浏览器中访问URL来显示相同​​的json。


## 我们现在在哪
目前为止，我们已经有了一个序列化API，它与Django的Forms API以及一些常规的Django视图非常相似。

我们的API视图目前不做其他的事情，除了服务json响应之外，还有一些我们仍然希望清理的错误处理边缘案例，但这是一个正常运行的Web API。

我们将看到此教程第2部分```Requests & Responses```。
