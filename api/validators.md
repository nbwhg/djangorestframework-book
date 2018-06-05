# 校验器Validators

校验器可以用于在不同类型的字段之间重新使用校验逻辑。

大多数情况下，您在REST框架中处理校验时，只需依赖缺省字段校验，或者在序列化程序或字段类上编写显式校验方法。

但是，有时您会希望将校验逻辑放置到可重用组件中，以便在整个代码库中轻松地重用它。这可以通过使用校验器函数和校验器类来实现。

## 在REST框架中校验
```Django REST```框架序列化器中的校验与```Django ModelForm```类中校验的工作方式有点不同。

使用```ModelForm```，校验部分地在窗体上执行，部分地在模型实例上执行。使用REST框架，校验完全在序列化程序类上执行。有利原因如下：

它引入了一个适当的问题分离，使您的代码行为更加明显。
使用快捷```ModelSerializer```类和使用显式```Serializer```类很容易切换。任何正在使用的校验行为```ModelSerializer```都很容易复制。
打印```repr```一个序列化器实例将显示它到底应用了哪些校验规则。在模型实例上没有额外的隐藏校验行为。
当你使用```ModelSerializer```所有这些都是为你自动处理的。如果您想要改为使用```Serializer```类，那么您需要明确定义校验规则。

###例

作为REST框架如何使用显式校验的示例，我们将采用一个简单的模型类，该类具有唯一性约束的字段。

```python
class CustomerReportRecord(models.Model):
    time_raised = models.DateTimeField(default=timezone.now, editable=False)
    reference = models.CharField(unique=True, max_length=20)
    description = models.TextField()
```

以下```ModelSerializer```是我们可用于创建或更新实例的基本信息```CustomerReportRecord```：

```python
class CustomerReportSerializer(serializers.ModelSerializer):
    class Meta:
        model = CustomerReportRecord    
```

如果我们manage.py shell现在使用我们可以打开Django shell

```python
>>> from project.example.serializers import CustomerReportSerializer
>>> serializer = CustomerReportSerializer()
>>> print(repr(serializer))
CustomerReportSerializer():
    id = IntegerField(label='ID', read_only=True)
    time_raised = DateTimeField(read_only=True)
    reference = CharField(max_length=20, validators=[<UniqueValidator(queryset=CustomerReportRecord.objects.all())>])
    description = CharField(style={'type': 'textarea'})
```

这里有趣的是该reference领域。我们可以看到唯一性约束由序列化程序字段上的校验程序明确执行。

由于这种更明确的风格，REST框架包含一些在核心Django中不可用的校验器类。这些类在下面详述。



## UniqueValidator
这个校验器可以用来强制unique=True模型字段的约束。它需要一个必需的参数和一个可选的```messages```参数：

1. queryset 必需 - 这是查询集应对其执行唯一性。
2. message - 校验失败时应使用的错误消息。
3. lookup - 用于查找具有正在校验的值的现有实例的查找。默认为'exact'。


这个校验器应该应用于序列化程序字段，如下所示：

```python
from rest_framework.validators import UniqueValidator

slug = SlugField(
    max_length=100,
    validators=[UniqueValidator(queryset=BlogPost.objects.all())]
)
```


## UniqueTogetherValidator
这个校验器可以用来强制```unique_together```对模型实例进行约束。它有两个必需的参数和一个可选```messages```参数：

1. queryset 必需 - 这是查询集应对其执行唯一性。
2. fields 必需 - 应该创建唯一集合的字段名称列表或元组。这些必须作为序列化程序类中的字段存在。
3. message - 校验失败时应使用的错误消息。

校验器应该应用于序列化器类，如下所示：

```python
from rest_framework.validators import UniqueTogetherValidator

class ExampleSerializer(serializers.Serializer):
    # ...
    class Meta:
        # ToDo items belong to a parent list, and have an ordering defined
        # by the 'position' field. No two items in a given list may share
        # the same position.
        validators = [
            UniqueTogetherValidator(
                queryset=ToDoItem.objects.all(),
                fields=('list', 'position')
            )
        ]
```

        
**注意：***UniqueTogetherValidation类总是施加一个隐含的约束，即它所应用的所有字段总是按需要处理。有default值的字段是一个例外，因为它们总是提供一个值，即使从用户输入中省略也是如此。*


## UniqueForDateValidator
## UniqueForMonthValidator
## UniqueForYearValidator
这些校验器可用于强制实施模型实例```unique_for_date```，```unique_for_month```并```unique_for_year```约束模型实例。他们采取以下论点：

1. queryset 必需 - 这是查询集应对其执行唯一性。
2. field 必需 - 在给定日期范围内唯一性将被校验的字段名称。这必须作为序列化程序类中的字段存在。
3. date_field required - 将用于确定唯一性约束的日期范围的字段名称。这必须作为序列化程序类中的字段存在。
4. message - 校验失败时应使用的错误消息。

校验器应该应用于序列化器类，如下所示：

```python
from rest_framework.validators import UniqueForYearValidator

class ExampleSerializer(serializers.Serializer):
    # ...
    class Meta:
        # Blog posts should have a slug that is unique for the current year.
        validators = [
            UniqueForYearValidator(
                queryset=BlogPostItem.objects.all(),
                field='slug',
                date_field='published'
            )
        ]
```      
        
序列化程序类中始终需要用于校验的日期字段。您不能简单地依赖模型类default=...，因为用于默认值的值直到校验运行后才会生成。

您可能需要使用几种样式，具体取决于您希望API的行为方式。如果您使用```ModelSerializer```的是REST框架，那么您可能仅仅依靠REST框架为您生成的默认值，但如果您正在使用```Serializer```或者只是想要更明确的控制，请使用下面演示的样式。

