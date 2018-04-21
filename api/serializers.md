## 序列化器

> 扩展serializers的有用性是我们想要解决的问题。但是，这不是一个微不足道的问题，而是需要一些严肃的设计工作。  
— Russell Keith-Magee,Django用户组

序列化器允许把像 查询结果集(```queryset```)和Model实例这样复杂的数据类型转换为Python原生的数据类型，以便于更加容易的将它渲染称为JSON，XML或者其他的内容类型。序列化器还提供反序列化的功能，允许在验证传入数据的有效性之后，将它们解析回复杂的数据类型。

REST framework中的serializers与Django的```Form```和```ModelForm```类非常像。我们提供了一个```Serializer```类，它为你提供了强大的, 通用方法来控制响应的输出，以及一个```ModelSerializer```类，它为创建用于处理模型实例和查询结果集的序列化程序提供了有用的快捷实现方式。

#### 声明序列化器

让我们先创建一个简单的对象来开始下面的例子：

```python
from datetime import datetime

class Comment(object):
    def __init__(self, email, content, created=None):
        self.email = email
        self.content = content
        self.created = created or datetime.now()

comment = Comment(email='leila@example.com', content='foo bar')
```

我们将会定义一个序列化器，用来序列化和反序列化```Comment```的对象。

定义一个序列化器看起来跟定义一个Django的Form是一样的：

```python
from rest_framework import serializers

class CommentSerializer(serializers.Serializer):
    email = serializers.EmailField()
    content = serializers.CharField(max_length=200)
    created = serializers.DateTimeField()
```

#### 序列化对象

现在，我们使用```CommentSerializer```来序列化一个```comment```对象，或者是一组```comment```列表。同样的，使用```Serializer```类看起来和Django的```Form```是一样的。

```python
In [6]: serializer = CommentSerializer(comment)

In [7]: serializer.data
Out[7]:
ReturnDict([('email', 'leila@example.com'),
            ('content', 'foo bar'),
            ('created', '2018-03-17T13:06:59.048567Z')])
```

此时，我们已经将模型实例转换成了Python常规的数据类型。序列化最终的过程是，需要将数据渲染成JSON。

```python
In [12]: from rest_framework.renderers import JSONRenderer

In [13]: json = JSONRenderer().render(serializer.data)

In [14]: json
Out[14]: b'{"email":"leila@example.com","content":"foo bar","created":"2018-03-17T13:06:59.048567Z"}'
```

注意： 这里的数据类型是bytes类型。

#### 反序列化对象

反序列化同样很简单，首先，我们需要将数据流stream解析成为正常的Python数据类型。

```python
In [22]: from django.utils.six import BytesIO

In [23]: from rest_framework.parsers import JSONParser

In [26]: stream = BytesIO(json)

In [27]: data = JSONParser().parse(stream)

In [28]: data
Out[28]:
{'content': 'foo bar',
 'created': '2018-03-17T13:06:59.048567Z',
 'email': 'leila@example.com'}

In [29]: type(data)
Out[29]: dict
```

然后将这些原生的数据类型恢复成已经验证的数据字典中：

```python
In [30]: serializer = CommentSerializer(data=data)

In [31]: serializer.is_valid()
Out[31]: True

In [32]: serializer.validated_data
Out[32]:
OrderedDict([('email', 'leila@example.com'),
             ('content', 'foo bar'),
             ('created',
              datetime.datetime(2018, 3, 17, 13, 6, 59, 48567, tzinfo=<UTC>))])

In [33]:
```

#### 保存实例

如果我们想要返回基于已经验证的有效数据的完整对象实例，我们需要实现```.create()```和```.update()```方法。比如下面这样：

```python
class CommentSerializer(serializers.Serializer):
    email = serializers.EmailField()
    content = serializers.CharField(max_length=120)
    created = serializers.DateTimeField()

    def create(self, validated_data):
        return Comment(**validated_data)

    def update(self, instance, validated_data):
        instance.email = validated_data.get('email', instance.email)
        instance.content = validated_data.get('content', instance.content)
        instance.created = validated_data.get('created', instance.created)
        return instance
```

如果你的对象实例是Django的模型实例，你想要确保这些方法会将这些数据保存进数据库。比如，如果```Comment```是一个Django模型，那么这些方法应该这么写：

```python
    def create(self, validated_data):
        return Comment.objects.create(**validated_data)  # 注意

    def update(self, instance, validated_data):
        instance.email = validated_data.get('email', instance.email)
        instance.content = validated_data.get('content', instance.content)
        instance.created = validated_data.get('created', instance.created)
        instance.save()  # 注意这里
        return instance
```

现在，当反序列化数据，我们可以调用```.save()```来返回对象实例，返回的是已经验证的有效的对象(如果是Django模型，那么是已经存入到数据中的数据):

```python
comment = serializer.save()
```

注意： 代码已经被修改，要重新导入模块尝试。

说明： ```.save()```会调用```.create()```或者是```.update()```。 仔细看下面的说明。

调用```.save()```将会创建一个新的实例，或者更新一个现有的实例，这取决于在实例化这个序列化器类的时候，是否传入了一个已经存在的数据对象，比如下面这样：

```python
# .save() 这样将会创建一个新的实例
serializer = CommentSerializer(data=data)

# .save() 这样将会更新一个现有的 `comment` 实例
serializer = CommentSerializer(comment, data=data)
```

方法```.create()```和```.update()```方法都是可选的，你可以根据你自己的序列化器类，来实现他们其中的一个，或者都实现，或者都不实现。

#### 给```.save()```方法传递额外的参数

有些时候，你想要在你的视图代码中可以被注入其他的数据，在保存实例的时候。这个额外的数据，可能是当前请求的用户，当前的时间，或者一些其他数据。

你可以在调用```.save()```方法的时候传入额外的数据。比如这样：

```python
serializer.save(owner=request.user)
```

任何其他的关键字参数都会包含在```validated_data```中，当```.create()```或者```.update()```被调用的时候。

注意： ```.save()```方法首先调用，那么额外参数就会到```validated_data```中，那么后续才会调用```.create()```或者```.update()```方法，这两个方法才是真正保存数据到数据库的方法。此时在这两个方法中通过```validated_data```就可以获取到额外的数据。

来看```rest_framework.serializers.BaseSerializer```中```.save()```源代码实现：

```python
    def save(self, **kwargs):
        …… 省略 ……

        validated_data = dict(
            list(self.validated_data.items()) +
            list(kwargs.items())   # 看这里, 这里就是额外的参数
        )

        if self.instance is not None:
            self.instance = self.update(self.instance, validated_data)
            assert self.instance is not None, (
                '`update()` did not return an object instance.'
            )
        else:
            self.instance = self.create(validated_data)
            assert self.instance is not None, (
                '`create()` did not return an object instance.'
            )

        return self.instance
```

