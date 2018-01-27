## 序列化

首先我们要定义序列化器。让我们创建一个新的模块```tutorial/quickstart/serializers.py```
```python
#!/usr/bin/env python

from django.contrib.auth.models import User, Group
from rest_framework import serializers

class UserSerializer(serializers.HyperlinkedModelSerializer):
    class Meta:
        model = User
        fields = ('url', 'username', 'email', 'groups')


class GroupSerializer(serializers.HyperlinkedModelSerializer):
    class Meta:
        model = Group
        fields = ('url', 'name')
```

注意：在本例中我们使用```HyperlinkedModelSerializer```来实现一个超链接关系。当然你还可以使用主键关系或者其他各种各样的关系，不过在RESTful API设计中超链接是最好的方式。
