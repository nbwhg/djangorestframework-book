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

##### 属性

**基础设置**:

下面的属性，控制基础的视图行为。
- ```queryset```: —— 用于从视图返回对象。通常情况下，你必须要设置这个属性或者是重写```get_queryset()```方法。如果你重写了视图的方法，一定要记住，不能直接访问这个属性，而是要调用```get_queryset()```方法，因为查询会评估一次，为后续所有的请求提供结果。
- ```serializer_class```: —— 这个属性设置的类，将用于验证用户的输入并反序列化，或者序列化输出结果。通常你需要设置这个属性，或者是重写```get_serializer_class()```方法。
- ```lookup_field```: —— 这是一个模型字段，用来执行单个的模型实例的对象查看(意思是，外键查看)。默认是```'pk'```。注意，当使用超链接的API，你需要确保API视图和序列化类设置```lookup```字段，如果你需要使用一个自定义的值。
- ```lookup_url_kwarg```: —— 被应用于对象查找URL关键字参数。URLconf中应包括相应于该值的关键字参数。如果未设置，那么默认使用和```lookup_field```相同的值。

**分页设置**:

下面的属性被用于当使用```list```视图的时候来控制分页。

- ```pagination_class```: —— 用于给```list```视图结果分页。默认使用的是```settings```中的```DEFAULT_PAGINATION_CLASS```选项，选项结果是```'rest_framework.pagination.PageNumberPagination'```。 设置```pagination_class=None```将会禁用该视图的分页。

注意： 在使用```Django REST Framework 3.7.7```版本的过程中，该分页设置为```None```，也就是默认没有分页效果。

默认设置位置: ```rest_framework.settings.DEFAULTS.DEFAULT_PAGINATION_CLASS```

关于分页类的源码位置: ```rest_framework.pagination.*Pagination```

**过滤设置**:

- ```filter_backends```: —— 可以用来过滤Queryset的过滤类的一个列表。默认值是```settings```中的```DEFAULT_FILTER_BACKENDS```选项。

注意： 在使用```Django REST Framework 3.7.7```版本的过程中，该过滤设置为```()```。

##### 方法

**基础方法**:

方法1：```get_queryset(self)```

方法2：```get_object(self)```

方法3：```filter_queryset(self, queryset)```

方法4：```get_serializer_class(self)```

**保存 和 删除  钩子**：

下面的这些方法是由```Mixins```类提供，并提供了简单的用于覆盖对象默认的 保存和删除 的行为。

- ```perform_create(self, serializer)``` —— 当保存一个新的对象实例的时候，通过```CreateModelMixin```来调用。
- ```perform_update(self, serializer)``` —— 当保存一个已有的对象实例的时候，通过```UpdateModelMixin```来调用。
- ```perform_destroy(self, instance)``` —— 当删除一个对象实例的时候，通过```DestroyModelMixin```来调用。

***其他方法**: 

你不需要重写以下方法，虽然你可能需要打电话到他们如果你使用```GenericAPIView```写自定义的视图。

---

### Mixins 类

源码位置： ```rest_framework.mixins.*Mixin```

#### ```ListModelMixin``` 类

#### ```CreateModelMixin``` 类

#### ```RetrieveModelMixin``` 类

#### ```UpdateModelMixin``` 类

#### ```DestroyModelMixin``` 类

---

### 命名视图 类

源码位置： ```rest_framework.generics.*APIView```

#### ```CreateAPIView``` 类

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

#### ```RetrieveAPIView``` 类

#### ```DestroyAPIView``` 类

#### ```UpdateAPIView``` 类

#### ```ListCreateAPIView``` 类

#### ```RetrieveUpdateAPIView``` 类

#### ```RetrieveDestroyAPIView``` 类

#### ```RetrieveUpdateDestroyAPIView``` 类

---

### 自定义 通用视图

#### 创建自定义的 ```mixins```

#### 创建自定义的 基础类

---

### ```PUT``` 创建 数据

---

### 第三方包

#### Django REST Framework bulk

#### Django Rest Multiple Models

---

参考链接:
- https://docs.djangoproject.com/en/2.0/ref/class-based-views/#base-vs-generic-views
- https://www.jianshu.com/p/fc556769a0b3