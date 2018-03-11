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

你还可以为基于```APIView```的单个视图或者视图集合(```viewset```)设置解析器：

```python
from rest_framework.parsers import JSONParser
from rest_framework.response import Response
from rest_framework.views import APIView

class ExampleView(APIView):
    """
    A view that can accept POST requests with JSON content.
    """
    parser_classes = (JSONParser,)

    def post(self, request, format=None):
        return Response({'received data': request.data})
```

或者为基于```@api_view```装饰器的函数视图设置解析器：

```python
from rest_framework.decorators import api_view
from rest_framework.decorators import parser_classes
from rest_framework.parsers import JSONParser

@api_view(['POST'])
@parser_classes((JSONParser,))
def example_view(request, format=None):
    """
    A view that can accept POST requests with JSON content.
    """
    return Response({'received data': request.data})
```

---

<br />
<br />
<br />
<br />

### API 参考

关于解析器的源码位置： ```rest_framework.parsers```

#### ```JSONParser```解析器

解析请求内容是```JSON```格式的数据

HTTP请求中的```.media_type```为 ```application/json```

#### ```FormParser```解析器

解析请求内容是HTML表单的数据。```request.data```的内容将被```QueryDict```填充。

源代码：

```python
class FormParser(BaseParser):
    """
    Parser for form data.
    """
    media_type = 'application/x-www-form-urlencoded'

    def parse(self, stream, media_type=None, parser_context=None):
        """
        Parses the incoming bytestream as a URL encoded form,
        and returns the resulting QueryDict.
        """
        parser_context = parser_context or {}
        encoding = parser_context.get('encoding', settings.DEFAULT_CHARSET)
        data = QueryDict(stream.read(), encoding=encoding)  ### 注意这里返回的 QueryDict
        return data
```

通常情况下，会将```FormParser```和```MultiPartParser```来一起使用，以便完美的支持HTML表单数据。

HTTP请求中的```.media_type```为 ```application/x-www-form-urlencoded```

#### ```MultiPartParser```解析器

解析支持文件上传的多部分HTML表单内容。

通常情况下，会将```FormParser```和```MultiPartParser```来一起使用，以便完美的支持HTML表单数据。

HTTP请求中的```.media_type```为 ```multipart/form-data```

#### ```FileUploadParser```解析器

解析原始文件上传的内容。```request.data```的属性将是一个包含上传文件的单个关键字```'file'```的字典。

相关源代码：

```python
class FileUploadParser(BaseParser):
    """
    Parser for file upload data.
    """
    media_type = '*/*'
    errors = {
        'unhandled': 'FileUpload parse error - none of upload handlers can handle the stream',
        'no_filename': 'Missing filename. Request should include a Content-Disposition header with a filename parameter.',
    }

    def parse(self, stream, media_type=None, parser_context=None):
        """
        Treats the incoming bytestream as a raw file upload and returns
        a `DataAndFiles` object.

        `.data` will be None (we expect request body to be a file content).
        `.files` will be a `QueryDict` containing one 'file' element.
        """
        parser_context = parser_context or {}
        request = parser_context['request']
        encoding = parser_context.get('encoding', settings.DEFAULT_CHARSET)
        meta = request.META
        upload_handlers = request.upload_handlers
        filename = self.get_filename(stream, media_type, parser_context)

        if not filename:
            raise ParseError(self.errors['no_filename'])

        # Note that this code is extracted from Django's handling of
        # file uploads in MultiPartParser.
        content_type = meta.get('HTTP_CONTENT_TYPE',
                                meta.get('CONTENT_TYPE', ''))
        try:
            content_length = int(meta.get('HTTP_CONTENT_LENGTH',
                                          meta.get('CONTENT_LENGTH', 0)))
        except (ValueError, TypeError):
            content_length = None

        # See if the handler will want to take care of the parsing.
        for handler in upload_handlers:
            result = handler.handle_raw_input(stream,
                                              meta,
                                              content_length,
                                              None,
                                              encoding)
            if result is not None:
                return DataAndFiles({}, {'file': result[1]})   ### 看这里返回的字典

        # This is the standard case.
        possible_sizes = [x.chunk_size for x in upload_handlers if x.chunk_size]
        chunk_size = min([2 ** 31 - 4] + possible_sizes)
        chunks = ChunkIter(stream, chunk_size)
        counters = [0] * len(upload_handlers)

        for index, handler in enumerate(upload_handlers):
            try:
                handler.new_file(None, filename, content_type,
                                 content_length, encoding)
            except StopFutureHandlers:
                upload_handlers = upload_handlers[:index + 1]
                break

        for chunk in chunks:
            for index, handler in enumerate(upload_handlers):
                chunk_length = len(chunk)
                chunk = handler.receive_data_chunk(chunk, counters[index])
                counters[index] += chunk_length
                if chunk is None:
                    break

        for index, handler in enumerate(upload_handlers):
            file_obj = handler.file_complete(counters[index])
            if file_obj is not None:
                return DataAndFiles({}, {'file': file_obj})   ### 看这里返回的字典

        raise ParseError(self.errors['unhandled'])
```

