## 序列化器字段Serializer fields

> 表单类中的每个字段不仅负责验证数据，还负责“cleaning”它们 —— 将它们转换为正确的格式。这是个非常好用的功能，因为它允许字段以多种方式输入数据，并总能得到一致的输出。  
—— [Django 官方文档(Form API)](https://docs.djangoproject.com/en/2.0/ref/forms/api/#django.forms.Form.cleaned_data)

序列化器字段负责处理原始数据和内部数据类型的互相转换。还负责处理验证输入的数据，以及从父级对象中检索和设置值。

---

注意： 这些序列化器字段是定义在```fields.py```文件中的，但是按照惯例，我们应该通过```from rest_framework import serializers```并且通过```serializers.<FieldName>```来使用这些字段。

---

#### 核心参数

每个序列化器字段类的构造至少需要下面的几个参数。一些字段类可能需要额外的，字段特定的参数，但是无论如何都需要下面的几个参数：

**```read_only```**

只读字段被包含在API输出中，但是不应该在创建或者更新期间包含在输入中。任何不正确包含在序列化器输入中的```read_onpy```字段将会被忽略。

设置字段的这个参数为```True```确保在序列化过程中能正确显示，但是在创建或者更新一个实例的反序列化期间不会被使用。

默认是```False```。

**```write_only```**

跟上面的正好相反。设置字段的这个参数为```True```，确保在创建或者更新一个实例的反序列化期间可以使用该字段， 但是在序列化的时候不会包含该字段。

默认是```False```。

**```required```**

如果在反序列化期间没有提供该字段，那么会触发一个异常。如果该字段在反序列化期间不是必须的，那么应该设置这个参数为```False```。

当序列化实例的时候，设置为```False```允许对象的属性或者是字典的key可以被忽略。如果不包含key，那么在输出表示中将不会被包含。

默认是```True```。

**```default```**

如果在输入的值中没有被提供，那么该字段的值将会使用默认值。如果没有设置的话，默认行为是什么都不填充。

在```局部更新PATCH```的期间，不会应用```default```参数。在局部更新期间，只有在传入数据中提供的值才会被验证返回。

可以设置为一个函数或者其他可以调用的任何对象，在这种场景下，每次使用该值的时候都会进行评估。当被调用的时候，它将不会接收任何参数。如果可调用的对象有```set_context```方法，那么在每次获取字段实例的值之前都会调用该方法。这个和```[validators](./validators.md)```的工作方式一样。

当序列化实例的时候，如果实例的对象属性或者字典中的关键字没有被提供的时候，将会使用该值。

注意，设置默认值意味着该字段并不是必须的。如果一个字段同时包含```default```和```required```关键字参数，那么是无效的会抛出异常。

**```allow_null```**

默认情况，当一个```None```传递给序列化字段的时候会触发异常。如果设置该属性为```True```，那么会认为```None```是有效的。

注意，在序列化的时候如果没有明确的指定```default```，如果设置该参数为```True```，那么意味着该字段```default```为```null```， 但是这并不是意味着在反序列化的时候的默认值。

默认为```False```

**```source```**

这个参数用于填充该字段的属性的名称。可能是一个方法，比如```URLField(source='get_absolute_url')```，也可能是一个属性```EmailField(source='user.email')```。如果在属性遍历的期间任何对象不存在，或者为空，那么可能需要提供一个默认值```default```。

注意```source='*'```是由特殊的表示意义的，他表示应该把对象传进来。这对创建嵌套表示或对于需要访问完整对象以确定输出表示的字段非常有用。

默认为该字段的名称。

**```validators```**

是一个验证器函数列表，被用于传进来的字段输入，最终要么触发验证错误的异常，要么只是简单的返回。验证器函数通常应该触发```serializers.ValidationError```异常。

**```error_messages```**

包含错误信息的字典。

**```label```**

一个简短的文本字符串，可用作html表单字段或其他描述性元素中字段的名称。

**```help_text```**

一个文本字符串，可用作html表单字段或其他描述性元素中字段的描述。

**```initial```**

一个应该用于预填充html表单字段值的值。你可以传递一个可调用对象，就像你对任何常规Django字段可以做的一样：

```python
import datetime
from rest_framework import serializers
class ExampleSerializer(serializers.Serializer):
    day = serializers.DateField(initial=datetime.date.today)
```

**```style```**

一个键值对的字典，可用于控制渲染器如何渲染字段。

```python
# Use <input type="password"> for the input.
password = serializers.CharField(
    style={'input_type': 'password'}
)

# Use a radio input instead of a select input.
color_channel = serializers.ChoiceField(
    choices=['red', 'green', 'blue'],
    style={'base_template': 'radio.html'}
)
```

<br />
<br />
<br />
<br />

---

### Boolean字段

#### BooleanField

表示一个布尔值。

注意，会使用```required=True```选项生成默认的```BooleanField```实例。因为Django的```models.BooleanField```总是```blank=True```。如果想改变这个行为，需要明确的指定```BooleanField```在序列化类中。

对应```django.db.models.fields.BooleanField```

#### NullBooleanField

可以接受```None```的布尔字段。

对应```django.db.models.fields.NullBooleanField```

<br />
<br />
<br />
<br />

---

### String 字段

#### CharField

一个文本表示。可选的验证参数```max_length```和```min_length```。

对应```django.db.models.fields.CharField```或者```django.db.models.fields.TextField```

原型：```CharField(max_length=None, min_length=None, allow_blank=False, trim_whitespace=True)```

#### EmailField

一个文本表示。但是默认会验证是否是Email。

对应```django.db.models.fields.EmailField```

#### RegexField

一个文本表示，用于验证给定值是否与某个正则表达式匹配。

对应```django.forms.fields.RegexField```

原型：```RegexField(regex, max_length=None, min_length=None, allow_blank=False)```

#### SlugField

一个```RegexField```，但是他默认只会匹配```[a-zA-Z0-9_-]+```

对应```django.db.models.fields.SlugField```

原型： ```SlugField(max_length=50, min_length=None, allow_blank=False)```

#### URLField

一个```RegexField```，但他只会匹配URL格式。有效的格式是```http://<host>/<path>```。

对应```django.db.models.fields.URLField```使用Django的校验器```django.core.validators.URLValidator```

原型： ```URLField(max_length=200, min_length=None, allow_blank=False)```

#### UUIDField

一个确保输入是有效的uuid字符串的字段。```to_internal_value```方法将会返回一个```uuid.UUID```实例。输出格式如下：

```python
"de305d54-75b4-431b-adb2-eb6b9e546013"
```

原型： ```UUIDField(format='hex_verbose')```

注意：```format```是用来确定UUID的样式：

- ```hex_verbose```: 十六进制，包含连字符。```"5ce0e9a5-5ffa-654b-cee0-1238041fb31a"```
- ```'hex'```: 紧凑的十六进制，不包含连字符。```"5ce0e9a55ffa654bcee01238041fb31a"```
- ```'int'```: 一个128位的uuid整数表示。 ```"123456789012312313134124512351145145114"```
- ```'urn'```: 遵循RFC 4122 URN的UUID。```"urn:uuid:5ce0e9a5-5ffa-654b-cee0-1238041fb31a"```

#### FilePathField

一个字段，其选项仅限于文件系统上某个目录中的文件名。

对应```django.forms.fields.FilePathField```

原型： ```FilePathField(path, match=None, recursive=False, allow_files=True, allow_folders=False, required=None, **kwargs)```

- ```path```: 这个```FilePathField ```应该得到其选择的目录的绝对文件系统路径。例如: ```"/home/images"```
- ```match```: 将会作为一个正则表达式来匹配文件名。但请注意正则表达式将将被作用于基本文件名，而不是完整路径。
- ```recursive```: 是否递归子目录。
- ```allow_files```: 指定是否应该包含指定位置的文件。该参数或```allow_folders``` 中必须有一个为 True
- ```allow_folders```: 声明是否包含指定位置的文件夹。该参数或 allow_files 中必须有一个为 True

#### IPAddressField

一个确保输入是有效的ipv4或ipv6字符串的字段。

对应```django.forms.fields.IPAddressField```或者```django.forms.fields.GenericIPAddressField```

原型： ```IPAddressField(protocol='both', unpack_ipv4=False, **options)```

<br />
<br />
<br />
<br />

---

### Numeric 字段

#### IntegerField

表示一个整数。

对应```django.db.models.fields.IntegerField```或者```django.db.models.fields.SmallIntegerField```或者```django.db.models.fields.PositiveIntegerField```或者```django.db.models.fields.PositiveSmallIntegerField```

原型：```IntegerField(max_value=None, min_value=None)```

#### FloatField

表示一个浮点数。

对应```django.db.models.fields.FloatField```

原型：```FloatField(max_value=None, min_value=None)```

#### DecimalField

表示一个十进制。他是Python的```Decimal```实例。

对应```django.db.models.fields.DecimalField```

原型：```DecimalField(max_digits, decimal_places, coerce_to_string=None, max_value=None, min_value=None)```

最大5位数，包含2位小数:

```python
serializers.DecimalField(max_digits=5, decimal_places=2)
```

小数点后10位的分辨率验证小于10亿的数字：

```python
serializers.DecimalField(max_digits=19, decimal_places=10)
```

<br />
<br />
<br />
<br />

---

### 日期和时间字段

#### DateTimeField

表示日期和时间。

对应```django.db.models.fields.DateTimeField```

原型: ```DateTimeField(format=api_settings.DATETIME_FORMAT, input_formats=None)```

注意，如果在使用```ModelSerializer```或者是```HyperlinkedModelSerializer```的时候，如果你的模型字段有```auto_now=True```或者是```auto_now_add=True```，那么该序列化器字段默认会是```read_only=True```

如果你想要重写这个行为，你需要显式的声明序列化器字段:

```python
class CommentSerializer(serializers.ModelSerializer):
    created = serializers.DateTimeField()

    class Meta:
        model = Comment
```

#### DateField

表示一个日期。

对应```django.db.models.fields.DateField```

原型: ```DateField(format=api_settings.DATE_FORMAT, input_formats=None)```

#### TimeField

表示一个时间。

对应```django.db.models.fields.TimeField```

原型： ```TimeField(format=api_settings.TIME_FORMAT, input_formats=None)```

#### DurationField

表示一个持续时间。

对应```django.db.models.fields.DurationField```

<br />
<br />
<br />
<br />

---

### 选择字段

#### ChoiceField

#### MultipleChoiceField

<br />
<br />
<br />
<br />

---

### 文件上传字段

#### FileField

#### ImageField

<br />
<br />
<br />
<br />

---

### 复杂类型字段

#### ListField

#### DictField

#### HStoreField

#### JSONField

<br />
<br />
<br />
<br />

---

### 其他字段

#### ReadOnlyField

#### HiddenField

#### ModelField

#### SerializerMethodField

<br />
<br />
<br />
<br />

---

### 自定义字段

#### 案例

<br />
<br />
<br />
<br />

---

### 第三方模块

#### DRF Compound Fields

#### DRF Extra Fields

#### djangorestframework-recursive

#### django-rest-framework-gis

#### django-rest-framework-hstore

---

参考链接：

- https://docs.djangoproject.com/en/2.0/ref/forms/api/#django.forms.Form.cleaned_data