#### 与可写日期字段一起使用。
如果您希望日期字段是可写的，唯一值得注意的是您应确保它始终可用于输入数据中，可以通过设置```default```参数或通过设置```required=True```。

```python
published = serializers.DateTimeField(required=True)
```
#### 与只读日期字段一起使用。
如果您希望日期字段可见，但用户无法编辑，请设置```read_only=True```并另外设置```default=...```参数。

```python
published = serializers.DateTimeField(read_only=True, default=timezone.now)
```
该字段对用户不可写，但默认值仍将传递给该用户```validated_data```。

#### 与隐藏的日期字段一起使用。
如果您希望日期字段对用户完全隐藏，请使用```HiddenField```。该字段类型不接受用户输入，而是始终将其默认值返回给```validated_data```序列化程序。

```python
published = serializers.HiddenField(default=timezone.now)
```

**注意**：*这些UniqueFor<Range>Validation类强加一个隐式约束，即它们应用于的字段总是按照需要处理。有default值的字段是一个例外，因为它们总是提供一个值，即使从用户输入中省略也是如此。*


## 高级字段默认值
了在串行化器在多个领域得到应用校验有时可以要求不应该由API客户端被提供一个字段输入，但是可作为输入提供给校验器。

您可能想要用于这种校验的两种模式包括：

* 使用HiddenField。该字段将出现在序列化器输出表示中，validated_data但不会被使用。
* 使用标准字段read_only=True，但也包括default=…参数。该字段将用于串行器输出表示中，但不能由用户直接设置。
* 
REST框架包含一些在这种情况下可能有用的默认值。

#### CurrentUserDefault

可用于表示当前用户的默认类。为了使用它，在实例化序列化程序时，'request'必须作为上下文字典的一部分提供。

```python
owner = serializers.HiddenField(
    default=serializers.CurrentUserDefault()
)
```

#### CreateOnlyDefault
可用于在创建操作期间仅设置默认参数的默认类。在更新期间，该字段被省略。

它采用一个参数，这是在创建操作期间应该使用的默认值或可调用参数。

```python
created_at = serializers.DateTimeField(
    default=serializers.CreateOnlyDefault(timezone.now)
)
```


## 校验器的限制
有一些不明确的情况，您需要明确处理校验，而不是依赖ModelSerializer生成的默认序列化程序类 。

在这些情况下，您可能希望通过为序列化程序Meta.validators属性指定一个空列表来禁用自动生成的校验程序。

### 可选字段
默认情况下，“独一无二”校验会强制所有字段都是 required=True。在某些情况下，您可能希望显式应用 required=False其中一个字段，在这种情况下，校验所需的行为是不明确的。

在这种情况下，您通常需要从序列化程序类中排除校验程序，而是在.validate()方法中或在视图中显式编写任何校验逻辑。

例如：

```python
class BillingRecordSerializer(serializers.ModelSerializer):
    def validate(self, data):
        # Apply custom validation either here, or in the view.

    class Meta:
        fields = ('client', 'date', 'amount')
        extra_kwargs = {'client': {'required': False}}
        validators = []  # Remove a default "unique together" constraint.
```        
        
### 更新嵌套序列化器
将更新应用于现有实例时，唯一性校验程序将从唯一性检查中排除当前实例。当前实例在唯一性检查的上下文中可用，因为它作为序列化程序中的一个属性存在，最初instance=...在实例化序列化程序时已通过使用 。

在嵌套序列化器上进行更新操作的情况下，无法应用此排除，因为该实例不可用。

再一次，你可能想要从序列化类中明确地移除校验器，并将校验约束的代码明确地写入.validate()方法或视图中。

### 调试复杂的案例
如果您不确定ModelSerializer类会产生什么行为，通常是运行一个好主意manage.py shell，并打印序列化程序的一个实例，以便您可以检查它自动为您生成的字段和校验程序。

```python
>>> serializer = MyComplexModelSerializer()
>>> print(serializer)
class MyComplexModelSerializer:
    my_fields = ...
```
    
还要记住，在复杂情况下，明确定义序列化程序类通常会更好，而不是依赖于默认 ModelSerializer行为。这涉及更多的代码，但确保了最终的行为更加透明。



## 编写自定义校验器
您可以使用任何Django现有的校验器，或编写您自己的自定义校验器。

### 基于功能
校验器可以是任何可引发serializers.ValidationError失败的可调用函数。

```python
def even_number(value):
    if value % 2 != 0:
        raise serializers.ValidationError('This field must be an even number.')
```        
### 现场级校验
您可以通过向子类添加.validate_<field_name>方法来指定自定义字段级校验Serializer。这在 Serializer文档中有记录

### 基于类的
要编写一个基于类的校验器，请使用该__call__方法。基于类的校验器很有用，因为它们允许您参数化和重用行为。

```python
class MultipleOf(object):
    def __init__(self, base):
        self.base = base

    def __call__(self, value):
        if value % self.base != 0:
            message = 'This field must be a multiple of %d.' % self.base
            raise serializers.ValidationError(message)
```            
            
### 运用 set_context()
在一些高级的情况下，你可能想要一个校验器被传递到序列化器字段，它被用作额外的上下文。你可以通过set_context在基于类的校验器上声明一个方法来实现。

```python
def set_context(self, serializer_field):
    # Determine if this is an update or a create operation.
    # In `__call__` we can then use that information to modify the validation behavior.
    self.is_update = serializer_field.parent.instance is not None

```
