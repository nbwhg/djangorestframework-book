# 认证和权限

目前我们的API没有任何限制对谁都可以编辑或删除代码。我们希望有一些更高级的行为来保证：

* 代码始终与创建者关联。
* 只有经过身份验证的用户才可以创建。
* 只有创建者可能会更新或删除它。
* 未经身份验证的请求应只具有完全只读访问权限。

# 将信息添加到我们的model

我们将对```Snippet``` model类进行一些修改改。首先，我们添加几个字段。其中一个字段将用于表示创建Snippet的用户。其他字段将用于存储代码的突出显示的HTML表示。
将以下两个字段添加到```Snippet```模型中```models.py```。

```python
owner = models.ForeignKey('auth.User', related_name='snippets', on_delete=models.CASCADE)
highlighted = models.TextField()
```

我们还需要确保保存模型时，我们使用pygments代码高亮库来填充突出显示的字段。

我们需要一些额外的import：

from pygments.lexers import get_lexer_by_name
from pygments.formatters.html import HtmlFormatter
from pygments import highlight


现在我们可以在.save()模型类中添加一个方法：

```python
def save(self, *args, **kwargs):
    """
     使用“pygments”库创建高亮的HTML。    
    """
    lexer = get_lexer_by_name(self.language)
    linenos = self.linenos and 'table' or False
    options = self.title and {'title': self.title} or {}
    formatter = HtmlFormatter(style=self.style, linenos=linenos,
                              full=True, **options)
    self.highlighted = highlight(self.code, lexer, formatter)
    super(Snippet, self).save(*args, **kwargs)
```

完成这些工作后，我们需要更新我们的数据库表。通常我们会创建一个数据库```migration```来完成这个任务，但为了本教程的目的，我们只需删除数据库并重新开始。

```
rm -f db.sqlite3
rm -r snippets/migrations
python manage.py makemigrations snippets
python manage.py migrate
```

您可能还想创建几个不同的用户，用于测试API。最快的方法是使用createsuperuser命令。

```python
python manage.py createsuperuser
```


## 为我们的USER model添加端点

现在我们有一些用户可以使用，我们最好将这些用户的描述添加到我们的API中。创建一个新的序列化器是很容易的。在```serializers.py```添加：

```python
from django.contrib.auth.models import User

class UserSerializer(serializers.ModelSerializer):
    snippets = serializers.PrimaryKeyRelatedField(many=True, queryset=Snippet.objects.all())

    class Meta:
        model = User
        fields = ('id', 'username', 'snippets')
```


因为在用户模型上```snippets```是外键关系，所以在使用```ModelSerializer```类时它不会被默认包含，所以我们需要为它添加一个显式字段。

我们还会添加几个视图到```views.py```。我们只想为用户使用只读视图，因此我们将使用```ListAPIView```和```RetrieveAPIView```基于类的通用视图。

```python
from django.contrib.auth.models import User


class UserList(generics.ListAPIView):
    queryset = User.objects.all()
    serializer_class = UserSerializer


class UserDetail(generics.RetrieveAPIView):
    queryset = User.objects.all()
    serializer_class = UserSerializer
```

确保导入UserSerializer

```python
from snippets.serializers import UserSerializer
```

最后，我们需要通过在URL conf中引用这些视图来将这些视图添加到API中。将以下内容添加到```urls.py```文件中。

```python
url(r'^users/$', views.UserList.as_view()),
url(r'^users/(?P<pk>[0-9]+)/$', views.UserDetail.as_view()),
```

## Snippets 与 Users 关联
现在，如果我们创建了一个snippets，那么会无法将创建snippets的用户与snippets实例关联起来。用户不是作为序列化描述的一部分发送的，而是作为传入请求的属性。

我们处理这个问题的方式是通过重写```.perform_create()```snippets视图上的方法，这允许我们修改实例保存的方式，并处理传入请求或请求URL中隐含的任何信息。

在SnippetList视图类中，添加以下方法：

```
def perform_create(self, serializer):
    serializer.save(owner=self.request.user)
```
        
我们的序列化器的```create()```方法现在将传递一个额外的'owner'字段，以及来自请求的验证数据。


## 更新我们的序列化器

现在，snippet与创建它的用户相关联，让我们更新```SnippetSerializer```。将以下字段添加到序列化文件```serializers.py```中：

```python
owner = serializers.ReadOnlyField(source='owner.username')
```

注意:确保在内部元类的字段列表中添加```owner```。

这个领域正在做一件很有趣的事情。该```source```参数控制用于填充字段的属性，并且可以指向序列化实例上的任何属性。它也可以采用上面显示的虚线符号，在这种情况下，它将遍历给定的属性，就像使用Django的模板语言一样。