如果在URL关键字参数中有```filename```，调用了相关的视图，视图调用了```FileUploadParser```解析器，那么这个关键字参数将会用作文件名。

如果并没有URL关键字参数，那么客户端必须在HTTP头中设置文件名。比如这样： ```Content-Disposition: attachment; filename=upload.jpg```。

HTTP请求中的```.media_type```为 ```*/*```

注意：

- 解析器```FileUploadParser```应该用于本地的客户端上传原始的数据的请求。如果是基于Web端的上传文件，或者是为本地客户端支持```multipart upload```，那么解析器应该用```MultiPartParser```。
- 由于```FileUploadParser```可以匹配任何类型的Media类型，所以通常这个解析器可以设置在APIView上。
- 解析器```FileUploadParser```使用的是Django的```request.upload_handlers```。

基本使用案例：

```python
# views.py
class FileUploadView(views.APIView):
    parser_classes = (FileUploadParser,)

    def put(self, request, filename, format=None):
        file_obj = request.data['file']
        # ...
        # do some stuff with uploaded file
        # ...
        return Response(status=204)

# urls.py
urlpatterns = [
    # ...
    url(r'^upload/(?P<filename>[^/]+)$', FileUploadView.as_view())
]
```

---

<br />
<br />
<br />
<br />

### 自定义 解析器

要想实现自定义的解析器，你应该继承```BaseParser```类，并且要设置```.media_type```属性，还要实现```.parse(self, stream, media_type, parser_context)```方法。

这个方法返回的数据，被填充在```request.data```中。

应该给```.parser()```方法中传递的参数有：

- ```stream```: 一个表示request body的流式对象。
- ```media_type```: 可选的，如果设置了，那么就代表进入请求的内容的Media类型。依赖HTTP请求头中的```Content-Type:```。
- ```parser_context```: 可选的。如果提供，这个参数将是一个字典，其中包含可能需要分析请求内容的任何附加上下文。默认包含以下key， ```view```, ```request```, ```args```和```kwargs```。

#### ```Example```

下面是一个普通文本解析器的一个实例，最后会将文本的内容字符串作为数据填充到```request.data```中：

```python
class PlainTextParser(BaseParser):
    """
    Plain text parser.
    """
    media_type = 'text/plain'

    def parse(self, stream, media_type=None, parser_context=None):
        """
        Simply return a string representing the body of the request.
        """
        return stream.read()
```

---

<br />
<br />
<br />
<br />

### 第三方包

下面是一些可用的第三方包。

#### YAML

[REST framework YAML](https://jpadilla.github.io/django-rest-framework-yaml/)提供了YAML的解析和渲染支持。之前在REST framework有YAML的支持，现在使用第三方的包来替代。

使用pip安装：

```shell
$ pip install djangorestframework-yaml
```

修改你的REST framework的设置：

```python
REST_FRAMEWORK = {
    'DEFAULT_PARSER_CLASSES': (
        'rest_framework_yaml.parsers.YAMLParser',
    ),
    'DEFAULT_RENDERER_CLASSES': (
        'rest_framework_yaml.renderers.YAMLRenderer',
    ),
}
```

#### XML

[REST Framework XML](https://jpadilla.github.io/django-rest-framework-xml/)提供了一个简单的非正式的XML格式。

使用pip安装：

```shell
$ pip install djangorestframework-xml
```

修改REST framework的设置：

```python
REST_FRAMEWORK = {
    'DEFAULT_PARSER_CLASSES': (
        'rest_framework_xml.parsers.XMLParser',
    ),
    'DEFAULT_RENDERER_CLASSES': (
        'rest_framework_xml.renderers.XMLRenderer',
    ),
}
```

#### MessagePack

MessagePack 是一个快速的、高效的 二进制序列化格式。 [djangorestframework-msgpack](https://github.com/juanriaza/django-rest-framework-msgpack)包为REST framework提供了```MessagePack```的渲染器和解析器。

#### 驼峰 JSON

[djangorestframework-camel-case](https://github.com/vbabiy/djangorestframework-camel-case)包为REST framework提供了驼峰JSON的渲染器和解析器。这意味着你可以在序列化程序中使用Python风格的下划线语法，而最后导出的时候使用JavaScript风格的驼峰语法。

---

参考链接：

- https://jpadilla.github.io/django-rest-framework-yaml/
- https://jpadilla.github.io/django-rest-framework-xml/
- https://github.com/juanriaza/django-rest-framework-msgpack
- https://github.com/vbabiy/djangorestframework-camel-case