#### 直接重写```.save()```方法

在某些场景中，```.create()```和```.update```的方法命名可能没有什么意义。比如，在联系人的一个表单中，我们可能不是创建一个新的实例，取而代之的可能是发送一封邮件或者其他信息。

在这些场景下，你可以选择直接重写```.save()```方法来使得他变得更可读更有意义。

```python
class ContactForm(serializers.Serializer):
    email = serializers.EmailField()
    message = serializers.CharField()

    def save(self):
        email = self.validated_data['email']    # 看这里, 直接访问 validated_data 属性
        message = self.validated_data['message']
        send_email(from=email, message=message)
```

注意：在上述情况下，我们现在不得不直接访问serializer的```.validated_data```属性。

#### 验证

在反序列化的时候，在访问有效数据(```self.validated_data```)之前或者在保存数据之前，你始终需要先调用```is_valid()```方法。如果发生任何的验证错误，那么在```.errors```属性中将包含表示生成的错误消息的字典。比如：

```python
serializer = CommentSerializer(data={'email': 'foobar', 'content': 'baz'})
serializer.is_valid()
# False
serializer.errors
# {'email': [u'Enter a valid e-mail address.'], 'created': [u'This field is required.']}
```

字典里的每一个键都是字段名称，值是与该字段对应的任何错误消息的字符串列表。```non_field_errors```键可能存在，它将列出任何一般验证错误信息。```non_field_errors```的名称可以通过REST framework设置中的```NON_FIELD_ERRORS_KEY```来自定义。

注意：当反序列化一个列表的时候，返回的也是一个由字典组成的列表，列表中的每个item表示对应的对象的详细错误。

#### 无效数据引发的异常

方法```.is_valid()```有个可选的标志```raise_exception```，如果存在验证错误将会抛出一个```serializers.ValidationError```异常。

这些异常由REST framework提供的默认异常处理程序自动处理，默认情况下将返回```HTTP 400 Bad Request```响应。

```python
# Return a 400 response if the data was invalid.
serializer.is_valid(raise_exception=True)
```

#### 字段级别的验证

你可以明确的指定字段级别的数据校验，通过给serializer子类中定义```validate_<field_name>```方法。这看起来跟Django forms的```clean_<field_name>```非常的类似。

这个字段接收单个参数，即需要验证的字段的值。

你的```validate_<field_name>```方法应该返回一个有效的值或者抛出错误```serializers.ValidationError```异常。比如：

```python
from rest_framework import serializers

class BlogPostSerializer(serializers.Serializer):
    title = serializers.CharField(max_length=100)
    content = serializers.CharField()

    def validate_title(self, value):
        """
        Check that the blog post is about Django.
        """
        if 'django' not in value.lower():
            raise serializers.ValidationError("Blog post is not about Django")
        return value
```

---

注意：如果在你的序列化器中声明```<field_name>```字段的时候附加了参数```required=False```，那么当该字段不被包含的时候此时验证步骤是不会发生的。

关于字段级别的数据校验的源码位置：```rest_framework.serializers``` 中的```Serializer``` 类的```to_internal_value```方法。

源码内容如下：

```python
    def run_validation(self, data=empty):
        """
        We override the default `run_validation`, because the validation
        performed by validators and the `.validate()` method should
        be coerced into an error dictionary with a 'non_fields_error' key.
        """
        (is_empty_value, data) = self.validate_empty_values(data)
        if is_empty_value:
            return data

        value = self.to_internal_value(data)  # 注意这里
        try:
            self.run_validators(value)
            value = self.validate(value)
            assert value is not None, '.validate() should return the validated data'
        except (ValidationError, DjangoValidationError) as exc:
            raise ValidationError(detail=as_serializer_error(exc))

        return value

    def to_internal_value(self, data):
        """
        Dict of native values <- Dict of primitive datatypes.
        """
        if not isinstance(data, Mapping):
            message = self.error_messages['invalid'].format(
                datatype=type(data).__name__
            )
            raise ValidationError({
                api_settings.NON_FIELD_ERRORS_KEY: [message]
            }, code='invalid')

        ret = OrderedDict()
        errors = OrderedDict()
        fields = self._writable_fields

        for field in fields:
            validate_method = getattr(self, 'validate_' + field.field_name, None)  # 注意看这里, 获取字段级别的验证方法
            primitive_value = field.get_value(data)
            try:
                validated_value = field.run_validation(primitive_value)
                if validate_method is not None:
                    validated_value = validate_method(validated_value) # 注意这里, 验证完原始值之后, 开始使用自定义的字段级别的验证方法验证
            except ValidationError as exc:
                errors[field.field_name] = exc.detail
            except DjangoValidationError as exc:
                errors[field.field_name] = get_error_detail(exc)
            except SkipField:
                pass
            else:
                set_value(ret, field.source_attrs, validated_value)

        if errors:
            raise ValidationError(errors)

        return ret
```

---

#### 对象级别的验证

如果要对对象级别的数据进行验证，那么实质上就是要对多个字段进行校验，可以在你定义的```Serializer```子类中定义```validate()```方法。这个方法接收一个参数，是一个字典并带有所有的字段值。这个方法最后应该返回一个有效的数据，或者抛出```ValidationError``` 错误。比如：

```python
from rest_framework import serializers

class EventSerializer(serializers.Serializer):
    description = serializers.CharField(max_length=100)
    start = serializers.DateTimeField()
    finish = serializers.DateTimeField()

    def validate(self, data):
        """
        Check that the start is before the stop.
        """
        if data['start'] > data['finish']:
            raise serializers.ValidationError("finish must occur after start")
        return data
```

关于字段级别的数据校验的源码位置：```rest_framework/serializers.py``` 中的```Serializer``` 类的```validate```方法，源代码中并没有提供任何数据校验的逻辑，所以我们可以重写这个方法。

```python
    def validate(self, attrs):
        return attrs
```

#### 验证器

序列化器上的每个字段都可以包含验证器。可以在序列化器中声明字段的时候包含自定义的校验器，如下：

```python
def multiple_of_ten(value):
    if value % 10 != 0:
        raise serializers.ValidationError('Not a multiple of ten')

class GameRecord(serializers.Serializer):
    score = IntegerField(validators=[multiple_of_ten])
    ...
```

序列化器类还可以包括应用于一组字段数据的可重用的验证器。这些验证器要在内部的```Meta```类中声明，如下所示：

