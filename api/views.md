## 视图```Views```

### 基于类的视图

> Django的基于类的视图和老式视图相比是更受欢迎的。  
——Reinout van Rees

REST framework提供了一个```APIView```类，是Django的```View```类的子类。

相比于常规的```View```类，REST framework的```APIView```有以下几点不同：

- 传递给处理方法的请求是REST framework的```Request```实例，而不是Django的```HttpRequest```实例。
- 处理方法可能返回REST framework的```Response```来代替Django的```HttpResponse```。该视图将会管理内容协商并给响应设置正确的渲染器。
- 任何异常将会被```APIException```捕捉，并且返回一个合适的响应。(包含一个合适的错误提示)
- 任何进入的请求，在派分到处理函数之前，都将是已经认证的，并且有适当的权限，或者还会有节流的检查。

使用```APIView```类基本上跟使用常规的```View```是基本一样的。像往常一样，进来的请求会被分派到适当的处理函数中，比如```.get()```或者是```.post()```。此外，还可以设置大量的属性，用来控制API策略的方方面面。

```
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import authentication, permissions
from django.contrib.auth.models import User

class ListUsers(APIView):
    """
    列出系统中所有的用户.

    * 需要进行Token验证.
    * 仅仅只有管理才可以访问这个视图.
    """
    authentication_classes = (authentication.TokenAuthentication,)
    permission_classes = (permissions.IsAdminUser,)

    def get(self, request, format=None):
        """
        Return a list of all users.
        """
        usernames = [user.username for user in User.objects.all()]
        return Response(usernames)
```

---

