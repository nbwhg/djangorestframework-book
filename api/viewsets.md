## 视图组Viewsets

> 路由器决定使用哪个控制器处理请求后，控制器负责解析请求，生成相应的输出。  
—— Ruby on Rails Documentation

点击此处[查看文档](http://guides.rubyonrails.org/action_controller_overview.html)。

Django REST framework允许您将一组相关视图的逻辑组合到一个名为```ViewSet```的类中。在其他框架中，您可能会发现概念上类似的实现，名为"Resources"或"Controllers"。

一个```ViewSet```类，只是一个简单的基于类的视图，它不提供任何处理方法，比如```.get()```或者```.post()```，而是提供了动作方法来代替，比如```.list()```和```.create()```。

一个```ViewSet```类，会使用```.as_view()```方法将处理方法和动作方法绑定。

相关的源代码位置在: ```rest_framework.viewsets.ViewSetMixin```

通常，不是在urlconf中通过viewset来显示的注册你的视图，而是将viewset注册到router类中，这将会自动的为您确定需要的urlconf。


#### 例子

让我们来定义一个简单的viewset， 主要目的是可以列出或者获取系统中的用户：

```python
from django.contrib.auth.models import User
from django.shortcuts import get_object_or_404
from myapps.serializers import UserSerializer
from rest_framework import viewsets
from rest_framework.response import Response

class UserViewSet(viewsets.ViewSet):
    """
    A simple ViewSet for listing or retrieving users.
    """
    def list(self, request):
        queryset = User.objects.all()
        serializer = UserSerializer(queryset, many=True)
        return Response(serializer.data)

    def retrieve(self, request, pk=None):
        queryset = User.objects.all()
        user = get_object_or_404(queryset, pk=pk)
        serializer = UserSerializer(user)
        return Response(serializer.data)
```

如果有必要，你可以绑定这个viewset到两个单独的视图里面，比如这样：

```python
user_list = UserViewSet.as_view({'get': 'list'})
user_detail = UserViewSet.as_view({'get': 'retrieve'})
```

通常我们不会这样做，而是注册viewset到router里面，这将会自动生成urlconf：

```python
from myapp.views import UserViewSet
from rest_framework.routers import DefaultRouter

router = DefaultRouter()
router.register(r'users', UserViewSet, base_name='user')
urlpatterns = router.urls
```

如果你不是编写你自己的```viewsets```，而是你要使用一些现有的基类提供的一组默认功能，你完全可以这样做：

```python
class UserViewSet(viewsets.ModelViewSet):
    """
    A viewset for viewing and editing user instances.
    """
    serializer_class = UserSerializer
    queryset = User.objects.all()
```

使用```ViewSet```类相比使用```View```类，有两个主要的优点：

- 重复的逻辑可以合并到一个类中。在上面的例子中，我们只需要制定一次```queryset```，就可以在多个视图中使用了。
- 通过使用routers， 我们不再需要自己处理URL配置。

这两个都有一个这种。使用常规的视图和URL配置，会更加清晰明确，并且更容易控制。但是如果想要快速的运行，使用ViewSets； 或者您有大量的API并且希望始终有一致的URL配置，那么ViewSets就会很方便。

#### ViewSet 包含的动作

REST framework中默认的路由器将会为 create/retrieve/update/destroy 风格的动作提供一系列的路由，如下：

```python
class UserViewSet(viewsets.ViewSet):
    """
    这是一个viewset 的演示,
    标准动作 将会 被 路由器处理.

    如果你使用格式后缀,
    请在每个动作中确保,
    包含了`format=None`关键字参数.
    """

    def list(self, request):
        pass

    def create(self, request):
        pass

    def retrieve(self, request, pk=None):
        pass

    def update(self, request, pk=None):
        pass

    def partial_update(self, request, pk=None):
        pass

    def destroy(self, request, pk=None):
        pass
```

在调度(```dispatch```)动作的期间，当前的动作可以通过```.action```属性获取。你可以通过检查```.action```来调整当前动作的行为。

比如，你可以在限制权限为处理```list```动作之外的其他：

```python
def get_permissions(self):
    """
    Instantiates and returns the list of permissions that this view requires.
    """
    if self.action == 'list':
        permission_classes = [IsAuthenticated]
    else:
        permission_classes = [IsAdmin]
    return [permission() for permission in permission_classes]
```

#### 让额外的动作 使用路由

如果你又额外的方法需要被路由，那么你可以使用```@detail_route```或者```@list_route```装饰器，来标记这些方法可以被路由。

装饰器```@detail_route```在URL中匹配```pk```并且适用于只需要单个实例的方法。

装饰器```@list_route```适用于需要操作一组实例的方法。

比如：

```python
from django.contrib.auth.models import User
from rest_framework import status
from rest_framework import viewsets
from rest_framework.decorators import detail_route, list_route
from rest_framework.response import Response
from myapp.serializers import UserSerializer, PasswordSerializer

class UserViewSet(viewsets.ModelViewSet):
    """
    A viewset that provides the standard actions
    """
    queryset = User.objects.all()
    serializer_class = UserSerializer

    @detail_route(methods=['post'])
    def set_password(self, request, pk=None):
        user = self.get_object()
        serializer = PasswordSerializer(data=request.data)
        if serializer.is_valid():
            user.set_password(serializer.data['password'])
            user.save()
            return Response({'status': 'password set'})
        else:
            return Response(serializer.errors,
                            status=status.HTTP_400_BAD_REQUEST)

    @list_route()
    def recent_users(self, request):
        recent_users = User.objects.all().order('-last_login')

        page = self.paginate_queryset(recent_users)
        if page is not None:
            serializer = self.get_serializer(page, many=True)
            return self.get_paginated_response(serializer.data)

        serializer = self.get_serializer(recent_users, many=True)
        return Response(serializer.data)
```

这些装饰器还可以为仅仅路由到的视图上设置一些额外的参数。比如：

```python
@detail_route(methods=['post'], permission_classes=[IsAdminOrIsSelf])
def set_password(self, request, pk=None):
    ...
```

这些装饰器，默认情况下是路由```GET```请求，但是也可以设置接收其他HTTP方法，使用```methods```参数。比如：

```python
@detail_route(methods=['post', 'delete'])
def unset_password(self, request, pk=None):
    ...
```

这两个新的动作将会在URL```^users/{pk}/set_password/$```和```^users/{pk}/unset_password/$```中可用。

#### 解析URL中的动作

如果你需要获取一个带有动作的URL，使用```.reverse_action()```方法。这是对```reverse()```的一个便捷封装，它会自动传递视图的```request```对象，并且将```url_name```预先加入```.basename```属性。

关于```reverser()```的文档，点击[查看](https://docs.djangoproject.com/en/2.0/ref/urlresolvers/)。

注意：```basename```是在```ViewSet```注册到路由的时候提供的。如果你不使用```router```，那么你必须提供```basename```参数到```.as_view()```方法。

使用前一个小结的例子：

```python
>>> view.reverse_action('set-password', args=['1'])
'http://localhost:8000/api/users/1/set_password'
```

参数```url_name```应该和装饰器```@list_route```和```@detail_route```相匹配。另外，这也可以被用于```reverse```默认的```list```和```detail```路由。

---

### API 参考

源码位置： ```rest_framework.viewsets```

#### ViewSet 类

类```ViewSet```是```APIView```的子类。所以你可以使用任何标准的属性，比如在你的ViewSet中使用```permission_classes```, ```authentication_classes```来控制API策略。

类```ViewSet```中并没有提供任何动作的实现。所以你如果要使用```ViewSet```，你需要自己定义动作的实现。

REST framework 中的源代码如下：

```python
class ViewSet(ViewSetMixin, views.APIView):
    """
    The base ViewSet class does not provide any actions by default.
    """
    pass
```

在```ViewSetMixin```和```APIView```中都没有关于动作的实现。

#### GenericViewSet 类

类```GenericViewSet```是```GenericAPIView```的子类，并且提供了默认的```get_object```和```get_queryset```方法，和其他通用的视图行为，但是不包括其他任何默认的动作。

如果要使用```GenericViewSet```类，你需要重写需要的mixin类，或者显示的定义动作的实现。

REST framework中源码实现：

```python
class GenericViewSet(ViewSetMixin, generics.GenericAPIView):
    """
    The GenericViewSet class does not provide any actions by default,
    but does include the base set of generic view behavior, such as
    the `get_object` and `get_queryset` methods.
    """
    pass
```

父类```ViewSetMixin```和```GenericAPIView```没有任何动作的实现。

#### ModelViewSet 类

类```ModelViewSet```是```GenericAPIView```的子类，并且包含了多种多样的动作的实现，通过多种多样的mixins类来提供各种行为。

类```ModelViewSet```提供的动作包括```.list()```, ```.retrieve()```, ```.create()```, ```.update()```, ```.partial_update()```和```.destroy()```。

**比如**

因为```ModelViewSet```扩展了```GenericAPIView```，你至少要提供```queryset```和```serializer_class```属性。比如：

```python
class AccountViewSet(viewsets.ModelViewSet):
    """
    A simple ViewSet for viewing and editing accounts.
    """
    queryset = Account.objects.all()
    serializer_class = AccountSerializer
    permission_classes = [IsAccountAdminOrReadOnly]
```

注意，你可以使用```GenericAPIView```类提供的任何标准的属性和方法。比如，如果你要动态的确定查询结果集，你可能会这么做：

```python
class AccountViewSet(viewsets.ModelViewSet):
    """
    A simple ViewSet for viewing and editing the accounts
    associated with the user.
    """
    serializer_class = AccountSerializer
    permission_classes = [IsAccountAdminOrReadOnly]

    def get_queryset(self):
        return self.request.user.accounts.all()
```

注意，当从```ViewSet```中删除```queryset```属性之后，与之关联的[```router```](./routers.md)将无法自动获取你Model的```base_name```， 所以你必须在注册路由的时候要自己指定```base_name```。

提示： ```base_name```是根据```viewset```中的```queryset```属性来得到当前操作的模型的名称(小写)

还要注意，尽管这个类默认提供了完整的create/list/retrieve/update/destroy操作集，但还可以通过使用标准权限类来限制可用操作。

#### ReadOnlyModelViewSet 类

类```ReadOnlyModelViewSet```类是```GenericAPIView```的之类，跟```ModelViewSet```类一样也包含多种动作的实现，但是不同的是，这里只包含只读的动作，```.list()```和```.retrieve()```。

**例子**

和```ModelViewSet```一样， 你至少要提供```queryset```和```serializer_class```属性。 比如 ：

```python
class AccountViewSet(viewsets.ReadOnlyModelViewSet):
    """
    A simple ViewSet for viewing accounts.
    """
    queryset = Account.objects.all()
    serializer_class = AccountSerializer
```

此外，同样的，你可以使用```GenericAPIView```提供的标准的属性和方法。

---

### 自定义ViewSet 基础类

你可能需要提供自定义的```ViewSet```类，并不像```ModelViewSet```类一样包含所有的动作， 或者自定义一些行为。

#### 例子

创建一个基础的viewset类，来提供```create```，```list```和```retrieve```操作，继承```GenericViewSet```，并混合需要的动作：

```python
from rest_framework import mixins

class CreateListRetrieveViewSet(mixins.CreateModelMixin,
                                mixins.ListModelMixin,
                                mixins.RetrieveModelMixin,
                                viewsets.GenericViewSet):
    """
    A viewset that provides `retrieve`, `create`, and `list` actions.

    To use it, override the class and set the `.queryset` and
    `.serializer_class` attributes.
    """
    pass
```

通过创建你自己的基础的```ViewSet```类，你可以提供常见的行为，然后在你的API中重用他们。

---

参考链接：
- http://guides.rubyonrails.org/action_controller_overview.html
- https://docs.djangoproject.com/en/2.0/ref/urlresolvers/