```python
class EventSerializer(serializers.Serializer):
    name = serializers.CharField()
    room_number = serializers.IntegerField(choices=[101, 102, 103, 201])
    date = serializers.DateField()

    class Meta:
        # 每个room_number每天只能有一个event  其实就是联合唯一索引的验证
        validators = UniqueTogetherValidator(
            queryset=Event.objects.all(),
            fields=['room_number', 'date']
        )
```

更多的校验器参考：[验证器](./validators.md)

---

#### 访问初始数据和实例

当传递一个初始化对象或者查询结果集传递给序列化实例的时候，可以通过```.instance```访问对象。如果没有传递初始化对象，那么```.instance```属性将是None。

```python
>>> obj = models.User.objects.get(pk=2)
>>> obj_seria = serializers.UserSerializer(obj)
>>> obj_seria.instance
<User: 2@2.com>
>>> obj_seria.instance.email
u'2@2.com'
```

当传递给serializer实例一个data关键字参数，并且没有做出修改工作的时候，可以通过```.initial_data```属性来访问这些数据。如果没有传递data关键字参数，那么```.initial_data```属性就不存在。

```python
>>> new_serializer = serializers.CommentSerializer(data=new_data)
>>> new_serializer.initial_data
{'content': 'test_data', 'email': '2@2.com', 'created': '2018-03-03T11:58:29'}
```

源码位置在: ```rest_framework/serializers.py```中的```BaseSerializer```类的构造函数中。

```python
class BaseSerializer(Field):
    """
    BaseSerializer类被用于自定义序列化器的实现

    Note that we strongly restrict the ordering of operations/properties
    that may be used on the serializer in order to enforce correct usage.

    当传入 `data=` 参数的时候(反序列化会传入data参数, 反序列化用于提交并创建数据):

    .is_valid() - Available.
    .initial_data - Available.
    .validated_data - Only available after calling `is_valid()`
    .errors - Only available after calling `is_valid()`
    .data - Only available after calling `is_valid()`

    当没有传入 `data=` 参数的时候(序列化通常会给一个模型对象(instance参数)，就是指定一个数据对象, 序列化用于获取对象):

    .is_valid() - Not available.
    .initial_data - Not available.
    .validated_data - Not available.
    .errors - Not available.
    .data - Available.
    """

    def __init__(self, instance=None, data=empty, **kwargs):
        self.instance = instance  # 看这里 关键
        if data is not empty:   #  看这里 关键
            self.initial_data = data
        self.partial = kwargs.pop('partial', False)
        self._context = kwargs.pop('context', {})
        kwargs.pop('many', None)
        super(BaseSerializer, self).__init__(**kwargs)

    def __new__(cls, *args, **kwargs):
        # We override this method in order to automagically create
        # `ListSerializer` classes instead when `many=True` is set.
        if kwargs.pop('many', False):
            return cls.many_init(*args, **kwargs)
        return super(BaseSerializer, cls).__new__(cls, *args, **kwargs)
```

#### 部分更新

默认情况下，serializer必须传入所有依赖字段的值，否则将会抛出数据无效的错误。然而你可以使用```partial```参数，来实现局部更新，也就是```PATCH```方法。

```python
# 局部更新comment
serializer = CommentSerializer(comment, data={'content': u'foo bar'}, partial=True)
```

#### 处理嵌套对象

前面的例子都是使用的简单的数据类型，但是有的时候我们可能需要更加复杂的数据类型，对象的某些属性可能不只是简单的数据类型，比如字符串，日期或者整数。

Serializer类本身就是一个字段类型(```Field```)(可查看源码)。所以就可以用来表示当一个对象嵌套在另一个对象中的关系。 比如：

```python
class UserSerializer(serializers.Serializer):
    email = serializers.EmailField()
    username = serializers.CharField(max_length=100)

class CommentSerializer(serializers.Serializer):
    user = UserSerializer()  # 实质上是外键关系，多对一关系
    content = serializers.CharField(max_length=200)
    created = serializers.DateTimeField()
```

如果嵌套关系选项可以接收一个```None```值，你应该传递```required=False```标志给嵌套序列。

```python
class CommentSerializer(serializers.Serializer):
    user = UserSerializer(required=False)  # 由于可能为None，所以可能是一个匿名用户
    content = serializers.CharField(max_length=200)
    created = serializers.DateTimeField()
```

同样的道理，如果嵌套的关系字段可能是多个对象组成的列表，那么应该传递```many=True```标志给嵌套的序列器

```python
class CommentSerializer(serializers.Serializer):
    user = UserSerializer(required=False)
    edits = EditItemSerializer(many=True)  # 一个嵌套的列 , 其实就是多对多关系
    content = serializers.CharField(max_length=200)
    created = serializers.DateTimeField()
```

#### 可写的嵌套对象表示

当含有嵌套关系serializer进行反序列化的时候，如果有任何的错误都将会嵌套再对应的```field```中。同样的，如果数据检验有效，也会再```.validated_data```属性中包含嵌入的数据结构。

```python
>>> serializer = CommentSerializer(data={'user': {'email': 'foobar', 'username': 'doe'}, 'content': 'baz'})
>>> serializer.is_valid()
False
>>> serializer.errors
{'user': {'email': [u'Enter a valid e-mail address.']}, 'created': [u'This field is required.']}
```

#### 为可写的嵌套关系重构```.create()```方法

如果想要嵌套关系支持写入，那么必须重构```create()```方法和```update()```方法，让它们能够处理多个对象。

下面是一个```create()```方法的例子, 演示如何处理创建一个具有嵌套的概要信息对象的用户。

```python
class UserSerializer(serializers.ModelSerializer):
    profile = ProfileSerializer()

    class Meta:
        model = User
        fields = ('username', 'email', 'profile')

    def create(self, validated_data):
        profile_data = validated_data.pop('profile')
        user = User.objects.create(**validated_data)  # 看这里
        Profile.objects.create(user=user, **profile_data)  # 看这里
        return user
```

#### 为可写的嵌套关系重构```.update()```方法

关于```update()```方法, 你要非常小心的考虑如何处理关联字段的更新。如果说关系数据没有提供，或者为```None```，那么下面哪种情况会发生？

- 在数据库中设置关联字段为```NULL```
- 删除相关联的实例
- 忽略数据并且保留实例
- 抛出验证错误

下面是一个关于```UserSerializer```类中```update()```方法的例子：

```python
    def update(self, instance, validated_data):
        profile_data = validated_data.pop('profile')
        # Unless the application properly enforces that this field is
        # always set, the follow could raise a `DoesNotExist`, which
        # would need to be handled.
        profile = instance.profile

        instance.username = validated_data.get('username', instance.username)
        instance.email = validated_data.get('email', instance.email)
        instance.save()

        profile.is_premium_member = profile_data.get(
            'is_premium_member',
            profile.is_premium_member
        )
        profile.has_support_contract = profile_data.get(
            'has_support_contract',
            profile.has_support_contract
         )
        profile.save()

        return instance
```

