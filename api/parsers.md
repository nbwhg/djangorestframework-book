## 解析器Parsers

> 机器交互式Web服务更倾向于使用更多的结构化格式来发送数据，而不是简单的表单格式。这是因为他们会发送比表单更复杂的数据。  
—— Malcom Tredinnick

REST framework包含了许多内置的解析器类，允许您使用各种```Media Types```来接收请求。当然，也可以支持定义你自己的解析器，这使得你可以灵活的设计你的API所接收的```Media Types```。

#### 解析器是如何确定的

视图中有效的解析器总是被定义为一个包含类的列表。当```request.data```中的数据被访问的时候，REST framework将会检查传进来的请求中的HTTP 头中的```Content-Type```来确定使用哪种解析器来解析请求数据(```request data```)。

---

注意： 在开发HTTP 客户端应用程序的时候，应该始终记住在请求中设置```Content-Type```头。

如果你不设置content type，大多数客户端默认将会使用```'application/x-www-form-urlencoded'```， 这可能不是你想要的。

举个例子，如果你要用```JQuery```的```.ajax()```方法发送一个使用JSON编码的数据，那么你应该始终设置```contentType: 'application/json'```。

---

#### 设置解析器

默认情况下解析器的设置可能是通过```DEFAULT_PARSER_CLASSES```设置的全局的。比如，下面的设置将仅仅允许```JSON```的请求，代替了默认的```JSON or form data```的设置。

```python
REST_FRAMEWORK = {
    'DEFAULT_PARSER_CLASSES': (
        'rest_framework.parsers.JSONParser',
    )
}
```

---

### API 参考

#### ```JSONParser```解析器

#### ```FormParser```解析器

#### ```MultiPartParser```解析器

#### ```FileUploadParser```解析器

---

### 自定义 解析器

#### ```stream```

#### ```media_type```

#### ```parser_context```

#### ```Example```

---

### 第三方包

#### YAML

#### XML

#### MessagePack

#### 驼峰 JSON

---

参考链接：

- 