我们添加的字段是无类型的```ReadOnlyField```类，与其他类型的字段相比，例如```CharField```，```BooleanField```等等... untyped ```ReadOnlyField```是只读的，并且将用于序列化表示，但它们被反序列化时不会用于更新模型当。我们也可以```CharField(read_only=True)```在这里使用。

## 为视图添加必要的权限
既然snippets与用户相关联，我们希望确保只有经过身份验证的用户才能创建、更新和删除snippets。

REST框架包含许多权限类，我们可以使用这些权限类来限制可以访问视图的用户。在这种情况下，我们正在寻找的是```IsAuthenticatedOrReadOnly```确保经过身份验证的请求获得读写访问权限，未经身份验证的请求获得只读访问权限。

首先在views文件中添加以下导入

```python
from rest_framework import permissions
```

接着，下面的属性添加到```SnippetList```和```SnippetDetail```视图类。

```python
permission_classes = (permissions.IsAuthenticatedOrReadOnly,)
```


## 添加登录到Browsable API
如果您现在打开浏览器并访问API，会发现您不能创建新的snippets。为了创建，我们需要能够以用户身份登录。

我们可以通过编辑项目级```urls.py```文件中的```URLconf```来添加API的登录视图。

在文件顶部添加以下导入：

```python
from django.conf.urls import include
```
并且，在文件末尾，添加一个包含可访问API的登录和注销视图。

```python
urlpatterns += [
    url(r'^api-auth/', include('rest_framework.urls')),
]
```
```r'^api-auth/'```模式的部分实际上可以是您想要使用的任何URL。

现在，如果您再打开浏览器并刷新页面，则会在页面右上方看到一个“login”链接。如果您以前创建的用户之一登录，则可以再次创建新的snippets。

一旦创建了几个代码片段，请访问```'/ users /'```url，并注意每个用户的“snippets”字段中与每个用户关联的代码段ID列表。


## 对象级权限
实际上，我们希望所有人都可以看到所有snippets，但也要确保只有创建snippets的用户才能更新或删除它。

为此，我们需要创建一个自定义权限。

在snippets应用程序中，创建一个新文件， ```permissions.py```

```python
from rest_framework import permissions


class IsOwnerOrReadOnly(permissions.BasePermission):
    """
    Custom permission to only allow owners of an object to edit it.
    """

    def has_object_permission(self, request, view, obj):
        # Read permissions are allowed to any request,
        # so we'll always allow GET, HEAD or OPTIONS requests.
        if request.method in permissions.SAFE_METHODS:
            return True

        # Write permissions are only allowed to the owner of the snippet.
        return obj.owner == request.user
```
        
现在我们可以通过编辑views类的```permission_classes```属性来将该自定义权限添加到我们的snippets实例```SnippetDetail```：

```python
permission_classes = (permissions.IsAuthenticatedOrReadOnly,
                      IsOwnerOrReadOnly,)
```                      
                      
确保也要导入```IsOwnerOrReadOnly```。

```python
from snippets.permissions import IsOwnerOrReadOnly
```

现在，如果您再次打开浏览器，如果您以创建snippets的相同用户身份登录，则会发现'DELETE'和'PUT'操作仅出现在代码片段实例端点上。


## 使用API​​进行身份验证
因为我们现在对API有一组权限，如果我们想要编辑任何snippets，我们需要验证对它的请求。我们还没有设置任何认证类，默认是当前应用的，它们是```SessionAuthentication```和```BasicAuthentication```。

当我们通过网络浏览器访问API时，然后浏览器会话将为请求提供所需的身份验证，我们就可以登录了。

如果我们正在以编程方式与API进行交互，那么我们需要在每个请求上明确提供身份验证凭据。

如果我们尝试创建一个没有进行身份验证的snippets，我们会得到一个错误：

```python
http POST http://127.0.0.1:8000/snippets/ code="print 123"

{
    "detail": "Authentication credentials were not provided."
}
```

我们可以通过添加我们之前创建的用户和密码来验证成功的请求。

```python
http -a admin:password123 POST http://127.0.0.1:8000/snippets/ code="print 789"

{
    "id": 1,
    "owner": "admin",
    "title": "foo",
    "code": "print 789",
    "linenos": false,
    "language": "python",
    "style": "friendly"
}
```

## 概要
现在我们已经在我们的Web API上获得了相当细致的权限集合，以及系统用户和他们创建的snippets的终点。

在本教程的第5部分中，我们将研究如何通过为突出显示的片段创建HTML端点来将所有内容绑定在一起，并通过对系统内的关系使用超链接来提高API的凝聚力。



       
    
        