由于嵌套关系的```create```和```update```的行为是不明确的，并且需要相关模型之间复杂的依赖关系。所以在REST framework 3.x版本中必须要显示的指定这些方法。默认的```ModelSerializer```的```create()```方法和```update()```方法不支持嵌入关系的可写。

#### 在model manager class中处理保存关联实例

在serializer中保存多个关系实例有一个替代方案是使用Django 的model ```manager class```。

比如，假如我们想要确保User和Profile总是成对创建的，我们就可以写一个自定义的```manager class```来实现它们：

```python
class UserManager(models.Manager):
    ...

    def create(self, username, email, is_premium_member=False, has_support_contract=False):
        user = User(username=username, email=email)
        user.save()
        profile = Profile(
            user=user,
            is_premium_member=is_premium_member,
            has_support_contract=has_support_contract
        )
        profile.save()
        return user
```

这个```manager class```现在更好的封装了User实例和Profile实例总是在同一时间创建。现在在serializer class中的```create()```方法，直接使用Django model的```create```方法就能实现多个实例。

```python
def create(self, validated_data):
    return User.objects.create(
        username=validated_data['username'],
        email=validated_data['email']
        is_premium_member=validated_data['profile']['is_premium_member']
        has_support_contract=validated_data['profile']['has_support_contract']
    )
```