**注意:** 刚开始可能会觉得Django REST Framework的```APIView```和```GenericAPIView```和各种各样的```Mixins```和```Viewsets```有非常多的方法，属性和关系。这里有一个在线的地址，[Classy Django REST Framework](http://www.cdrf.co/)提供了一个可以浏览的资源，包含完整的方法和属性，对于Django REST Framework的基于类的视图。

---

#### API策略 属性

以下属性用于控制API视图的方方面面, 这些属性都是可插拔的。

##### ```.renderer_classes```

##### ```.parser_classes```

##### ```.authemtication_classes```

##### ```.throttle_classes```

##### ```.permission_classes```

##### ```.content_negotiation_class```


#### API策略 实例方法

REST framework 会使用下面的方法 去实例化 各种可插拔的 API 策略。 通常情况你不需要重写这些方法。

##### ```.get_renderers(self)```

##### ```.get_parsers(self)```

##### ```.get_authenticators(self)```

##### ```.get_throttles(self)```

##### ```.get_permissions(self)```

##### ```.get_content_negotiator(self)```

##### ```.get_exception_handler(self)```

#### API策略 实施方法

在REST framework 把请求分发到相应的处理函数之前，会调用下面的方法。

##### ```.check_permissions(self, request)```

##### ```.check_throttles(self, request)```

##### ```.perform_content_negotiation(self, request, force=False)```

#### Dispatch 分派方法

下面的这些方法会直接被视图的```.dispatch()```方法来调用。在调用处理函数(比如 ```.get()```, ```.post()```, ```.put()```, ```.patch()```)之前或者之后执行一些动作。

##### ```.initial(self, request, *args, **kwargs)```

不管执行任何动作的处理函数之前，这个方法都会被首先调用。这个方法用来执行权限和限流功能，并且勇于处理内容协商。

你通常不需要修改这个方法。

##### ```.handle_exception(self, exc)```

执行处理函数中抛出的任何异常都会被传递到这个方法中，该方法将会返回一个```Response```实例，或者重新```raise```一个异常。

默认处理异常的行为可以是任何```rest_framework.exceptions.APIException```的子类，和Django的```Http404```和```PermissionDenied```异常一样， 都会返回一个合理恰当的错误响应。

如果你需要在你的API中自定义错误响应，你应该在你继承的子类中重写该方法。

##### ```.initialize_request(self, request, *args, **kwargs)```

初始化请求。

确保传递给处理方法的请求对象是```Request```的实例，而不是常规的Django的```HttpRequest```。

该方法实际上就是返回一个```Request```的实例。

你通常不需要修改这个方法。

##### ```.finalize_response(self, request, response, *args, **kwargs)```

返回最终的响应对象。

该方法会在最后调用，确保处理函数返回的任何```Response```对象会被正确的渲染。

该方法会通过内容协商(```self.perform_content_negotiation(request, force=True)```)，确定```response.accepted_renderer```和```response.accepted_media_type```。

你通常也不需要修改这个方法。

---

### 基于函数的视图

> 基于类的视图是一个优越的解决方案，这是一个误解。
——Nick Coghlan

REST framework对于基于函数的视图同样也能非常好的工作。REST framework提供了一组简单的装饰器，用来包装你的基于函数的视图；来保证你的视图接收到的请求是```Request```的实例(而不是Django的HttPRequest)，并且返回的响应是```Response```的实例(而不是Django的HttpResponse)，并且允许您配置如何处理请求。

#### ```@api_view()```

**语法：** ```@api_view(http_method_names=['GET'])```

核心功能就是```@api_view```装饰器，并且要把你视图应该接收处理的HTTP方法以列表的形式，写到装饰器的参数中。

比如，下面的例子是如何编写一个非常简单的视图，并且仅仅是手动返回一些数据：

```python
from rest_framework.decorators import api_view

@api_view()
def hello_world(request):
    return Response({"message": "Hello, world!"})
```

这个视图将使用[```settings```](./settings.md)中指定的默认的渲染器，解析器和身份认证类。

默认情况只有```GET```方法会被允许。其他方法将会返回```"405 Method Not Allowed"```。如果要改变这种行为，需要指定视图允许哪些方法，比如下面这段代码：

```python
@api_view(['GET', 'POST'])
def hello_world(request):
    if request.method == 'POST':
        return Response({"message": "Got some data!", "data": request.data})
    return Response({"message": "Hello, world!"})
```

#### API策略 装饰器

REST framework提供了其他的装饰器，用于给你的视图增加一些设置，用于覆盖默认的设置。

这必须放置在```@api_view```装饰器的下面。

比如，限制([限流](./throttling.md))的视图每天每个用户只可以访问一次，可以使用装饰器```@throttle_classes```，并把限制策略类以列表的形式传递给这个装饰器：

```python
from rest_framework.decorators import api_view, throttle_classes
from rest_framework.throttling import UserRateThrottle

class OncePerDayUserThrottle(UserRateThrottle):
        rate = '1/day'

@api_view(['GET'])
@throttle_classes([OncePerDayUserThrottle])
def view(request):
    return Response({"message": "Hello for today! See you tomorrow!"})
```

这些装饰器对应```APIView```中设置的子类，他们功能是一样的。

可用的装饰器如下：
- ```@renderer_classes(...)```
- ```@parser_classes(...)```
- ```@authentication_classes(...)```
- ```@throttle_classes(...)```
- ```@permission_classes(...)```

每个装饰器，至少接收一个类型为list或者tuple的参数，并且元素是一个类。

#### 视图schema 装饰器

如果要给基于函数的视图，覆盖默认的schema生成，你可以使用```@schema```装饰器，同样的，这个装饰器必须在```@api_view```装饰器的下面。 比如：

```python
from rest_framework.decorators import api_view, schema
from rest_framework.schemas import AutoSchema

class CustomAutoSchema(AutoSchema):
    def get_link(self, path, method, base_url):
        # override view introspection here...

@api_view(['GET'])
@schema(CustomAutoSchema())
def view(request):
    return Response({"message": "Hello for today! See you tomorrow!"})
```

这个装饰器的参数是一个```AutoSchema```的实例。具体参考文档[schemas](./schemas.md)。当然你也可以传递一个```None```，比如：

```python
@api_view(['GET'])
@schema(None)
def view(request):
    return Response({"message": "Will not appear in schema!"})
```

---

参考链接：
- http://www.cdrf.co/
- 