## Responses

> 与基本的HttpResponse 对象不同，TemplateResponse 对象会记住视图提供的模板和上下文的详细信息来计算响应。 响应的最终结果在后来的响应处理过程中直到需要时才计算。  
——Django 官方文档

点击[这里查看官方文档](https://docs.djangoproject.com/en/2.0/ref/template-response/)

REST framework支持HTTP内存的协商通过给定的```Response```类，你返回的内容可以渲染成多种```content type```，这依赖于客户端的请求。

这里的```Response```是Django的```SimpleTemplateResponse```的子类。REST framework使用标准的HTTP内容协商来确定最终返回的内容应该如何被渲染。

对于```Response```类并不是强制使用的，如果需要的话，你可以从你的视图中返回一个常规的```HttpResponse```或者是```StreamingHttpResponse```对象。使用```Response```仅是提供了一个方便的接口用于返回多种多样的格式的响应数据。

除非你出于某些原因想定制化开发REST framework，否则你应该总是使用```APIView```类或者是```@api_view```函数，为你的视图返回一个```Response```对象。这样做可以在你的视图返回结果之前，来确定视图执行内容协商并且选择恰当合适的渲染器来渲染响应结果。

关于```SimpleTemplateResponse```的参考链接，点[这里](http://python.usyiyi.cn/documents/django_182/ref/template-response.html)。

---

### 创建Response

#### ```Response()```

**初始化语法：** ```Response(self, data=None, status=None, template_name=None, headers=None, exception=False, content_type=None)```

和常规的```HttpResponse```对象不同，你不需要传入一个已经渲染的内容来实例化```Response```对象。相反你传递的是一个未渲染的数据。

即使```Response```类使用渲染器也并不能处理复杂的数据类型，比如Django Model实例类型。所以在创建```Response```对象之前，你需要序列化数据为原始的数据类型。

你可以使用REST framework的```Serializer```类来处理数据的序列化，或者使用你自定义的序列化器。

参数：

- ```data```: 用于response的已经序列的数据
- ```status```: 用于response的状态码
- ```template_name```: 如果```HTMLRenderer```被选择，那么使用这个模板来渲染数据
- ```headers```: 一个字典，用于在response中的HTTP头
- ```content_type```: 用于response的content type。通常，这是有渲染器通过内容协商自动设定。有一些特殊的情况可以显示的指定内容类型。

---

### 属性

#### ```.data```

未渲染的，已经序列化的响应数据。

#### ```.status_code```

HTTP响应代码。

#### ```.content```

已经渲染的内容。如果要调用```.render()```方法，那么```.content```必须要可以被访问。

#### ```.template_name```

如果指定了，这显示指定的模板名。

#### ```.accepted_renderer```

渲染器(```renderer```)实例，用于渲染响应结果。

由```APIView```或者是```@api_view```自动设置。

#### ```.accepted_media_type```

在内容协商阶段确定的媒体类型(```media type```)。

由```APIView```或者是```@api_view```自动设置。

#### ```.renderer_context```

将要传递给渲染器的```.render()```方法的一个额外的上下文信息。是一个字典。

由```APIView```或者是```@api_view```自动设置。

---

### 标准的```HttpResponse```属性

REST framework中的```Response```是Django原生的```SimpleTemplateResponse```的扩展。所以标准的属性和方法都可以使用。比如你可以使用标准的方法设置响应头：

```python
response = Response()
response['Cache-Control'] = 'no-cache'
```

#### ```.render()```

**初始化语法：** ```.render()```

这个方法被调用用于将响应的序列化数据渲染到最终的响应内容中。当```.render()```被调用，响应内容被设置，调用```.render(data, accepted_media_type, renderer_context)```方法，在```accepted_renderer```实例中。

通常你不需要自己去调用```.render()```。

---

参考链接：
- https://docs.djangoproject.com/en/2.0/ref/template-response/