更多关于Django Model Manager的文档，查看[managers](https://docs.djangoproject.com/en/2.0/topics/db/managers/)
和[经典博文](https://www.dabapps.com/blog/django-models-and-encapsulation/)

#### 处理多个对象

Serializer类也能序列化和反序列化一个对象列表。

##### 序列化多个对象

为了能够序列化一个查询结果集或者是一个对象列表，而不是一个单个对象实例，所以在实例化序列化器类的时候传一个```many=True```参数。这样就能序列化一个查询集或一个对象列表。

```python
queryset = Book.objects.all()
serializer = BookSerializer(queryset, many=True)
serializer.data
# [
#     {'id': 0, 'title': 'The electric kool-aid acid test', 'author': 'Tom Wolfe'},
#     {'id': 1, 'title': 'If this is a man', 'author': 'Primo Levi'},
#     {'id': 2, 'title': 'The wind-up bird chronicle', 'author': 'Haruki Murakami'}
# ]
```

在源码文件的```rest_framework.mixins.ListModelMixin```有同样的例子，该函数用来序列化一组实例：

```python
class ListModelMixin(object):
    """
    List a queryset.
    """
    def list(self, request, *args, **kwargs):
        queryset = self.filter_queryset(self.get_queryset())

        page = self.paginate_queryset(queryset)
        if page is not None:
            serializer = self.get_serializer(page, many=True)
            return self.get_paginated_response(serializer.data)

        serializer = self.get_serializer(queryset, many=True)
        return Response(serializer.data)
```

##### 反序列化多个对象

默认情况下，反序列化得行为是支持多个对象得创建得，但是不支持多个对象得更新操作，具体查看下文的```ListSerializer```部分。

#### 包含额外的上下文

在一些特殊的场景中，除了要序列化对象之外，你可能需要传递给serializer一些额外的上下文信息。一个常见的例子就是，当serializer中使用了超链接的时候，就可以通过额外的上下文开控制超链接是一个绝对地址还是相对地址。

可以在实例化序列化器的时候传递一个context参数来传递任意的附加上下文。例如：

```python
serializer = AccountSerializer(account, context={'request': request})
serializer.data
# {'id': 6, 'owner': u'denvercoder9', 'created': datetime.datetime(2013, 2, 12, 09, 44, 56, 678870), 'details': 'http://example.com/accounts/6/details'}
```

额外的上下文的字典内容可以在serializer内部的任何地方使用，比如自定义的```.to_representation()```方法中可以通过访问```self.context```属性获取上下文字典。甚至是自定义的```create()```方法和```update()```方法都可以通过```self```来访问。

```python
class DepartmentSerializer(serializers.HyperlinkedModelSerializer):
    url = serializers.HyperlinkedIdentityField(view_name="users:department-detail")
    permissions = serializers.HyperlinkedRelatedField(view_name='users:permission-detail', lookup_field='pk', many=True, read_only=True)
    class Meta:
        model = models.Department
        fields = ('url', 'id', 'name', 'permissions')
    def to_representation(self, instance):
        print(self.context)
        return super(DepartmentSerializer, self).to_representation(instance)
```

现在通过Python Django Shell执行如下的命令：

```python
>>> from users import serializers
>>> from users import models
>>> g2 = models.Department.objects.get(pk=2)
>>> g2_seria = serializers.DepartmentSerializer(g2, context={'request': None})
>>> g2_seria.to_representation(g2)
{'request': None}
OrderedDict([('url', u'/users/departments/2/'), ('id', 2), ('name', u'g2'), ('permissions', [u'/users/permissions/4/'])])
>>> g2_seria.data
{'request': None}
{'url': u'/users/departments/2/', 'permissions': [u'/users/permissions/4/'], 'id': 2, 'name': u'g2'}
```

<br />
<br />
<br />

---

### ```ModelSerializer```类

通常，你想要序列化器类和你的Django Model定义密切的对应。

幸好有```ModelSerializer```类，它能帮助你快速的自动的来创建一个跟Django Model字段对应的```Serializer```类。

这个```ModelSerializer```和```Serializer```类基本是一样的，除了一下几点：

- 它会自动的为你生成一组字段，基于你的Django Model字段
- 它会自动的为你生成一些验证，比如```unique_together```
- 默认实现了简单的```.create()```方法和```.update()```方法

定义一个```ModelSerializer```是这样的：

```python
class AccountSerializer(serializers.ModelSerializer):
    class Meta:
        model = Account
        fields = ('id', 'account_name', 'users', 'created')
```

默认情况下，所有的模型的字段都将映射到序列化器上相应的字段。

模型中任何关联字段比如外键都将映射到```PrimaryKeyRelatedField```字段。默认情况下不包括反向关联，除非像```[serializer relations](./serializersrelat.md)```文档中规定的那样显示包含。

##### 检查```ModelSerializer```

序列化类生成详细信息，允许你全面检查其字段的状态。 这在使用ModelSerializers时特别有用，因为你想确定自动创建了哪些字段和验证器。

要检查的话，打开Django shell,执行 ```python manage.py shell```，然后导入序列化器类，实例化它，并打印对象：

```python
>>> from myapp.serializers import AccountSerializer
>>> serializer = AccountSerializer()
>>> print(repr(serializer))
AccountSerializer():
    id = IntegerField(label='ID', read_only=True)
    name = CharField(allow_blank=True, max_length=100, required=False)
    owner = PrimaryKeyRelatedField(queryset=User.objects.all())
```

#### 指定包含的字段

如果只希望在模型序列化器中使用默认字段的子集，你可以使用```fields```或```exclude```选项来执行此操作，就像使用```ModelForm```一样。强烈建议你使用```fields```属性显示的设置要序列化的字段。这样就不会因为你修改了模型而无意中暴露了数据。

比如：

```python
class AccountSerializer(serializers.ModelSerializer):
    class Meta:
        model = Account
        fields = ('id', 'account_name', 'users', 'created')
```

你也可以将```fields```属性设置成```'__all__'```，来使用模型中所有的字段。

比如：

```python
class AccountSerializer(serializers.ModelSerializer):
    class Meta:
        model = Account
        fields = '__all__'
```

你还可以将```exclude```属性设置为，你序列化器要排除的字段的列表。

比如：

```python
class AccountSerializer(serializers.ModelSerializer):
    class Meta:
        model = Account
        exclude = ('users',)
```

在上面的例子中，如果```Account```模型有三个字段```account_name```、```users```和```created```，那么最后的结果是只有```account_name```和```created```会被序列化。

上面```exclude```和```fields```属性中的名称通常会映射到模型类中的模型字段名称。

或者```fields```选项中的名称可以映射到模型类中不存在任何参数的属性或方法。

从版本3.3.0开始，强制必须使用```fields```或者```exclude```属性。

#### 指定嵌套序列化

默认```ModelSerializer```使用主键来进行关联，但是你也可以使用```depth```选项来轻松的生成嵌套关联：

```python
class AccountSerializer(serializers.ModelSerializer):
    class Meta:
        model = Account
        fields = ('id', 'account_name', 'users', 'created')
        depth = 1
```

选项```depth```任何时候都只能被设置为整数，它表示要遍历的关系的深度。

如果要自定义序列化的方式你需要自定义该子段。

#### 显式指定字段

在```ModelSerializer```中你可以增加一个额外的字段，或者通过定义一个字段来覆盖重写默认的字段。就跟在```Serializer```类中一样那么简单。

```python
class AccountSerializer(serializers.ModelSerializer):
    url = serializers.CharField(source='get_absolute_url', read_only=True)  # 额外的字段
    groups = serializers.PrimaryKeyRelatedField(many=True)   # 重写默认的字段

    class Meta:
        model = Account
```

额外的字段可以对应于模型上的任何属性或可调用的字段。

#### 指定只读字段

你可能希望指定多个字段为只读。而不是为每个字段都添加```read_only=True```属性(就像上面的```url```字段)。这种情况你可以使用```read_only_fields```Meta选项。

这个选项的值应该是由字段名组成的一个列表或者元组。如下：

```python
class AccountSerializer(serializers.ModelSerializer):
    class Meta:
        model = Account
        fields = ('id', 'account_name', 'users', 'created')
        read_only_fields = ('account_name',)
```

模型字段中有设置```editable=False```和默认为只读的```AutoField```字段，不需要增加```read_only_fields```选项。

---

注意：有一种特殊情况，其中一个只读字段是模型级别```unique_together```(联合唯一约束)约束的一部分。在这种情况下，序列化器类需要该字段才能验证约束，但也不能由用户编辑。

处理此问题的正确方法是在序列化器上显式指定该字段，同时提供```read_only=True```和```default=…```关键字参数。

这种情况的一个例子就是当前认证的```User```是只读的并且有```unique_together```验证。 在这种情况下你可以像下面这样声明```user```字段：

```python
user = serializers.PrimaryKeyRelatedField(read_only=True, default=serializers.CurrentUserDefault())
```

详情查看[Validators](./validators.md)。

---

#### 额外的关键字参数

你还可以通过使用```extra_kwargs```选项方便快捷的在字段上指定任意的额外关键字参数。在```read_only_fields```这种情况下，你不需要显式的声明在序列化器上的字段。

这个选项是一个将具体字段名称当作键值的字典。例如：

```python
class CreateUserSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = ('email', 'username', 'password')
        extra_kwargs = {'password': {'write_only': True}}

    def create(self, validated_data):
        user = User(
            email=validated_data['email'],
            username=validated_data['username']
        )
        user.set_password(validated_data['password'])
        user.save()
        return user
```

#### 关系字段

当序列化模型实例的时候，你可以选择多种不同的方式来表示关联关系。对于```ModelSerializer```默认是使用关联实例的主键。

替代的其他方法包括使用 '超链接的序列化'，'完整的序列化嵌套'或者使用'自定义的序列化'。

更多详细信息请查阅[serializer relations](./serializersrelat.md)文档。

#### 自定义字段映射

对于```ModelSerializer```的自动映射，它还提供了一个API来专门改变这种行为。以便于在实例化序列化器的时候改变序列化器字段的自动映射。

通常情况下，如果```ModelSerializer```没有生成默认情况下你需要的字段，那么你应该将它们显式地添加到类中，或者直接使用常规的```Serialize```r类。但是在某些情况下，你可能需要创建一个新的基类，来定义给任意模型创建序列化字段的方式。

**```.serializer_field_mapping```**  
Django Model类到REST framework序列化器类的映射。你可以重写此映射来更改每个模型类应该使用的默认序列化器类。

**```.serializer_related_field```**  
此属性应该是序列化程序字段类，默认情况下用于关联字段。

对于```ModelSerializer```默认是```PrimaryKeyRelatedField```

对于```HyperlinkedModelSerializer```默认是```serializers.HyperlinkedRelatedField```

**```.serializer_url_field```**  
这个序列化字段类，应该被用于序列化器中的任何```url```字段。

默认是```serializers.HyperlinkedIdentityField```

**```.serializer_choice_field```**  
这个序列化字段类，应该被用于序列化器中的任何```choice```字段。

默认是```serializers.ChoiceField```

##### ```field_class``` 和 ```field_kwargs``` 的API

调用下面的方法来确定应该被自动包含在序列化器类中每个字段的类和关键字参数。这些方法都应该返回两个 ```(field_class, field_kwargs)```元组。

**```.build_standard_field(self, field_name, model_field)```**  
调用来生成一个映射到标准字段的序列化字段。

默认实现是根据```serializer_field_mapping```属性返回一个序列化器类。

**```.build_relational_field(self, field_name, relation_info)```**  
调用来生成一个映射到关系字段的序列化字段。

默认实现是根据```serializer_relational_field```属性返回一个序列化器类。

参数```relation_info```是一个命名元组。它包含```model_field```、```related_model```、```to_many```和```has_through_model```属性。

**```.build_nested_field(self, field_name, relation_info, nested_depth)```**  
当```depth```选项被设置时，调用生成一个映射到关联模型字段的序列化器字段。

默认实现是动态创建一个基于```ModelSerializer```或```HyperlinkedModelSerializer```的嵌套的序列化器类。

关于```nested_depth```的值是```depth```的值减1。

关于```relation_info```参数是一个命名元祖，包含```model_field```、```related_model```、```to_many```和```has_through_model```属性。

**```.build_property_field(self, field_name, model_class)```**  
调用后生成一个映射到模型类中属性或无参数方法的序列化器字段。

默认实现是返回一个```ReadOnlyField```类。

**```.build_url_field(self, field_name, model_class)```**  
调用后为序列化器自己的```url```字段生成一个序列化器字段。默认实现是返回一个```HyperlinkedIdentityField```类。

**```.build_unknown_field(self, field_name, model_class)```**  
当字段名称没有映射到任何模型字段或者模型属性的时候调用。 默认实现会抛出一个错误，尽管子类可能会自定义该行为。

<br />
<br />
<br />

---

### ```HyperlinkedModelSerializer```类

关于```HyperlinkedModelSerializer```类和```ModelSerializer```类是基本一样的，除了使用超链接来表示关系映射，而不是使用主键。

默认情况下序列化器会包含一个```url```字段而不是主键字段。

这个```url```字段使用```HyperlinkedIdentityField```序列化字段来表示。并且其他任何的模型关系字段都会使用```HyperlinkedIdentityField```序列化字段。

当然你可以在```fields```选项中明确的指定包含主键字段。比如：

```python
class AccountSerializer(serializers.HyperlinkedModelSerializer):
    class Meta:
        model = Account
        fields = ('url', 'id', 'account_name', 'users', 'created')
```

#### 绝对和相对URL

当实例化一个```HyperlinkedModelSerializer```的时候， 你必须在序列化器的```context```中包含当前的```request```，比如：

```python
serializer = AccountSerializer(queryset, context={'request': request})
```

这样做可以确保超链接包含适当的```hostname```，以便于生成一个完整的URLs， 比如：

```
http://api.example.com/accounts/1/
```

而不是一个相对的URLs，比如：
 
```
/accounts/1/
```

如果你就是要使用相对的URLs， 那么你需要明确的给序列化器的```context```传递一个```{'request': None}```。

#### 如何确定超链接视图

需要一种方法确定哪些视图应该被用于超链接到模型实例。

默认情况下，超链接期望一个视图名称能匹配```'{model_name}-detail'```这样的风格。并且通过一个```pk```关键字参数来查询实例。

你可以在```extra_kwargs```设置中配置```view_name```和```lookup_field```选项来重写URL字段和查询字段。比如这样：

```python
class AccountSerializer(serializers.HyperlinkedModelSerializer):
    class Meta:
        model = Account
        fields = ('account_url', 'account_name', 'users', 'created')
        extra_kwargs = {
            'url': {'view_name': 'accounts', 'lookup_field': 'account_name'},
            'users': {'lookup_field': 'username'}
        }
```

或者你可以显示的在你的序列化字段上进行设置，比如下面这样：

```python
class AccountSerializer(serializers.HyperlinkedModelSerializer):
    url = serializers.HyperlinkedIdentityField(
        view_name='accounts',
        lookup_field='slug'
    )
    users = serializers.HyperlinkedRelatedField(
        view_name='user-detail',
        lookup_field='username',
        many=True,
        read_only=True
    )

    class Meta:
        model = Account
        fields = ('url', 'account_name', 'users', 'created')
```

---

提示： 正确的匹配超链接关系和URL配置有些时候是非常困难的。可以打印```HyperlinkedModelSerializer```实例的```repr```是一个非常有用的方式，用来检查关联关系映射视图名称和查询字段。

---

#### 更改URL字段名称

URL字段的默认名称是```url```。 你可以使用全局配置```URL_FIELD_NAME```来覆盖这一行为。

<br />
<br />
<br />

---

### ```ListSerializer```类

关于```ListSerializer```类提供了一次性序列化和校验多个对象的能力。你通常不需要直接使用```ListSerializer```，但是应该在实例一个序列化器的时候传递一个简单的```many=True```来替代```ListSerializer```。

当一个序列化器被实例化的时候， 并且传递了```many=True```的时候，一个```ListSerializer```将会被创建。该序列化器类将会变成```ListSerializer```的子类。

下面的参数也可以传递给```ListSerializer```字段，或者一个传递了```many=True```的序列化器。

**allow_empty**

这个参数，默认是```True```，但是如果你不想把空列表作为有效输入的话，可以把它设置成```False```。

#### 自定义```ListSerializer```行为

下面一些情况你可能想要自定义```ListSerializer```的行为。比如：

- 你想要为列表提供一个特定的验证，比如你想要检查列表中的一个元素是否和另外一个元素冲突。
- 你想要自定义多个对象的创建或者更新的行为。

对于这些情况，当```many=True```被传递的时候，你可以修改这个类，通过使用序列化器下的```Meta```类的```list_serializer_class```选项。

比如：

```python
class CustomListSerializer(serializers.ListSerializer):
    ...

class CustomSerializer(serializers.Serializer):
    ...
    class Meta:
        list_serializer_class = CustomListSerializer
```

#### 自定义多个对象的创建

默认情况下对于多个对象的创建是简单的调用列表中的每个元素的```.create()```方法。 如果你想自定义这个行为。你可以自定义当```many=True```被传递时，使用的```ListSerializer```类中的```.create()```方法。

比如：

```python
class BookListSerializer(serializers.ListSerializer):
    def create(self, validated_data):
        books = [Book(**item) for item in validated_data]
        return Book.objects.bulk_create(books)

class BookSerializer(serializers.Serializer):
    ...
    class Meta:
        list_serializer_class = BookListSerializer
```

#### 自定义多个字段的更新

默认情况下```ListSerializer```类不支持多个对象的更新。这是因为插入和删除应该预期的行为是不明确的。

如果要支持多个对象的更新，需要自己实现。当实现该功能的时候，你需要注意以下几点内容：

- 如何确定数据列表中的哪个实例应该被更新？
- 如何处理插入的行为？它们是无效的， 或者是创建新的对象？
- 如何处理删除行为？删除单个对象，或者删除关联对象？应该被忽略还是提示无效？
- 如何处理排序行为？改变两个元素的位置，是否意味着状态改变，或者应该被忽略？

你需要往实例序列化器中显示的添加一个```id```字段。默认隐式生成的```id```字段是```read_only```。这会导致在更新的时候被删除。一旦你明确的定义它，它将会在列表序列化器的```update```方法中可用。

下面是一个如何选择多个对象来更新的案例：

```python
class BookListSerializer(serializers.ListSerializer):
    def update(self, instance, validated_data):
        # Maps for id->instance and id->data item.
        book_mapping = {book.id: book for book in instance}
        data_mapping = {item['id']: item for item in validated_data}

        # Perform creations and updates.
        ret = []
        for book_id, data in data_mapping.items():
            book = book_mapping.get(book_id, None)
            if book is None:
                ret.append(self.child.create(data))
            else:
                ret.append(self.child.update(book, data))

        # Perform deletions.
        for book_id, book in book_mapping.items():
            if book_id not in data_mapping:
                book.delete()

        return ret

class BookSerializer(serializers.Serializer):
    # 我们需要使用主键来表示列表中的元素,
    # 所以在着了使用可写字段, 而不是默认的read-only.
    id = serializers.IntegerField()
    ...

    class Meta:
        list_serializer_class = BookListSerializer
```

#### 自定义```ListSerializer```初始化

当一个带有```many=True```的序列化器被实例化的时候，我们需要确定哪些参数和关键字应该被传递给```.__init__()```方法。

默认实现是传递所有的参数，除了```validators```和任何自定义的关键字参数，它们假定被用于子序列化器类。

偶尔，当```many=True```被传递的时候，你可能想要明确的指定子类和父类应该如何被实例化。这个时候你可以使用```many_init```类方法。

比如：

```python
@classmethod
    def many_init(cls, *args, **kwargs):
        # 实例化子序列化器类.
        kwargs['child'] = cls()
        # 实例化父list serializer.
        return CustomListSerializer(*args, **kwargs)
```

<br />
<br />
<br />

---

### ```BaseSerializer```类

关于```BaseSerializer```被用于简单的支持选择性的序列化和反序列化的风格。

这个类实现实现了```Serializer```类基本的API：

- ```.data``` - 返回原始的数据。
- ```.is_valid()``` - 反序列化并且验证进来的数据。
- ```.validated_data``` - 返回已经验证完的进来的数据。
- ```.errors``` - 返回在验证期间的任何错误。
- ```.save()``` - 持久化一个已经验证完的数据到一个对象实例。

有四个方法可以被覆盖，这取决于你的序列化器类想要支持哪种功能：

- ```.to_representation()``` - 覆盖这个用于支持序列化，用于读取操作
- ```.to_internal_value()``` - 覆盖这个用于支持反序列化，用于写入操作
- ```.create()``` 和 ```.update()``` - 覆盖这两种的一个或者全部，来支持保存实例

因为这个类提供了和```Serializer```类相同的接口，所以你可以像现有的常规的```Serializer```或者```ModelSerializer```类一样，将```BaseSerializer```与现有的基于类的视图来一起使用。

这样做的唯一的区别是， 当使用```BaseSerializer```类的时候不会生成可视化API的HTML表单。这是由于返回的数据不包含任何的字段信息。这些字段信息会允许将每个字段呈现为合适的HTML表单元素。

#### 只读```BaseSerializer```类

要使用```BaseSerializer```实现一个只读的序列化器类，我们只需要重写```.to_representation()```方法即可。

下面是一个简单的Django模型：

```python
class HighScore(models.Model):
    created = models.DateTimeField(auto_now_add=True)
    player_name = models.CharField(max_length=10)
    score = models.IntegerField()
```

使用下面的方法创建一个只读的序列化器，用于将```HighScore```实例转换为原始的数据类型：

```python
class HighScoreSerializer(serializers.BaseSerializer):
    def to_representation(self, obj):
        return {
            'score': obj.score,
            'player_name': obj.player_name
        }
```

现在，我们可以使用这个序列化器类来序列化一个```HighScore```实例：

```python
@api_view(['GET'])
def high_score(request, pk):
    instance = HighScore.objects.get(pk=pk)
    serializer = HighScoreSerializer(instance)
    return Response(serializer.data)
```

或者使用它来序列化多个实例：

```python
@api_view(['GET'])
def all_high_scores(request):
    queryset = HighScore.objects.order_by('-score')
    serializer = HighScoreSerializer(queryset, many=True)
    return Response(serializer.data)
```

#### 读写```BaseSerializer```类

要创建一个可以读写的序列化器类，我们首先要实现```.to_internal_value()```方法。这个方法用于返回已经校验的值，这些值用于构造数据实例，并且如果数据的格式不正确，会引发```serializers.ValidationError```异常。

一旦你实现了```.to_internal_value()```，那么在序列化器中的基本的校验API将全部可用，也就是你可以使用```.is_valid()```，```。validated_data```和```.errors```。

如果你想要支持```.save()```， 你需要实现```.create()```或者```.update()```中的一个或者全部。

下面是一个支持可读写的序列化器类：

```python
class HighScoreSerializer(serializers.BaseSerializer):
    def to_internal_value(self, data):
        score = data.get('score')
        player_name = data.get('player_name')

        # 处理数据验证.
        if not score:
            raise serializers.ValidationError({
                'score': 'This field is required.'
            })
        if not player_name:
            raise serializers.ValidationError({
                'player_name': 'This field is required.'
            })
        if len(player_name) > 10:
            raise serializers.ValidationError({
                'player_name': 'May not be more than 10 characters.'
            })

        # 返回已经验证的值. This will be available as
        # the `.validated_data` property.
        return {
            'score': int(score),
            'player_name': player_name
        }

    def to_representation(self, obj):
        return {
            'score': obj.score,
            'player_name': obj.player_name
        }

    def create(self, validated_data):
        return HighScore.objects.create(**validated_data)
```

#### 创建新的基类

如果你想要实现新的通用的序列化器类用于处理特定的序列化风格，或者是用于与替代的后端存储整合，那么使用```BaseSerializer```是非常有用的。

下面是通用序列化器类的例子，它可以处理将对象类型转换为原始类型。

```python
class ObjectSerializer(serializers.BaseSerializer):
    """
    A read-only serializer that coerces arbitrary complex objects
    into primitive representations.
    """
    def to_representation(self, obj):
        for attribute_name in dir(obj):
            attribute = getattr(obj, attribute_name)
            if attribute_name('_'):
                # 忽略私有属性.
                pass
            elif hasattr(attribute, '__call__'):
                # Ignore methods and other callables.
                pass
            elif isinstance(attribute, (str, int, bool, float, type(None))):
                # Primitive types can be passed through unmodified.
                output[attribute_name] = attribute
            elif isinstance(attribute, list):
                # Recursively deal with items in lists.
                output[attribute_name] = [
                    self.to_representation(item) for item in attribute
                ]
            elif isinstance(attribute, dict):
                # Recursively deal with items in dictionaries.
                output[attribute_name] = {
                    str(key): self.to_representation(value)
                    for key, value in attribute.items()
                }
            else:
                # Force anything else to its string representation.
                output[attribute_name] = str(attribute)
```

<br />
<br />
<br />

---

### 高级的序列化器使用

#### 重写序列化和反序列化的行为

如果你想要更改序列化器的序列化和反序列化的行为， 你可以重写```.to_representation()```和```.to_internal_value()```方法。

可能有以下原因：

- 为新的序列化基类增加新的行为
- 为现有的类稍微修改一些行为
- 为频繁访问，返回大量数据的API端点，提高序列化性能

这些方法的签名如下：

---

```python
.to_representation(self, obj)
```

接受需要序列化的对象实例，并返回一个原始对象。通常这意味着返回一个Python內建的数据类型。可以被处理的确切的类型取决于您API配置的渲染器类。

可能会被重写，以便于修改表现样式。比如：

```python
def to_representation(self, instance):
    """Convert `username` to lowercase."""
    ret = super().to_representation(instance)
    ret['username'] = ret['username'].lower()
    return ret
```

---

```python
.to_internal_value(self, data)
```

将未验证的传入的数据作为输入，并返回已经验证的数据，提供给```serializer.validated_data```。如果在序列化器类中调用```.save()```方法，那么这些数据将会被传递给```.create()```或者```.update()```方法。

如果有任何验证失败，那么这个方法应该抛出```serializers.ValidationError(errors)```异常。

这个```errors```参数应该是一个字段名称映射的字典，包含错误信息的列表，或者是```settings.NON_FIELD_ERRORS_KEY```。如果你并不是想要重写反序列化的行为，而只是想简单的提供对象级别的验证，那么建议重写```.validate()```方法。

关于```data```参数，通常是```request.data```，所以这个数据类型取决于您API配置的解析器类。

#### 序列化继承

类似于Django的forms， 你可以通过继承来扩展或者重写序列化器类。这允许你在父类中定义一些通用的字段或者方法，在子类中就可以使用。比如：

```python
class MyBaseSerializer(Serializer):
    my_field = serializers.CharField()

    def validate_my_field(self):
        ...

class MySerializer(MyBaseSerializer):
    ...
```

像Django的```Model```和```ModelForm```类一样，序列化器类中的```Meta```类不会隐式的从父类的```Meta```继承，如果你想要继承父类的```Meta```，你需要明确的指定。比如：

```python
class AccountSerializer(MyBaseSerializer):
    class Meta(MyBaseSerializer.Meta):
        model = Account
```

通常不建议在内部的```Meta```类上使用继承，而是明确声明所有的选项。

此外，使用序列化器继承需要注意以下几点：

- 正常Python的名称解析规则适用。如果有多个基类定义了```Meta```内部类，那么第一个会被使用。
- 可以在子类中使用```None```来显式的删除从父类继承过来的字段。

```python
class MyBaseSerializer(ModelSerializer):
    my_field = serializers.CharField()

class MySerializer(MyBaseSerializer):
    my_field = None
```

然而，使用该方法只能从父类中声明定义的字段中退出。它不能像```ModelSerializer```生成默认的字段。

#### 动态修改字段

一旦序列化程序初始化完毕，就可以使用```.fields```属性访问序列化程序中设置的字段字典。访问和修改此属性允许您动态修改序列化程序。

直接修改```fields```参数可以让你做一些有趣的事情，比如在运行时改变序列化字段上的参数，而不是在声明序列化程序的时候。

#### 例子

例如，如果您想要设置序列化程序在初始化时应使用哪些字段，您可以创建一个类似于此的序列化程序类：

```python
class DynamicFieldsModelSerializer(serializers.ModelSerializer):
    """
    A ModelSerializer that takes an additional `fields` argument that
    controls which fields should be displayed.
    """

    def __init__(self, *args, **kwargs):
        # Don't pass the 'fields' arg up to the superclass
        fields = kwargs.pop('fields', None)

        # Instantiate the superclass normally
        super(DynamicFieldsModelSerializer, self).__init__(*args, **kwargs)

        if fields is not None:
            # Drop any fields that are not specified in the `fields` argument.
            allowed = set(fields)
            existing = set(self.fields)
            for field_name in existing - allowed:
                self.fields.pop(field_name)
```

这将允许您执行以下操作：

```python
>>> class UserSerializer(DynamicFieldsModelSerializer):
>>>     class Meta:
>>>         model = User
>>>         fields = ('id', 'username', 'email')
>>>
>>> print UserSerializer(user)
{'id': 2, 'username': 'jonwatts', 'email': 'jon@example.com'}
>>>
>>> print UserSerializer(user, fields=('id', 'email'))
{'id': 2, 'email': 'jon@example.com'}
```

---

#### 自定义默认字段

REST 2 提供了修改API。 REST 3不再提供。

<br />
<br />
<br />

---

### 第三方包

下面是可用的第三方包。

#### Django REST marshmallow

官方链接： https://marshmallow-code.github.io/django-rest-marshmallow/

#### Serpy

#### MongoengineModelSerializer

这个包提供了一个```MongoEngineModelSerializer```的序列化器类，支持使用MongoDB作为Django Rest框架的存储层。

链接地址： https://github.com/umutbozkurt/django-rest-framework-mongoengine

#### GeoFeatureModelSerializer

这个包提供了一个```GeoFeatureModelSerializer```序列化器类，支持GeoJSON的读取和写入操作。

链接地址： https://github.com/djangonauts/django-rest-framework-gis

#### HStoreSerializer

#### Dynamic REST

#### Dynamic Fields Mixin

#### DRF FlexFields

#### Serializer Extensions

这个包提供了一系列的工具。允许在每个请求的基础上定义字段。字段可以列入白名单，列入黑名单并且可以扩展子序列化器。

链接地址： https://github.com/evenicoulddoit/django-rest-framework-serializer-extensions

#### HTML JSON Forms

#### DRF-Base64

#### QueryFields

#### DRF Writable Nested

---

参考链接:

- https://docs.djangoproject.com/en/2.0/topics/db/managers/
- https://www.dabapps.com/blog/django-models-and-encapsulation/
- https://marshmallow-code.github.io/django-rest-marshmallow/
- https://github.com/umutbozkurt/django-rest-framework-mongoengine
- https://github.com/evenicoulddoit/django-rest-framework-serializer-extensions
- https://github.com/djangonauts/django-rest-framework-gis