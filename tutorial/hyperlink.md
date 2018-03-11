# 关系和超链接

目前，我们的API中的关系用主键表示。在本教程的这一部分中，我们将通过使用超链接来提高API的内聚性和可发现性

# 为我们的API的根创建一个endpoint

现在我们有```'snippets'```和```'users'```的端点，但是我们没有一个入口指向我们的API。要创建一个，我们将使用一个常规的基于函数的视图和我们之前介绍的装饰器```@api_view```。在你的```snippets/views.py```添加中：

```python
from rest_framework.decorators import api_view
from rest_framework.response import Response
from rest_framework.reverse import reverse


@api_view(['GET'])
def api_root(request, format=None):
    return Response({
        'users': reverse('user-list', request=request, format=format),
        'snippets': reverse('snippet-list', request=request, format=format)
    })
```

这里应该注意两件事。首先，我们使用REST框架的```reverse```函数来返回完全限定的URL; 其次，URL模式通过方便的名称来标识，我们稍后会在我们的网站中声明```snippets/urls.py```。

# 为高亮的snippets创建一个endpoint

我们的pastebin API仍然存在的另一个显而易见的问题是突出端点的代码。

与其他所有API端点不同，我们不想使用JSON，而只是呈现HTML表示。REST框架提供了两种HTML呈现器，一种用于处理使用模板展现的HTML，另一种用于处理预展现的HTML。第二个渲染器是我们希望用于此endpoint的渲染器。

在创建代码高亮显示视图时，我们需要考虑的另一件事是没有现有的具体的通用视图可以使用。我们不是返回一个对象实例，而是返回一个对象实例的属性。

我们不使用具体的通用视图，而是使用基类来表示实例，并创建我们自己的```.get()```方法。在你的```snippets/views.py```添加中：

```python
from rest_framework import renderers
from rest_framework.response import Response

class SnippetHighlight(generics.GenericAPIView):
    queryset = Snippet.objects.all()
    renderer_classes = (renderers.StaticHTMLRenderer,)

    def get(self, request, *args, **kwargs):
        snippet = self.get_object()
        return Response(snippet.highlighted)
```


像之前一样，我们需要将我们创建的新视图添加到我们的```URLconf```中。我们将在我们的新API根中添加一个网址格式```snippets/urls.py```：

```python
url(r'^$', views.api_root),
```
然后为代码高亮添加一个网址格式：

```python
url(r'^snippets/(?P<pk>[0-9]+)/highlight/$', views.SnippetHighlight.as_view()),
```



## 超链接我们的API
处理实体之间的关系是```Web API```设计中更具挑战性的方面之一。我们可以选择代表一种关系的方式有很多种：

1. 使用主键。
2. 在实体之间使用超链接。
3. 在相关实体上使用唯一标识段落字段。
4. 使用相关实体的默认字符串表示形式。
5. 将相关实体嵌套在父代表中。
6. 一些其他自定义表示。


REST框架支持这些所有的样式，并且可以跨前向或反向关系应用它们，或者将其应用到自定义管理器（如通用外键）中。
在这种情况下，我们希望在实体之间使用超链接样式。为了做到这一点，我们将修改序列化器来扩展```HyperlinkedModelSerializer```而不是现有的序列化器```ModelSerializer```。

```HyperlinkedModelSerializer```和```ModelSerializer```有以下区别：

* 它id默认不包含该字段。
* 它包括一个url字段，使用HyperlinkedIdentityField。
* 关系使用HyperlinkedRelatedField，而不是PrimaryKeyRelatedField。


我们可以轻松地重写我们现有的序列化程序来使用超链接。在你的```snippets/serializers.py```中添加：

```python
class SnippetSerializer(serializers.HyperlinkedModelSerializer):
    owner = serializers.ReadOnlyField(source='owner.username')
    highlight = serializers.HyperlinkedIdentityField(view_name='snippet-highlight', format='html')

    class Meta:
        model = Snippet
        fields = ('url', 'id', 'highlight', 'owner',
                  'title', 'code', 'linenos', 'language', 'style')


class UserSerializer(serializers.HyperlinkedModelSerializer):
    snippets = serializers.HyperlinkedRelatedField(many=True, view_name='snippet-detail', read_only=True)

    class Meta:
        model = User
        fields = ('url', 'id', 'username', 'snippets')
        
```


请注意，我们还添加了一个新```'highlight'```字段。该字段与```url```字段的类型相同，只是它指向的是```'snippet-highlight'```url模式，而不是```'snippet-detail'```url模式。

由于我们已经包含格式后缀的URL ```'.json'```，我们还需要在```highlight```字段上指明它返回的任何格式后缀超链接应使用```'.html'```后缀。



## 确保我们的URL模式被命名
如果我们要有超链接的API，我们需要确保我们命名我们的URL模式。我们来看看我们需要命名的URL模式。

* 我们的API的根源是'user-list'和'snippet-list'。
* 我们的片段序列化程序包含一个指向的字段'snippet-highlight'。
* 我们的用户序列化程序包含一个指向的字段'snippet-detail'。
* 我们的片段和用户序列化程序包含'url'默认情况下会引用的字段，'{model_name}-detail'在这种情况下将是'snippet-detail'和'user-detail'。
* 将所有这些名称添加到我们的URLconf后，我们的最终snippets/urls.py文件应该如下所示：


```python
from django.conf.urls import url, include
from rest_framework.urlpatterns import format_suffix_patterns
from snippets import views

# API endpoints
urlpatterns = format_suffix_patterns([
    url(r'^$', views.api_root),
    url(r'^snippets/$',
        views.SnippetList.as_view(),
        name='snippet-list'),
    url(r'^snippets/(?P<pk>[0-9]+)/$',
        views.SnippetDetail.as_view(),
        name='snippet-detail'),
    url(r'^snippets/(?P<pk>[0-9]+)/highlight/$',
        views.SnippetHighlight.as_view(),
        name='snippet-highlight'),
    url(r'^users/$',
        views.UserList.as_view(),
        name='user-list'),
    url(r'^users/(?P<pk>[0-9]+)/$',
        views.UserDetail.as_view(),
        name='user-detail')
])
```


## 添加分页
用户和snippets的列表视图可能会返回很多实例，所以我们确实要确保对结果进行分页，并允许API客户端遍历每个单独的页面。

通过tutorial/settings.py稍微修改我们的文件，我们可以更改默认列表样式以使用分页。添加以下设置：

```python
REST_FRAMEWORK = {
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.PageNumberPagination',
    'PAGE_SIZE': 10
}
```

请注意，REST框架中的设置都被命名为一个单独的字典设置REST_FRAMEWORK，这有助于使它们与其他项目设置保持良好的分离。

如果我们也需要，我们也可以自定义分页样式，但在这种情况下，我们会坚持默认。

## 浏览API
如果我们打开一个浏览器并导航到可浏览的API，您会发现现在您可以通过简单的链接访问API。

您还可以看到片段实例上的“highlight”链接，这会将您带到突出显示的代码HTML表示。

在本教程的第```6```部分中，我们将介绍如何使用```ViewSets```和`routers`来减少构建API所需的代码量。
