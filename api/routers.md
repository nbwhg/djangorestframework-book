## 路由器Routers

> 资源路由（resource routing）允许我们为资源式控制器快速声明所有常见路由。只需一行代码即可完成资源路由的声明，无需为 index、show、new、edit、create、update 和 destroy 动作分别声明路由。  
—— Ruby on Rails文档

点击此处, [查看文档](http://edgeguides.rubyonrails.org/routing.html)

一些Web框架比如Rails提供了自动确定应该如何为应用程序的URLs映射到逻辑处理功能来处理传入的请求。

REST framework为Django增加了自动路由的功能，并且为你提供了一个简单的、快速的和一致的方式来根据你的视图逻辑设置一组URLs。

#### 用法

这是一个简单的URL配置，使用了```SimpleRouter```类：

```python
from rest_framework import routers

router = routers.SimpleRouter()
router.register(r'users', UserViewSet)
router.register(r'accounts', AccountViewSet)
urlpatterns = router.urls
```

方法```register()```必须接受两个参数：

- ```prefix```： 这组路由的URL前缀。
- ```viewset```： 对应的```viewset```类。

当然，你还可以指定一个额外的可选参数：

- ```base_name```： 基于这个来确定创建的URL的名称。如果没有设置该参数，那么会自动的根据```viewset```中的```queryset```属性自动生成。如果```viewset```中没有定义```queryset```属性，那么在注册路由的时候必须指定该属性。

源码位置: ```rest_framework.routers.SimpleRouter```： 

```python
def get_default_base_name(self, viewset):
    """
    If `base_name` is not specified, attempt to automatically determine
    it from the viewset.
    """
    queryset = getattr(viewset, 'queryset', None)

    assert queryset is not None, '`base_name` argument not specified, and could ' \
        'not automatically determine the name from the viewset, as ' \
        'it does not have a `.queryset` attribute.'

    return queryset.model._meta.object_name.lower()
```

提示： ```base_name```是根据```viewset```中的```queryset```属性来得到当前操作的模型的名称(小写)

提示： ```base_name```最后会自动生成```name={basename}-list```，```name={basename}-detail```和```name='{basename}-{methodnamehyphen}'```。

上面的示例将产生以下的URL模式：

- URL pattern： ```^users/$``` 名称： ```'user-list'```
- URL pattern: ```^users/{pk}/$``` 名称： ```'user-detail'```
- URL pattern: ```^accounts/$``` 名称： ```'account-list'```
- URL pattern: ```^accounts/{pk}/$``` 名称： ```'account-detail'```

---

注意： ```base_name```参数被用来指定```view name pattern```初始化的一部分。 在上面的例子中， ```user```或者```account```就代表这一部分。

通常来说我们不需要指定```base_name```参数。但是，如果你再```viewset```中自定义了```get_queryset```方法，那么```viewset```将不会再由```.queryset```属性。 此时， 如果你尝试去注册你的```viewset```，那么会得到如下错误：

```python
'base_name' argument not specified, and could not automatically determine the name from the viewset, as it does not have a '.queryset' attribute.
```

这个错误的意思是，当不能自动确定你的模型的名称的时候，当你注册```viewset```的时候，你需要显示的指定```base_name```参数。

---

##### 在Django的根URL中 包含 REST framework URLs

对于```router```的实例的```.urls```属性是一个简单的、标准的列表, 元素是```URL patterns```。所以说，就可以有多种风格将这些路由添加到你的Django根路由。

比如，你可以添加```router.urls```到现有的list：

```python
router = routers.SimpleRouter()
router.register(r'users', UserViewSet)
router.register(r'accounts', AccountViewSet)

urlpatterns = [
    url(r'^forgot-password/$', ForgotPasswordFormView.as_view()),
]

urlpatterns += router.urls
```

或者，你还可以使用Django的```include```函数， 比如：

```python
urlpatterns = [
    url(r'^forgot-password/$', ForgotPasswordFormView.as_view()),
    url(r'^', include(router.urls)),
]
```

当然也可以指定命名空间：

```python
urlpatterns = [
    url(r'^forgot-password/$', ForgotPasswordFormView.as_view()),
    url(r'^api/', include(router.urls, namespace='api')),
]
```

如果你使用了```hyperlinked serializers```和 ```url 命名空间``` ，那么你需要确定```serializers```中的```view_name```正确的反应了你的命名空间。 在上面的例子中，你需要在你的序列化```user detail```视图的超链接字段中指定参数```view_name='api:user-detail'```。

```python
class UserSerializer(serializers.HyperlinkedModelSerializer):
    # 如果在project/urls.py中引用API的话就不需要设定, 直接使用 view_name="model-detail"
    # 如果在project/urls.py中定义了app01/urls.py的namespace为app01，那么这里就必须定义，而且必须是view_name="app01:model-detail"这种格式
    url = serializers.HyperlinkedIdentityField(view_name="user-detail")
```

##### 设置额外的 链接地址 和 动作

在```viewset```中的任何方法，只要加上装饰器```@detail_route```或者是```@list_route```之后，就会被路由。比如，在```UserViewSet```中给定一个方法：

```python
from myapp.permissions import IsAdminOrIsSelf
from rest_framework.decorators import detail_route

class UserViewSet(ModelViewSet):
    ...

    @detail_route(methods=['post'], permission_classes=[IsAdminOrIsSelf])
    def set_password(self, request, pk=None):
        ...
```

以下URL pattern将会生成:

- URL pattern: ```^users/{pk}/set_password/$``` Name： ```'user-set-password'```

如果你不想为你的自定义的动作使用默认的规则生成URL， 你可以使用```url_path```参数定制URL，来代替它。

比如，你想要改变我们自定义的动作的URL 为 ```^users/{pk}/change-password/$```， 你可以这样做：

```python
from myapp.permissions import IsAdminOrIsSelf
from rest_framework.decorators import detail_route

class UserViewSet(ModelViewSet):
    ...

    @detail_route(methods=['post'], permission_classes=[IsAdminOrIsSelf], url_path='change-password')
    def set_password(self, request, pk=None):
        ...
```

上面的示例，将会生成如下的URL pattern：

- URL pattern: ```^users/{pk}/change-password/$``` Name： ```'user-change-password'```

如果，你想要改变你自定义动作的默认的 名字， 你可以使用```url_name```参数来定制它。

比如， 如果你想要为你的自定义动作，改变名称为 ```'user-change-password'```， 你可以这样：

```python
from myapp.permissions import IsAdminOrIsSelf
from rest_framework.decorators import detail_route

class UserViewSet(ModelViewSet):
    ...

    @detail_route(methods=['post'], permission_classes=[IsAdminOrIsSelf], url_name='change-password')
    def set_password(self, request, pk=None):
        ...
```

上面的示例，将会生成如下的URL pattern：

- URL pattern： ```^users/{pk}/set_password/$``` Name： ```'user-change-password'```

当然你还可以同时使用```url_path```和```url_name```参数来生成URL。

关于```ViewSet```的文档，[点击这里查看](./viewsets.md)。

---

<br />
<br />
<br />
<br />

### API 指南

#### ```SimpleRouter```类

这个路由实例包含了标准的```list```, ```create```, ```retrieve```, ```update```, ```partial_update```和```destroy```路由动作。在```viewset```中使用```@detail_route```和```@list_route```装饰器的额外的方法也可以被路由。

<table>
<tr><th>URL 风格</th><th>HTTP 方法</th><th>动作</th><th>URL 名称</th></tr>
<tr><td rowspan="2">{prefix}/</td><td>GET</td><td>list</td><td rowspan="2">{basename}-list</td></tr>
<tr><td>POST</td><td>create</td></tr>
<tr><td>{prefix}/{methodname}/</td><td>GET, 或者通过方法的参数指定</td><td>使用`@list_route`装饰器的函数</td><td>{basename}-{methodname}</td></tr>
<tr><td rowspan="4">{prefix}/{lookup}/</td><td>GET</td><td>retrieve</td><td rowspan="4">{basename}-detail</td></tr>
<tr><td>PUT</td><td>update</td></tr>
<tr><td>PATCH</td><td>partial_update</td></tr>
<tr><td>DELETE</td><td>destroy</td></tr>
<tr><td>{prefix}/{lookup}/{methodname}/</td><td>GET, 或者通过方法的参数指定</td><td>使用`@detail_route`装饰器的函数</td><td>{basename}-{methodname}</td></tr>
</table>

默认情况下，使用```SimpleRouter```创建的URL的末尾都会追加一个```/```， 可以通过在初始化路由实例的时候，指定参数```trailing_slash```为```False```可以修改这个行为，比如：

```python
router = SimpleRouter(trailing_slash=False)
```

在URL的末尾带有```/```是Django的常规的行为，但是这种行为在其他的框架中并没有。选择使用哪种风格的URL在很大程度上是一个偏好问题，尽管一些JavaScript框架可能会期望特定的路由风格。

路由器会通过匹配任何除了```/```和```.```的字符来进行查找。如果要设置更加宽松或者更加严格的匹配规则，可以在```viewset```中设置```lookup_value_regex```属性。比如，比如你可以将查找设置为有效的UUIDs：

```python
class MyModelViewSet(mixins.RetrieveModelMixin, viewsets.GenericViewSet):
    lookup_field = 'my_model_id'
    lookup_value_regex = '[0-9a-f]{32}'
```

#### ```DefaultRouter```类

这个路由器，跟上面的```SimpleRouter```基本上是一样的。但是这个路由器额外的包含了一个默认的API根视图，这个视图会返回一个包含所有的```list view```的超链接。它还可以为可选的后缀为```.json```的风格生成路由。

<table>
<tr><th>URL 风格</th><th>HTTP 方法</th><th>动作</th><th>URL 名称</th></tr>
<tr><td>[.format]</td><td>GET</td><td>自动生成的根视图</td><td>api-root</td></tr>
<tr><td rowspan="2">{prefix}/[.format]</td><td>GET</td><td>list</td><td rowspan="2">{basename}-list</td></tr>
<tr><td>POST</td><td>create</td></tr>
<tr><td>{prefix}/{methodname}/[.format]</td><td>GET, 或者通过方法的参数指定</td><td>使用`@list_route`装饰器的函数</td><td>{basename}-{methodname}</td></tr>
<tr><td rowspan="4">{prefix}/{lookup}/[.format]</td><td>GET</td><td>retrieve</td><td rowspan="4">{basename}-detail</td></tr>
<tr><td>PUT</td><td>update</td></tr>
<tr><td>PATCH</td><td>partial_update</td></tr>
<tr><td>DELETE</td><td>destroy</td></tr>
<tr><td>{prefix}/{lookup}/{methodname}/[.format]</td><td>GET, 或者通过方法的参数指定</td><td>使用`@detail_route`装饰器的函数</td><td>{basename}-{methodname}</td></tr>
</table>

和```SimpleRouter```一样，会在URL末尾追加```/```， 然后可以通过初始化路由器的时候，修改参数来改变：

```python
router = DefaultRouter(trailing_slash=False)
```

---

<br />
<br />
<br />
<br />

### 自定义路由

实现自定义的路由，并不是经常要做的事情，但是如果你对API的URL结构有特殊的需求，那么这块就会非常的有用。这样，你就可以将URL以可重用的方式封装，以确保不会显式的编写URL pattern。

最简答的实现自定义路由的方式是，以现有的路由器类来自定义一个路由器之类。```.routes```属性是用来映射URL pattern和每个```viewset```视图的模板。```.routes```属性是个包含```Router``命名元组的列表。

这个```Route```命名元组的参数解释如下：

- ```url```：一个字符串，表示要路由的URL。可能包含以下几个格式化字符串：
    - ```{prefix}```: 这组路由使用的URL前缀
    - ```{lookup}```: 用于匹配单个实例的查找字段
    - ```{trailing_slash}```: 可能为```/```或者是空字符串，依赖于```trailing_slash```参数
- ```mapping```: HTTP方法名称和```view```方法的映射关系
- ```name```: 可以被Django中的```reverse```来调用的URL的名称。可能包含以下格式化字符串：
    - ```{basename}```: URL名字是基于这个来创建的。 具体参考[viewset中设置basename](./viewsets.md)
- ```initkwargs```: 在实例化视图时应该传递的任何附加参数的字典。注意，```.suffix```参数保留用于标识viewset的类型，用于生成视图名称和面包屑导航。

源码位置： ```rest_framework.routers.SimpleRouter```

```python
class SimpleRouter(BaseRouter):

    routes = [
        # List route.
        Route(
            url=r'^{prefix}{trailing_slash}$',
            mapping={
                'get': 'list',
                'post': 'create'
            },
            name='{basename}-list',
            initkwargs={'suffix': 'List'}
        ),
        # Dynamically generated list routes.
        # Generated using @list_route decorator
        # on methods of the viewset.
        DynamicListRoute(
            url=r'^{prefix}/{methodname}{trailing_slash}$',
            name='{basename}-{methodnamehyphen}',
            initkwargs={}
        ),
        # Detail route.
        Route(
            url=r'^{prefix}/{lookup}{trailing_slash}$',
            mapping={
                'get': 'retrieve',
                'put': 'update',
                'patch': 'partial_update',
                'delete': 'destroy'
            },
            name='{basename}-detail',
            initkwargs={'suffix': 'Instance'}
        ),
        # Dynamically generated detail routes.
        # Generated using @detail_route decorator on methods of the viewset.
        DynamicDetailRoute(
            url=r'^{prefix}/{lookup}/{methodname}{trailing_slash}$',
            name='{basename}-{methodnamehyphen}',
            initkwargs={}
        ),
    ]

    def __init__(self, trailing_slash=True):
        self.trailing_slash = trailing_slash and '/' or ''
        super(SimpleRouter, self).__init__()
……省略……
```

#### 自定义动态路由

当然你还可以自定义```@list_route```和```@detail_route```装饰器应该被如何路由。它们都包含在```.routes```列表的```DynamicListRoute```和```DynamicDetailRoute```的命名元组中。

参数解释如下：

- ```url```: 表示被路由的URL字符串。
- ```name```: 可以被Django中的```reverse```来调用的URL的名称。
- ```initkwargs```: 在实例化视图时应该传递的任何附加参数的字典。

源代码，看上一小节。

#### 案例

在下面的例子中我们将仅仅会路由```list```和```retrieve```动作，并且不会遵循Django的```/```惯例：

```python
from rest_framework.routers import Route, DynamicDetailRoute, SimpleRouter

class CustomReadOnlyRouter(SimpleRouter):
    """
    A router for read-only APIs, which doesn't use trailing slashes.
    """
    routes = [
        Route(
            url=r'^{prefix}$',
            mapping={'get': 'list'},
            name='{basename}-list',
            initkwargs={'suffix': 'List'}
        ),
        Route(
            url=r'^{prefix}/{lookup}$',
            mapping={'get': 'retrieve'},
            name='{basename}-detail',
            initkwargs={'suffix': 'Detail'}
        ),
        DynamicDetailRoute(
            url=r'^{prefix}/{lookup}/{methodnamehyphen}$',
            name='{basename}-{methodnamehyphen}',
            initkwargs={}
        )
    ]
```

接下来，让我们看看，我们的```CustomReadOnlyRouter```将会为我们简单的```viewset```生成怎么样的路由。

首先在```view.py```中，代码如下：

```python
class UserViewSet(viewsets.ReadOnlyModelViewSet):
    """
    viewset 默认提供了标准的动作, 这里我们只定义额外的动作
    """
    queryset = User.objects.all()
    serializer_class = UserSerializer
    lookup_field = 'username'

    @detail_route()
    def group_names(self, request, pk=None):
        """
        返回属于给定用户的所有组的名字的列表.
        """
        user = self.get_object()
        groups = user.groups.all()
        return Response([group.name for group in groups])
```

在```urls.py```中，代码如下：

```python
router = CustomReadOnlyRouter()
router.register('users', UserViewSet)
urlpatterns = router.urls
```

最后，会生成如下的映射关系：

URL | HTTP 方法 | 动作 | URL 名称
--- | --- | --- | ---
/users| GET | list | user-list
/users/{username} | GET | retrieve | user-detail
/users/{username}/group-names | GET | group_names | user-group-names

更多的其他属性的设置，查看```SimpleRouter```源代码。

#### 更加高级自定义路由

如果你想要提供一个完整的自定义的行为，你可以继承```BaseRouter```类，并且要重写```get_urls(self)```方法。这个方法应该检查注册的```viewset```并返回一个URL pattern列表。可以通过访问```self.registry```来检查注册的```prefix```(前缀), ```viewsets```(视图集) 和 ```basename```。

你可能想要覆盖```get_default_base_name(self, viewset)```方法，或者在向路由器注册的```ViewSet```中始终显式的设置```base_name```参数。


源代码位置: ```rest_framework.routers.SimpleRouter```

```python
class BaseRouter(object):
    def __init__(self):
        self.registry = []

    def register(self, prefix, viewset, base_name=None):
        if base_name is None:
            base_name = self.get_default_base_name(viewset)
        self.registry.append((prefix, viewset, base_name))
……省略……


class SimpleRouter(BaseRouter):
……省略……
    def get_default_base_name(self, viewset):
        """
        如果 `base_name` 没有指定, 就会尝试自动去确定它,
        从viewset中.
        """
        queryset = getattr(viewset, 'queryset', None)

        assert queryset is not None, '`base_name` argument not specified, and could ' \
            'not automatically determine the name from the viewset, as ' \
            'it does not have a `.queryset` attribute.'

        return queryset.model._meta.object_name.lower()
    ……省略……
    def get_urls(self):
        """
        使用已经注册的 viewsets 来生成 URL patterns 列表.
        """
        ret = []

        for prefix, viewset, basename in self.registry:
            lookup = self.get_lookup_regex(viewset)
            routes = self.get_routes(viewset)

            for route in routes:

                # Only actions which actually exist on the viewset will be bound
                mapping = self.get_method_map(viewset, route.mapping)
                if not mapping:
                    continue

                # Build the url pattern
                regex = route.url.format(
                    prefix=prefix,
                    lookup=lookup,
                    trailing_slash=self.trailing_slash
                )

                # If there is no prefix, the first part of the url is probably
                #   controlled by project's urls.py and the router is in an app,
                #   so a slash in the beginning will (A) cause Django to give
                #   warnings and (B) generate URLS that will require using '//'.
                if not prefix and regex[:2] == '^/':
                    regex = '^' + regex[2:]

                initkwargs = route.initkwargs.copy()
                initkwargs.update({
                    'basename': basename,
                })

                view = viewset.as_view(mapping, **initkwargs)
                name = route.name.format(basename=basename)
                ret.append(url(regex, view, name=name))   ## 看这里的最后的结果就是Django的URL pattern

        return ret
```

---

<br />
<br />
<br />
<br />

### 第三方包

以下第三方包也可以用。

#### DRF Nested Routers

第三方包[drf-nested-routers package](https://github.com/alanjds/drf-nested-routers) 用于处理嵌套资源的 路由 和 关系字段。

#### ModelRouter(wq.db.rest)

第三方包[wq.db package](https://wq.io/wq.db)提供了一个高级的[```ModelRouter```](https://wq.io/1.0/docs/router)类。

#### DRF-extensions

第三方包[DRF-extensions](https://chibisov.github.io/drf-extensions/docs/)包提供用于创建[嵌套视图集的路由器](https://chibisov.github.io/drf-extensions/docs/#nested-routes)，具有[可定制端点名称](https://chibisov.github.io/drf-extensions/docs/#controller-endpoint-name)的[集合级控制器](https://chibisov.github.io/drf-extensions/docs/#collection-level-controllers)。

---

参考链接：

- http://edgeguides.rubyonrails.org/routing.html
- https://github.com/alanjds/drf-nested-routers
- https://wq.io/1.0/docs/router
- https://chibisov.github.io/drf-extensions/docs/#nested-routes
- https://chibisov.github.io/drf-extensions/docs/#collection-level-controllers
- https://chibisov.github.io/drf-extensions/docs/#controller-endpoint-name
