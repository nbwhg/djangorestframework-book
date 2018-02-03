## Requests

> *如果你基于REST做一些web应用，那么你应该忽略```request.POST```.*  
—— Malcom Tredinnick

REST framework的```Request```类是标准的```HttpRequest```的扩展(并不是类继承)，增加了更加灵活的请求处理和请求认证。

---

### Request 解析

REST framework的```Request```对象提供了非常灵活的请求解析处理，无论是JSON数据格式还是其他类型的数据格式，```Request```对象都会用同样的方式来处理所有的表单数据。

#### ```.data```

```request.data```返回的是经过处理后的request body的内容。这和标准的```request.POST```和```request.FILES```属性是类似的，除此之外，还有以下特性：

- 包含所有解析的内容，包含*file*和*non-file*的输入
- 它支持解析除了```POST```以外其他的HTTP方法的内容，这意味着你可以访问```PUT```和```PATCH```的请求
- 它支持REST framework灵活的请求解析，不只是简简单单的表单处理。比如，处理JSON数据和处理表单数据是一样的。

更多的解析，查看文档 [解析处理](./parsers.md)

#### ```.query_params```

```request.query_params``` 对于标准的```request.GET```是一个更加正确的命名方式。

为了代码更加的清晰，我们建议您使用```request.query_params```来代替Django标准的```request.GET```方法。

这样做能保证代码更加正确且清晰，因为所有的HTTP方法都会包含有请求参数，而不仅仅是```GET```方法。

#### ```.parsers```

REST中的```APIView```类或者是```@api_view```装饰器会确保```.parsers```属性正确的设置为一个```list```。这个```list```包含一组```Parser```实例。

这些```Parser```实例是基于```view```中的```parser_classes```或者全局的```DEFAULT_PARSER_CLASSES```设置。

通常情况下不会访问这个属性。

可以查看```rest_framework.view.APIView```类中的```get_parsers```方法。有多少个对象```list```中就会有多少个```Parser```实例。

---

**注意：** 如果客户端发送的是一个畸形的内容，那么访问```request.data```可能会抛出```ParseError```异常。默认的REST framework的```APIView```类或者是```api_view```装饰器会捕捉错误，并返回一个```400 Bad Request```响应。

如果客户端发送请求并携带```content-type```，REST framework不能解析，那么会抛出```UnsupportedMediaType```异常，REST 会捕捉这个异常，并返回```415 Unsupported Media Type```响应。

---

### 内容协商

REST framework中的```request```还暴露了一些方法允许通过协商来决定结果。也就是说，这允许你自己实现一些行为，比如给不同的数据类型一个不同的序列化方案。

源码位置： ```rest_framework.view.APIView```类中的```perform_content_negotiation```方法。

#### ```.accepted_renderer```

这是在内容协商过程中允许选择的渲染实例。

#### ```accepted_media_type```

这个字符串代表在内容协商过程中允许的类型。

---

### 认证

REST framework为认证提供了非常灵活的需求：

- 在不同API部分可以使用不同的认证策略
- 支持使用多种认证策略
- 对进来的请求提供关联的用户和Token信息

#### ```.user```

REST framework中的```request.user```是```django.contrib.auth.models.User```返回的一个实例。这个行为取决于使用的认证策略。

如果请求是未认证的，那么```request.user```返回的是```django.contrib.auth.models.AnonymousUser```的实例。

更多详情查看 [认证中心](./authentication.md)

#### ```.auth```

REST framework中的```request.auth``` 会返回任何附加的认证上下文。准确的说这依赖于所使用的认证策略。通常，可能会是已经认证的用户的Token实例。

如果请求未认证，或者说没有额外附加的上下文，那么```request.auth```将返回```None```。

更多详情查看 [认证中心](./authentication.md)

#### ```.authenticators```

REST framework中的```APIView```类或者```api_view```装饰器会确保这个数据自动的被设置。这个属性是一个由```Authentication```实例组成的```list```。

这些```Authentication```实例基于```view```中定义的```authentication_classes```或者是全局的```DEFAULT_AUTHENTICATORS```的设置。

同样的，通常不需要访问这个属性。

源码位置：```rest_framework.view.APIView```类中的```get_authenticators```方法。

---

**注意：** 当调用```.user```或者```.auth```属性的时候，你可能会看到```WrappedAttributeError```异常。

---

### 浏览器增强

REST framework支持一些增强的浏览器，比如基于浏览器表单的```PUT```、```PATCH```和```DELETE```。

#### ```.method```

REST framework中的```request.method```会返回请求的HTTP方法的大写字符串。

基于浏览器的```PUT```、```PATCH```和```DELETE```也支持。

更多信息查看 [浏览器增强]()

#### ```.content_type```

REST framework中的```request.content_type```返回一个字符串对象，表示当前请求的内容主体的类型(media type of the HTTP request's body)。如果没有提供，那么返回空。

通常不需要直接访问请求内容的类型，大多数情况REST framework默认的行为就够用了。

如果你真的需要访问请求内容的类型，那么应该使用```.content_type```而不是```request.META.get('HTTP_CONTENT_TYPE')```。因为```.content_type```透明的支持基于浏览器的non-form内容。

更多信息查看 [浏览器增强]()


#### ```.stream```

`REST framework中的``request.stream```返回一个stream代表请求主体的内容。

通常不需要直接访问请求内容的类型，大多数情况REST framework默认的行为就够用了。

---

### 标准的```HttpRequest```属性

REST framework的```Request```作为标准的Django的```HttpRequest```扩展，肯定的一点是Django标准的属性和方法都可以正常使用。比如```request.META```和```request.session```都能正常的使用。

注意： REST framework的```Request```并不是Django的```HttpRequest```类的继承。
