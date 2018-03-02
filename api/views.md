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

##### ```.handle_exception(self, exc)```

##### ```.initialize_request(self, request, *args, **kwargs)```

##### ```.finalize_response(self, request, response, *args, **kwargs)```

---

### 基于函数的视图

> 基于类的视图是一个优越的解决方案，这是一个误解。
——Nick Coghlan

REST framework对于基于函数的视图同样也能非常好的工作。REST framework提供了一组简单的装饰器，用来包装你的基于函数的视图；来保证你的视图接收到的请求是```Request```的实例(而不是Django的HttPRequest)，并且返回的响应是```Response```的实例(而不是Django的HttpResponse)，并且允许您配置如何处理请求。

#### ```@api_view()```

#### API策略 装饰器

#### 视图schema 装饰器

---

参考链接：
- http://www.cdrf.co/
- 