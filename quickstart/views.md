## 视图

现在，我们该写一些视图了，打开```tutorial/quickstart/views.py```
```python
#!/usr/bin/env python

from django.contrib.auth.models import User, Group
from rest_framework import viewsets
from tutorial.quickstart.serializers import UserSerializer, GroupSerializer

class UserViewSet(viewsets.ModelViewSet):
    """
    API endpoint 允许 查看 或者 编辑 用户
    """
    queryset = User.objects.all().order_by('-date_joined')
    serializer_class = UserSerializer


class GroupViewSet(viewsets.ModelViewSet):
    """
    API endpoint 允许 查看 或者 编辑 用户组
    """
    queryset = Group.objects.all()
    serializer_class = GroupSerializer
```

和传统的做法不一样，我们不是将多个视图写在一起，而是将所有公共行为集合```viewsets```类中。

如果有需求，可以很轻松的将各个功能拆解到单独的视图中，但是使用```viewsets```可以保证我们的视图非常的干净整洁。
