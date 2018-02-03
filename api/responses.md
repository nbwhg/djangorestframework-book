## Responses

> 与基本的HttpResponse 对象不同，TemplateResponse 对象会记住视图提供的模板和上下文的详细信息来计算响应。 响应的最终结果在后来的响应处理过程中直到需要时才计算。  
——Django 文档


---

### 创建Response

#### ```Response()```

**初始化语法：** ```Response(self, data=None, status=None, template_name=None, headers=None, exception=False, content_type=None)```

你可以使用REST framework的```Serializer```类来处理数据的序列化，或者使用你自定义的序列化器。

参数：

- ```data```: 
- ```status```:
- ```template_name```:
- ```headers```:
- ```content_type```:

---

### 属性

#### ```.data```

未渲染的，已经序列化的响应数据。

#### ```.status_code```

HTTP响应代码。

#### ```.content```

#### ```.template_name```

#### ```.accepted_renderer```

#### ```.accepted_media_type```

#### ```.renderer_context```

---

### 标准的```HttpResponse```属性

REST framework中的```Response```是Django原生的```SimpleTemplateResponse```的扩展。所以标准的属性和方法都可以使用。比如你可以使用标准的方法设置响应头。

```python
response = Response()
response['Cache-Control'] = 'no-cache'
```

#### ```.render()```

