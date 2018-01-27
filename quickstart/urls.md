## 路由

现在让我们连接API到路由中，在```tutorial/urls.py```
```python
from django.conf.urls import url
from django.conf.urls import include
from rest_framework import routers
from tutorial.quickstart import views


router = routers.DefaultRouter()
router.register(r'users', views.UserViewSet)
router.register(r'groups', views.GroupViewSet)

# 将自动生成的URL 配置 放置在真正的路由配置中
# 此外, 我们还配置登录 登出的 路由
urlpatterns = [
    url(r'^', include(router.urls)),
    url(r'^api-auth', include('rest_framework.urls', namespace='rest_framework'))
]
```

因为我们使用的是```viewsets(视图集合)```而不是多个视图```views```。所以我们可以通过将```viewsets```注册到```router```类中来自动的生成API相关的URL。

如果，我们需要对API URL进行更多的控制，那么可以降低到使用常规的```class-based```基于类的视图的，然后对每个URL 进行详细的控制。

最后，我们为可视化的API添加了登录/登出视图。当然，这是可选的，但是这对于可视化API和一些需要认证的API这是非常有必要的。