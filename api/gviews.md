## 通用视图Generic views

> Django 的通用视图建立在基础视图之上，用于作为经常用到的功能的快捷方式，例如显示对象的详细信息。它们提炼视图开发中常见的风格和模式并将它们抽象，这样你可以快速编写常见的视图而不用重复你自己。  
—— Django 官方文档(内置的基于类的视图API)

点击这里[查看官方文档](https://docs.djangoproject.com/en/2.0/ref/class-based-views/#base-vs-generic-views)。

基于类的视图，最大的好处之一就是他将允许你编写可以重用的行为。REST framework利用这一个有点提供了一些预先构建好的视图，这些视图提供了常用的模式。

REST framework提供的通用视图允许你快速的来构建一个和你的数据库紧密映射的一个API视图。

如果REST framework提供的通用视图不能够满足你的API需求，你可以使用更底层的```APIView```类。或者重新使用```mixins```和通用视图基类来重新编写一个你自己的可以重复使用的通用视图类。

#### 例子

通常使用通用视图类的时候，你仅仅需要设置一些类属性即可：

```python
from django.contrib.auth.models import User
from myapp.serializers import UserSerializer
from rest_framework import generics
from rest_framework.permissions import IsAdminUser

class UserList(generics.ListCreateAPIView):
    queryset = User.objects.all()
    serializer_class = UserSerializer
    permission_classes = (IsAdminUser,)
```

对于更复杂的场景，你可能还想要覆盖视图类中的各种方法，比如下面：

```python
class UserList(generics.ListCreateAPIView):
    queryset = User.objects.all()
    serializer_class = UserSerializer
    permission_classes = (IsAdminUser,)

    def list(self, request):
        # Note the use of `get_queryset()` instead of `self.queryset`
        queryset = self.get_queryset()
        serializer = UserSerializer(queryset, many=True)
        return Response(serializer.data)
```

在极简单的情况下，你可能只需要通过```.as_view()```传递给类一些属性就可以了。比如，在你的```URLconf```中可能会包含如下的代码：

```python
url(r'^/users/', ListCreateAPIView.as_view(queryset=User.objects.all(), serializer_class=UserSerializer), name='user-list')
```

---

### API 参考

#### ```GenericAPIView``` 类

源码位置： ```rest_framework.generics.GenericAPIView```

这个类是所有具名通用视图的基础类。

这个类扩展了REST framework的```APIView```类，为标准的```list```视图和```detail```视图添加了一些必要的行为。

每个提供给用户使用的具体的命名通用视图，是使用内置的```GenericAPIView```和一个或者多个的```Mixins```类组合而成的。具体的下面有介绍。

<br />
<br />
<br />
<br />

##### 属性

**基础设置**:

下面的属性，控制基础的视图行为。
- ```queryset```: —— 用于从视图返回对象。通常情况下，你必须要设置这个属性或者是重写```get_queryset()```方法。如果你重写了视图的方法，一定要记住，不能直接访问这个属性，而是要调用```get_queryset()```方法； 因为```queryset```属性，只会评估计算一次，为后续所有的请求提供结果。
- ```serializer_class```: —— 这个属性设置的类，将用于验证用户的输入并反序列化，或者序列化输出结果。通常你需要设置这个属性，或者是重写```get_serializer_class()```方法。
- ```lookup_field```: —— 这是一个模型字段，用来执行单个的模型实例的对象查看(意思是，外键查看)。默认是```'pk'```。注意，当使用超链接的API，你需要确保API视图和序列化类设置```lookup```字段，如果你需要使用一个自定义的值。
- ```lookup_url_kwarg```: —— 被应用于对象查找URL关键字参数。URLconf中应包括相应于该值的关键字参数。如果未设置，那么默认使用和```lookup_field```相同的值。

<br />
<br />

**分页设置**:

下面的属性被用于当使用```list```视图的时候来控制分页。

- ```pagination_class```: —— 用于给```list```视图结果分页。默认使用的是```settings```中的```DEFAULT_PAGINATION_CLASS```选项，选项结果是```'rest_framework.pagination.PageNumberPagination'```。 设置```pagination_class=None```将会禁用该视图的分页。

注意： 在使用```Django REST Framework 3.7.7```版本的过程中，该分页设置为```None```，也就是默认没有分页效果。

默认设置位置: ```rest_framework.settings.DEFAULTS.DEFAULT_PAGINATION_CLASS```

关于分页类的源码位置: ```rest_framework.pagination.*Pagination```

<br />
<br />

**过滤设置**:

- ```filter_backends```: —— 可以用来过滤Queryset的过滤类的一个列表。默认值是```settings```中的```DEFAULT_FILTER_BACKENDS```选项。

注意： 在使用```Django REST Framework 3.7.7```版本的过程中，该过滤设置为```()```。

<br />
<br />
<br />
<br />

##### 方法

**基础方法**:

方法1：```get_queryset(self)```

返回```list```视图所需要的```queryset```(查询结果集)，并且可以用于```detail```视图的查询基础。默认情况下返回的是```queryset```属性指定的查询结果集。

应该总是使用```get_queryset()```方法而不是直接使用```self.queryset```来获取查询结果集。因为```self.queryset```仅仅计算一次，并且这些结果被缓存用于所有后续的请求。

在某些情况下，你可能要重写这个方法，用来适应动态的请求，比如，为每个当前访问的用户返回特定的结果集。代码如下:

```python
def get_queryset(self):
    user = self.request.user
    return user.accounts.all()
```

方法2：```get_object(self)```

返回用于```detail```视图的一个实例对象。默认使用```lookup_field```参数来过滤基础查询集。

在某些情况下，你可能要重写这个方法，用来提供更复杂的行为，比如基于多个```URL kwarg```的对象查询。代码如下：

```python
def get_object(self):
    queryset = self.get_queryset()
    filter = {}
    for field in self.multiple_lookup_fields:
        filter[field] = self.kwargs[field]

    obj = get_object_or_404(queryset, **filter)
    self.check_object_permissions(self.request, obj)
    return obj
```

注意，如果你的API不包含任何对象级别的权限，你完全不需要```self.check_object_permissions```，只需要从```get_object_or_404```查找返回对象即可。

方法3：```filter_queryset(self, queryset)```

给定一个查询结果集，然后过滤器会处理该查询结果集，最后返回一个新的```queryset```查询结果集。

比如：

```python
def filter_queryset(self, queryset):
    filter_backends = (CategoryFilter,)

    if 'geo_route' in self.request.query_params:
        filter_backends = (GeoRouteFilter, CategoryFilter)
    elif 'geo_point' in self.request.query_params:
        filter_backends = (GeoPointFilter, CategoryFilter)

    for backend in list(filter_backends):
        queryset = backend().filter_queryset(self.request, queryset, view=self)

    return queryset
```

方法4：```get_serializer_class(self)```

返回用于序列化的class。默认返回的是```serializer_class```属性设定的类。

在某些情况下，你可能要重写这个方法，用来适应动态的行为，比如可以为读 和 写 操作 设定不同的序列化器， 或者为 不同的用户 设定 不同的序列化器。

比如：

```python
def get_serializer_class(self):
    if self.request.user.is_staff:
        return FullAccountSerializer
    return BasicAccountSerializer
```

<br />
<br />

**保存 和 删除  钩子**：

下面的这些方法是由```Mixins```类提供，并提供了简单的用于覆盖对象默认的 保存和删除 的行为。

- ```perform_create(self, serializer)``` —— 当保存一个新的对象实例的时候，通过```CreateModelMixin```来调用。
- ```perform_update(self, serializer)``` —— 当保存一个已有的对象实例的时候，通过```UpdateModelMixin```来调用。
- ```perform_destroy(self, instance)``` —— 当删除一个对象实例的时候，通过```DestroyModelMixin```来调用。

这些钩子对于设置请求中的隐藏属性，而这些属性又不是请求数据的一部分，是非常有用的。比如，你可能会基于请求的用户，或者基于URL的关键字参数来设置属性，比如下面这段代码：

```python
def perform_create(self, serializer):
    serializer.save(user=self.request.user)
```

这些可覆盖的点非常的多，用于在保存数据之前或者之后的各种行为。比如，在数据保存完之后可以发送确认邮件，或者日志更新等等。如下这段代码：

```python
def perform_update(self, serializer):
    instance = serializer.save()
    send_email_confirmation(user=self.request.user, modified=instance)
```

你还可以使用这些钩子来增加一些额外的验证，通过抛出```ValidationError()```异常。如果你需要在数据保存到数据库之前做一些逻辑性的验证操作，这可能是非常有用的。比如下面这段代码：

```python
def perform_create(self, serializer):
    queryset = SignupRequest.objects.filter(user=self.request.user)
    if queryset.exists():
        raise ValidationError('You have already signed up')
    serializer.save(user=self.request.user)
```

注意： 这些方法取代了```2.x```版本中的```pre_save```, ```post_save```, ```pre_delete```和```post_delete```方法，它们将不再可用。

<br />
<br />

**其他方法**: 

你不需要重写以下方法，如果你使用```GenericAPIView```在编写自定义的视图，里面可能会用到它们。

- ```get_serializer_context(self)``` —— 返回一个字典包含任意的额外上下文，来提供给序列化器。默认包含```'request'```, ```'view'```和```'format'```key。
- ```get_serializer(self, instance=None, data=None, many=False, partial=False)``` —— 返回序列化器的实例。
- ```get_paginated_response(self, data)``` —— 返回一个已经分页的```Response```对象。
- ```paginate_queryset(self, queryset)``` —— 如果需要对```queryset```来进行分页，那么会返回一个分页对象，或者当没有给视图设置分页的时候返回一个```None```。
- ```filter_queryset(self, queryset)``` —— 给定一个查询结果集，然后过滤器会处理该查询结果集，最后返回一个新的```queryset```查询结果集。

---

### Mixins 类

源码位置： ```rest_framework.mixins.*Mixin```

Mixins类用来提供基本的视图行为动作。注意，Mixins类提供的是动作方法，而不是处理方法，比如提供了```.create()```和```.list()```而不是```.post()```和```.get()```。这样做，会允许更加灵活的组合。也就是说，```GET```请求的处理方法```.get()```, 可以包含多种动作方法，比如```.create()```和```.list()```方法。

这些mixins类，可以通过```rest_framework.mixins```来导入。

#### ```ListModelMixin``` 类

提供了```.list(request, *args, **kwargs)```方法，用于实现列出查询结果集。

如果查询结果集被填充，那么将返回一个```200 OK```的响应，response body是已经序列化的查询结果。当然这个响应的结果可能是已经分页的。

#### ```CreateModelMixin``` 类

提供了```.create(request, *args, **kwargs)```方法，用于实现创建并保存一个模型实例。

如果一个对象被创建，那么会返回一个```201 Created```的响应，response body是已经序列化的对象。如果包含一个名为```url```的key，那么response header的```Location```将会填充该值。

关于```Location```，点击[此处](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Location)。

如果用于创建对象的请求数据是无效的，那么会返回一个```400 Bad Request```的响应，response body是错误详情。

#### ```RetrieveModelMixin``` 类

提供了```.retrieve(request, *args, **kwargs)```方法，用来返回一个已经存在的模型实例。

如果对象可以被取出来，那么将返回一个```200 OK```的响应，response body是已经序列化的实例。否则的会返回```404 Not Found```。

#### ```UpdateModelMixin``` 类

提供一个```.update(request, *args, **kwargs)```方法，用于实现更新并保存一个已经存在的模型实例。

在这个类里面提供了```.partial_update(request, *args, **kwargs)```方法，同样是更新，只不过该方法用于局部更新，更新字段是可选的。这将允许```PATCH```的HTTP请求。

如果对象被更新，返回一个```200 OK```的响应，response body是已经序列化的实例。

如果用于更新对象的请求数据是无效的，那么会返回一个```400 Bad Request```, response body是错误详情。

#### ```DestroyModelMixin``` 类

提供一个```.destroy(request, *args, **kwargs)```方法， 用于实现删除一个已经存在的模型实例。

如果对象被删除，那么返回```204 No Content```，否则返回```404 Not Found```。

---

### 命名视图 类

源码位置： ```rest_framework.generics.*APIView```

下面的类是具体的可以被直接使用的通用视图。如果你使用通用视图，那么下面的这些方法将足够你使用，除非你需要重写大量的行为。

这些类，可以从```rest_framework.generics```中导入。

#### ```CreateAPIView``` 类

用于**create-only**endpoints。

提供一个```post```处理方法。

包含```CreateModelMixin```和```GenericAPIView```

源代码：

```python
class CreateAPIView(mixins.CreateModelMixin,
                    GenericAPIView):
    """
    创建一个模型实例的 具体的命名视图
    """
    def post(self, request, *args, **kwargs):
        return self.create(request, *args, **kwargs)
```

#### ```ListAPIView``` 类

用于一个只读的端点，来代表一个模型实例的集合。(查询结果集)

提供一个```get```处理方法。

包含```GenericAPIView```, ```ListModelMixin```

#### ```RetrieveAPIView``` 类

用于一个只读的端点，来代表一个单个的模型实例。

提供一个```get```处理方法。

包含```GenericAPIView```, ```RetrieveModelMixin```

#### ```DestroyAPIView``` 类

用于删除一个单个的模型实例。

提供一个```delete```处理方法。

包含```GenericAPIView```, ```DestroyModelMixin```

#### ```UpdateAPIView``` 类

用于更新一个单个的模型实例。

提供```put```和```patch```处理方法。

包含```GenericAPIView```, ```UpdateModelMixin```

#### ```ListCreateAPIView``` 类

用于一个可读 可写的 端点，来代表一个模型实例的集合。

提供```get```和```post```处理方法。

包含```GenericAPIView```, ```ListModelMixin```, ```CreateModelMixin```

源代码：

```python
class ListCreateAPIView(mixins.ListModelMixin,
                        mixins.CreateModelMixin,
                        GenericAPIView):
    """
    具名通用视图， 用于列出一个查询结果集  或者 创建一个模型实例
    """
    def get(self, request, *args, **kwargs):
        return self.list(request, *args, **kwargs)

    def post(self, request, *args, **kwargs):
        return self.create(request, *args, **kwargs)
```

#### ```RetrieveUpdateAPIView``` 类

用于可读 可更新 的端点， 来代表一个单个的模型实例。

提供```get```, ```put```和```patch```处理方法。

包含```GenericAPIView```, ```RetrieveModelMixin```, ```UpdateModelMixin```

#### ```RetrieveDestroyAPIView``` 类

用于可读  可删除 的端点， 来代表一个单个的模型实例。

提供```get```，```delete```处理方法。

包含```GenericAPIView```, ```RetrieveModelMixin```, ```DestroyModelMixin```

#### ```RetrieveUpdateDestroyAPIView``` 类

用于 可读  可更新  可删除的端点， 来代表一个单个的模型实例。

提供```get```，```delete```, ```put```和```patch```处理方法。

包含```GenericAPIView```, ```RetrieveModelMixin```, ```UpdateModelMixin```, ```DestroyModelMixin```

---

### 自定义 通用视图

通常你会使用已经存在的通用视图，只是会有一些自定义的行为。如果你发现你自己在多个地方重用了这些自定义的行为，你可能需要将这些行为重构到一个公共的类里面，然后就可以在任何的视图或者视图集合中来应用它。

#### 创建自定义的 ```mixins```

比如，你可能需要URL中的多个字段来查询对象，那么你可能需要创建一个Mixins类，比如下面这样：

```python
class MultipleFieldLookupMixin(object):
    """
    应用这个mixin 到任何的view 或者 viewset 中 来获得 多个字段的过滤， 基于 `lookup_fields`属性；
    来代替默认的单个字段过滤。
    """
    def get_object(self):
        queryset = self.get_queryset()             # Get the base queryset
        queryset = self.filter_queryset(queryset)  # Apply any filter backends
        filter = {}
        for field in self.lookup_fields:
            if self.kwargs[field]: # Ignore empty fields.
                filter[field] = self.kwargs[field]
        obj = get_object_or_404(queryset, **filter)  # Lookup the object
        self.check_object_permissions(self.request, obj)
        return obj
```

在任何时候，只要你需要，你都可以在任何视图或者视图集合中去使用它，比如这样：

```python
class RetrieveUserView(MultipleFieldLookupMixin, generics.RetrieveAPIView):
    queryset = User.objects.all()
    serializer_class = UserSerializer
    lookup_fields = ('account', 'username')
```

如果你需要自定义某些行为，那么使用Mixins是最好的选择。

#### 创建自定义的 基础类

如果你使用了多个Mixins类，那么进一步，你可以自定义一个你自己的基础类，然后你可以在你的项目中使用它们。 比如下面这样：

```python
class BaseRetrieveView(MultipleFieldLookupMixin, generics.RetrieveAPIView):
    pass

class BaseRetrieveUpdateDestroyView(MultipleFieldLookupMixin, generics.RetrieveUpdateDestroyAPIView):
    pass
```

如果在你的项目中，有大量的视图来处理各种行为，那么最好的办法是指定一个基础类。

---

### ```PUT``` 用作 创建 数据

在REST framework的3.0版本之前，对于```PUT```可以是更新，也可以是创建操作； 这取决于对象是否存在。

允许```PUT```来当做创建操作是由问题的，因为这必然会暴露对象存在或者不存在的信息。

在3.0的版本中，不会再由```PUT``` 用作 ```404``` 和 ```PUT``` 用作创建数据，  取而代之的是将```404```设为默认的行为，因为这更简单，更明了。

如果你需要使用通用的```PUT-as-create```， 你可以在Mixins中包含```AllowPUTAsCreateMixin```。点击此处[查看代码](https://gist.github.com/tomchristie/a2ace4577eff2c603b1b)。

---

### 第三方包

以下第三方包提供了额外的通用视图的实现。

#### Django REST Framework bulk

这个[django-rest-framework-bulk package](https://github.com/miki725/django-rest-framework-bulk)实现了一些通用视图和Mixins类以及一些具体的命名通用视图来实现通过API 请求实现批量操作。

#### Django Rest Multiple Models

这个[Django Rest Multiple Models](https://github.com/MattBroach/DjangoRestMultipleModels)是了一些通用视图和Mixins视图，来实现通过一个单独的API请求来发送多个序列的模型实例集合或者是queryset(查询结果集)。

---

参考链接:
- https://docs.djangoproject.com/en/2.0/ref/class-based-views/#base-vs-generic-views
- https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Location
- https://github.com/miki725/django-rest-framework-bulk
- https://github.com/MattBroach/DjangoRestMultipleModels
- https://gist.github.com/tomchristie/a2ace4577eff2c603b1b