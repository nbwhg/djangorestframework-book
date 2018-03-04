# 类视图

我们也可以使用基于类的视图来编写API视图，而不是基于函数的视图。正如我们将看到的，这是一个强大的模式，可以让我们复用常用功能，并帮助我们保持代码干爽。

## 使用基于类的视图重写我们的API
我们将首先将根视图重写为基于类的视图。这些都涉及到重构```views.py```。

```python
from snippets.models import Snippet
from snippets.serializers import SnippetSerializer
from django.http import Http404
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status


class SnippetList(APIView):
    """
    List all snippets, or create a new snippet.
    """
    def get(self, request, format=None):
        snippets = Snippet.objects.all()
        serializer = SnippetSerializer(snippets, many=True)
        return Response(serializer.data)

    def post(self, request, format=None):
        serializer = SnippetSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
```

到现在为止还挺好。它看起来与之前的用法非常相似，但我们在不同的HTTP方法做了更好的分离。我们还需要更新实例视图```views.py```。

```python
class SnippetDetail(APIView):
    """
    Retrieve, update or delete a snippet instance.
    """
    def get_object(self, pk):
        try:
            return Snippet.objects.get(pk=pk)
        except Snippet.DoesNotExist:
            raise Http404

    def get(self, request, pk, format=None):
        snippet = self.get_object(pk)
        serializer = SnippetSerializer(snippet)
        return Response(serializer.data)

    def put(self, request, pk, format=None):
        snippet = self.get_object(pk)
        serializer = SnippetSerializer(snippet, data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

    def delete(self, request, pk, format=None):
        snippet = self.get_object(pk)
        snippet.delete()
        return Response(status=status.HTTP_204_NO_CONTENT)
```        
        
这看起来不错。同样，它现在仍然非常类似基于函数的视图。

我们现在还需要稍微重构我们使用基于类的视图的```snippets/urls.py```。

```python
from django.conf.urls import url
from rest_framework.urlpatterns import format_suffix_patterns
from snippets import views

urlpatterns = [
    url(r'^snippets/$', views.SnippetList.as_view()),
    url(r'^snippets/(?P<pk>[0-9]+)/$', views.SnippetDetail.as_view()),
]

urlpatterns = format_suffix_patterns(urlpatterns)
```

## 使用mixins

使用基于类的视图的最大好处之一是，它允许我们轻松地编写可重用的行为片段。

到目前为止，我们使用的create/retrieve/update/delete操作对于我们创建的任何模型支持的API视图将非常相似。这些常见行为的某些部分在REST框架的mixin类中实现。


我们来看看如何使用mixin类组合视图。这是我们的```views.py```模块了。

```python
from snippets.models import Snippet
from snippets.serializers import SnippetSerializer
from rest_framework import mixins
from rest_framework import generics

class SnippetList(mixins.ListModelMixin,
                  mixins.CreateModelMixin,
                  generics.GenericAPIView):
    queryset = Snippet.objects.all()
    serializer_class = SnippetSerializer

    def get(self, request, *args, **kwargs):
        return self.list(request, *args, **kwargs)

    def post(self, request, *args, **kwargs):
        return self.create(request, *args, **kwargs)
```

我们将花一点时间仔细检查这里发生了什么。我们正在构建我们的视图```GenericAPIView```，并添加```ListModelMixin```和```CreateModelMixin```。

基类提供核心功能，mixin类提供```.list()```和```.create()```操作。然后，我们明确地将方法```get```和```post```方法绑定到适当的操作。迄今为止足够简单的东西。

```python
class SnippetDetail(mixins.RetrieveModelMixin,
                    mixins.UpdateModelMixin,
                    mixins.DestroyModelMixin,
                    generics.GenericAPIView):
    queryset = Snippet.objects.all()
    serializer_class = SnippetSerializer

    def get(self, request, *args, **kwargs):
        return self.retrieve(request, *args, **kwargs)

    def put(self, request, *args, **kwargs):
        return self.update(request, *args, **kwargs)

    def delete(self, request, *args, **kwargs):
        return self.destroy(request, *args, **kwargs)
```

非常相似。同样，我们正在使用的```GenericAPIView```类来提供核心功能，并混入增加提供```.retrieve()```，```.update()```和```.destroy()```。


## 使用通用的基于类的视图
使用mixin类，我们重写了视图，使用比以前稍少的代码，但我们可以更进一步。REST框架提供了一组已经混合的通用视图，我们可以使用这些视图更多地修剪我们的```views.py```模块。

```
from snippets.models import Snippet
from snippets.serializers import SnippetSerializer
from rest_framework import generics


class SnippetList(generics.ListCreateAPIView):
    queryset = Snippet.objects.all()
    serializer_class = SnippetSerializer


class SnippetDetail(generics.RetrieveUpdateDestroyAPIView):
    queryset = Snippet.objects.all()
    serializer_class = SnippetSerializer
```    
    
Wow，这很简洁。我们免费获得了大量的代码，而且我们的代码看起来很好，干净，惯用的Django。

接下来，我们将转到本教程的第4部分，其中我们将介绍如何处理API的身份验证和权限。
