# 视图集合和路由

REST框架包含一个用于处理的抽象概念```ViewSets```，它允许开发人员专注于对API的状态和交互进行建模，并根据通用约定自动处理URL构造。

```ViewSet```类与类几乎相同```View```，除了它们提供例如```read```或```update```的操作，而不是例如``get``` or ``put``的方法处理程序。

一个```视图集合```类只在最后时刻绑定到一组方法来处理程序，当它被实例化为一组视图时，通常通过使用一个```Router```类来为您处理定义URL conf的复杂性。

## 重构视图
我们来看看我们当前的一组视图，并将它们重构为视图集。

首先，让我们将```UserList```和```UserDetail```视图重构为单个```UserViewSet```。我们可以删除这两个视图，并用一个类替换它们：

```python
from rest_framework import viewsets

class UserViewSet(viewsets.ReadOnlyModelViewSet):
    """
    这个视图集合会自动提供“list”和“detail”操作。
    """
    queryset = User.objects.all()
    serializer_class = UserSerializer
```    

这里我们使用这个```ReadOnlyModelViewSet```类来自动提供默认的“只读”操作。我们仍然像在使用常规视图时那样设置```queryset```和```serializer_class```属性，但是我们不再需要向两个单独的类提供相同的信息。
接下来我们要更换```SnippetList```，```SnippetDetail```和```SnippetHighlight```视图类。我们可以删除三个视图，并再次用一个类替换它们。

```python
from rest_framework.decorators import detail_route
from rest_framework.response import Response

class SnippetViewSet(viewsets.ModelViewSet):
    """
    该视图自动提供 `list`, `create`, `retrieve`,
    `update` and `destroy` actions.

    Additionally we also provide an extra `highlight` action.
    """
    queryset = Snippet.objects.all()
    serializer_class = SnippetSerializer
    permission_classes = (permissions.IsAuthenticatedOrReadOnly,
                          IsOwnerOrReadOnly,)

    @detail_route(renderer_classes=[renderers.StaticHTMLRenderer])
    def highlight(self, request, *args, **kwargs):
        snippet = self.get_object()
        return Response(snippet.highlighted)

    def perform_create(self, serializer):
        serializer.save(owner=self.request.user)
```

这次我们使用这个```ModelViewSet```类来获得完整的默认读写操作。

请注意，我们也使用```@detail_route```装饰器来创建一个名为的自定义操作```highlight```。这个装饰器可以用来添加任何不符合标准```create/ update/ delete```风格的自定义端点。

使用```@detail_route``` 装饰器的自定义操作将在默认情况下响应GET请求。如果我们想要一个响应POST请求的操作，我们可以使用```methods```参数。

自定义操作的URL默认取决于方法名称本身。如果你想改变构造url的方式，你可以包含url_path作为装饰器关键字参数。


## 显式地将ViewSets绑定到URL


当我们定义URLConf时，处理程序方法只绑定到操作。为了查看引擎盖下发生了什么，让我们首先从我们的视图中显式地创建一组视图。

在```snippets/urls.py```文件中，我们将我们的```ViewSet```类绑定到一组具体的视图中。

```python
from snippets.views import SnippetViewSet, UserViewSet, api_root
from rest_framework import renderers

snippet_list = SnippetViewSet.as_view({
    'get': 'list',
    'post': 'create'
})
snippet_detail = SnippetViewSet.as_view({
    'get': 'retrieve',
    'put': 'update',
    'patch': 'partial_update',
    'delete': 'destroy'
})
snippet_highlight = SnippetViewSet.as_view({
    'get': 'highlight'
}, renderer_classes=[renderers.StaticHTMLRenderer])
user_list = UserViewSet.as_view({
    'get': 'list'
})
user_detail = UserViewSet.as_view({
    'get': 'retrieve'
})
```


请注意，我们是如何从每个```ViewSet```类创建多个视图，方法是将http方法绑定到每个视图所需的操作。

现在我们已经将资源绑定到具体的视图中，我们可以像往常一样从```URL conf```注册视图。

```python
urlpatterns = format_suffix_patterns([
    url(r'^$', api_root),
    url(r'^snippets/$', snippet_list, name='snippet-list'),
    url(r'^snippets/(?P<pk>[0-9]+)/$', snippet_detail, name='snippet-detail'),
    url(r'^snippets/(?P<pk>[0-9]+)/highlight/$', snippet_highlight, name='snippet-highlight'),
    url(r'^users/$', user_list, name='user-list'),
    url(r'^users/(?P<pk>[0-9]+)/$', user_detail, name='user-detail')
])  
```

## 使用路由器
因为我们使用的是```ViewSet```类而不是```View```类，所以我们实际上不需要自己设计URL。将资源连接到视图和URL可以使用```Router```类自动处理。我们所需要做的就是用路由器注册适当的视图集，然后让其完成。

这是我们重新连接的```snippets/urls.py```文件。

```python
from django.conf.urls import url, include
from rest_framework.routers import DefaultRouter
from snippets import views

# Create a router and register our viewsets with it.
router = DefaultRouter()
router.register(r'snippets', views.SnippetViewSet)
router.register(r'users', views.UserViewSet)

# The API URLs are now determined automatically by the router.
urlpatterns = [
    url(r'^', include(router.urls))
]

```
向路由器注册视图类似于提供urlpattern。我们包含两个参数 - 视图的URL前缀和视图本身。

```DefaultRouter```我们使用的类也为我们自动创建了API根视图，因此我们现在可以```api_root```从```views```模块中删除该方法。


## 视图与视图集之间的权衡
使用viewsets是一个非常有用的抽象。它有助于确保URL约定在您的API中是一致的，最小化您需要编写的代码量，并允许您集中精力于API提供的交互和表示，而不是URL conf的细节。

这并不意味着它总是正确的方法。在使用基于类的视图而不是基于功能的视图时，有一组类似的折衷方案。使用viewset比单独构建视图更不明确。

在本教程的第7部分中，我们将介绍如何添加API模式，以及如何使用客户端库或命令行工具与我们的API进行交互。




      
        