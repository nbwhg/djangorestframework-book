# 分页

Django提供了几个类，可以帮助您管理分页数据 - 也就是说，通过“上一页/下一页”链接分割多个页面的数据。

  -- Django文档
  
REST框架包含对可定制分页样式的支持。这允许您修改多大的结果集被分成单独的数据页面。

分页API可以支持：

作为响应内容的一部分提供的分页链接。
分页链接包含在响应标题中，例如Content-Range或Link。
内置的样式当前都使用作为响应内容的一部分的链接。使用可浏览的API时，此样式更易于访问。

分页仅在您使用通用视图或视图集时自动执行。如果您使用的是常规APIView，则需要自己调用分页API以确保您返回分页响应。查看示例的```mixins.ListModelMixin```和```generics.GenericAPIView```类的源代码。

分页可以通过设置分页类来关闭None。  


## 设置分页样式

分页样式可以使用DEFAULT_PAGINATION_CLASS和PAGE_SIZE设置键全局设置。例如，要使用内置的限制/偏移分页，你可以这样做：
```
REST_FRAMEWORK = {
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.LimitOffsetPagination',
    'PAGE_SIZE': 100
}
```
请注意，您需要设置分页类和应使用的页面大小。双方```DEFAULT_PAGINATION_CLASS```并```PAGE_SIZE```都默认None。

您还可以使用该```pagination_class```属性在单个视图上设置分页类。通常，您希望在整个API中使用相同的分页样式，但您可能希望在每个视图基础上更改分页的各个方面，例如默认或最大页面大小。

## 修改分页样式
如果要修改分页样式的特定方面，则需要重写其中一个分页类，并设置要更改的属性。

```
class LargeResultsSetPagination(PageNumberPagination):
    page_size = 1000
    page_size_query_param = 'page_size'
    max_page_size = 10000

class StandardResultsSetPagination(PageNumberPagination):
    page_size = 100
    page_size_query_param = 'page_size'
    max_page_size = 1000
```    
然后，您可以使用.pagination_class属性将新样式应用于视图：
```
class BillingRecordsView(generics.ListAPIView):
    queryset = Billing.objects.all()
    serializer_class = BillingRecordsSerializer
    pagination_class = LargeResultsSetPagination
```

或者使用```DEFAULT_PAGINATION_CLASS```设置键全局应用样式。例如：

```
REST_FRAMEWORK = {
    'DEFAULT_PAGINATION_CLASS': 'apps.core.pagination.StandardResultsSetPagination'
}
```


------------------

## API参考
### PageNumberPagination

此分页样式在请求查询参数中接受单个号码页码。

要求：

```
GET https://api.example.org/accounts/?page=4
```

回应：

```
HTTP 200 OK
{
    "count": 1023
    "next": "https://api.example.org/accounts/?page=5",
    "previous": "https://api.example.org/accounts/?page=3",
    "results": [
       …
    ]
}
```

#### 建立
要```PageNumberPagination```全局启用样式，请使用以下配置，并PAGE_SIZE根据需要进行设置：

```
REST_FRAMEWORK = {
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.PageNumberPagination',
    'PAGE_SIZE': 100
}
```

在```GenericAPIView```子类上，您还可以将该```pagination_class```属性设置```PageNumberPagination```为按照每个视图进行选择。

组态
所述```PageNumberPagination```类包括多个可重写修改分页样式属性。

要设置这些属性，您应该重写PageNumberPagination该类，然后像上面那样启用您的自定义分页类。

```django_paginator_class``` - 要使用的Django Paginator类。默认值是```django.core.paginator.Paginator```，对大多数用例来说应该没问题。
```page_size``` - 指示页面大小的数值。如果设置，则会覆盖PAGE_SIZE设置。默认值与PAGE_SIZE设置键相同。
```page_query_param``` - 一个字符串值，指示用于分页控件的查询参数的名称。
```page_size_query_param``` - 如果设置，这是一个字符串值，指示查询参数的名称，允许客户端根据每个请求设置页面大小。默认为None，表示客户端可能无法控制所请求的页面大小。
```max_page_size``` - 如果设置，这是一个数字值，表示允许的最大页面大小。该属性仅在```page_size_query_param```设置时才有效。
```last_page_strings```- 字符串值的列表或元组值，指示可用于```page_query_param```请求集合中最终页面的值。默认为('last',)
```template``` - 在可浏览API中呈现分页控件时使用的模板的名称。可能会被覆盖以修改渲染样式，或者设置为None完全禁用HTML分页控件。默认为```"rest_framework/pagination/numbers.html"```。


-------

### LimitOffsetPagination
这种分页样式反映了查找多个数据库记录时使用的语法。客户端包含“限制”和“偏移量”查询参数。该限制表示要返回的项目的最大数量，并且等同page_size于其他样式。偏移量指示查询的起始位置与完整的未分类项目集的关系。

要求：

```
GET https://api.example.org/accounts/?limit=100&offset=400
```

回应：

```
HTTP 200 OK
{
    "count": 1023
    "next": "https://api.example.org/accounts/?limit=100&offset=500",
    "previous": "https://api.example.org/accounts/?limit=100&offset=300",
    "results": [
       …
    ]
}
```

#### 建立
要```LimitOffsetPagination```全局启用样式，请使用以下配置：

```
REST_FRAMEWORK = {
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.LimitOffsetPagination'
}
```
或者，您也可以设置一个PAGE_SIZE密钥。如果该PAGE_SIZE参数也被使用，则limit查询参数将是可选的，并且可以被客户端省略。

在```GenericAPIView```子类上，您还可以将该```pagination_class```属性设置```LimitOffsetPagination```为按照每个视图进行选择。

组态
所述```LimitOffsetPagination```类包括多个可重写修改分页样式属性。

要设置这些属性，您应该重写```LimitOffsetPagination```该类，然后像上面那样启用您的自定义分页类。

```default_limit``` - 一个数字值，指示客户端在查询参数中未提供的限制。默认值与```PAGE_SIZE```设置键相同。
```limit_query_param``` - 一个字符串值，指示“限制”查询参数的名称。默认为'limit'。
```offset_query_param - 一个字符串值，指示“偏移量”查询参数的名称。默认为'offset'。
```max_limit``` - 如果设置，这是一个数字值，表示客户可能要求的最大允许限制。默认为None。
```template``` - 在可浏览API中呈现分页控件时使用的模板的名称。可能会被覆盖以修改渲染样式，或者设置为None完全禁用HTML分页控件。默认为```"rest_framework/pagination/numbers.html"```。


-----


### CursorPagination
基于光标的分页提供了一个不透明的“光标”指示器，客户端可以使用该指示器来遍历结果集。此分页样式仅提供前向和反向控件，并且不允许客户端导航到任意位置。

基于游标的分页需要在结果集中存在唯一的，不变的项目顺序。这种排序通常可能是记录上的创建时间戳，因为这提供了一致的排序。

基于光标的分页比其他方案更复杂。它还要求结果集呈现固定顺序，并且不允许客户端任意索引结果集。但它确实提供了以下好处：

提供一致的分页视图。正确使用```CursorPagination```时，即使在分页过程中，其他客户端正在插入新项目时，客户端也不会在查看记录时看到相同的项目两次。
支持使用非常大的数据集。使用极大数据集分页时，使用基于偏移量的分页样式可能会变得效率低下或无法使用。基于游标的分页方案具有固定时间属性，并且不会随着数据集大小的增加而减慢。
细节和限制
正确使用基于光标的分页需要稍微注意细节。您需要考虑您希望将该方案应用于何种顺序。默认是按顺序排列"-created"。这假设在模型实例上必须有一个“创建的”时间戳字段，并且会呈现一个“时间轴”样式分页视图，其中最近添加的项目是第一个。

您可以通过重写'ordering'分页类上的属性或者使用```OrderingFilter```过滤器类来修改排序```CursorPagination```。您与```OrderingFilter```一起使用时，应强烈考虑限制用户可以自定义的字段。

正确使用游标分页应该有一个满足以下条件的排序字段：

在创建时应该是一个不变的值，例如时间戳，slu，或其他只设置一次的字段。
应该是独特的，或几乎独一无二的。毫秒精度时间戳就是一个很好的例子。这种游标分页的实现使用了一种智能的“位置加偏移”风格，允许它正确地支持非严格唯一的值作为排序。
应该是可以强制为字符串的非空值。
不应该是一个浮动。精度错误很容易导致错误的结果。提示：改用小数。（如果您已经有一个浮点型字段并且必须对其进行分页，则可以在此处找到一个使用小数来限制精度的 示例CursorPagination子类。）
该字段应该有一个数据库索引。
使用不满足这些约束条件的排序字段通常仍然有效，但是您将失去光标分页的一些好处。

有关用于光标分页的实现的更多技术细节，“为Disqus API构建游标”博客文章对基本方法进行了很好的概述。

#### 建立

要```CursorPagination```全局启用样式，请使用以下配置，```PAGE_SIZE```根据需要修改：
```
REST_FRAMEWORK = {
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.CursorPagination',
    'PAGE_SIZE': 100
}
```

在```GenericAPIView```子类上，您还可以将该```pagination_class```属性设置```CursorPagination```为按照每个视图进行选择。


#### 组态
所述```CursorPagination```类包括多个可重写修改分页样式属性。

要设置这些属性，您应该重写```CursorPagination```该类，然后像上面那样启用您的自定义分页类。

```page_size```=表示页面大小的数值。如果设置，则会覆盖PAGE_SIZE设置。默认值与PAGE_SIZE设置键相同。
```cursor_query_param=```一个字符串值，指示“游标”查询参数的名称。默认为```'cursor'```。
```ordering```=这应该是一个字符串或字符串列表，指示将应用基于光标的分页的字段。例如：```ordering = 'slug'```。默认为-created。该值也可以通过OrderingFilter在视图上使用来覆盖。
```template```=在可浏览API中呈现分页控件时使用的模板的名称。可能会被覆盖以修改渲染样式，或者设置为None完全禁用HTML分页控件。默认为```"rest_framework/pagination/previous_and_next.html"```。

----

## 自定义分页样式
要创建自定义分页序列化程序类，您应该继承```pagination.BasePagination```并覆盖```paginate_queryset(self, queryset, request, view=None)和get_paginated_response(self, data)```方法：

该```paginate_queryset```方法传递给初始查询集，并返回一个只包含请求页面中数据的可迭代对象。
该```get_paginated_response```方法传递序列化的页面数据并返回一个Response实例。
请注意，该```paginate_queryset```方法可能会在分页实例上设置状态，稍后可能会使用该```get_paginated_response```方法。

#### 例
假设我们想用一个修改后的格式替换默认的分页输出样式，该样式包含嵌套的“链接”键下的下一个和前一个链接。我们可以像这样指定一个自定义分页类：

```
class CustomPagination(pagination.PageNumberPagination):
    def get_paginated_response(self, data):
        return Response({
            'links': {
                'next': self.get_next_link(),
                'previous': self.get_previous_link()
            },
            'count': self.page.paginator.count,
            'results': data
        })
```
        
然后我们需要在我们的配置中设置自定义类：

```
REST_FRAMEWORK = {
    'DEFAULT_PAGINATION_CLASS': 'my_project.apps.core.pagination.CustomPagination',
    'PAGE_SIZE': 100
}
```

请注意，如果您关心如何在可浏览的API中响应中显示键的顺序，则可以选择OrderedDict在构建分页响应的主体时使用该顺序，但这是可选的。

### 使用您的自定义分页类

要默认使用您的自定义分页类，请使用以下DEFAULT_PAGINATION_CLASS设置：

```
REST_FRAMEWORK = {
    'DEFAULT_PAGINATION_CLASS': 'my_project.apps.core.pagination.LinkHeaderPagination',
    'PAGE_SIZE': 100
}
```
列表端点的API响应现在将包含一个Link标题，而不是将分页链接包含为响应正文的一部分，例如：

### 分页和模式
通过实现一种```get_schema_fields()```方法，您还可以将分页控件提供给REST框架提供的模式自动生成。此方法应具有以下签名：
```
get_schema_fields(self, view)
```
该方法应该返回一个```coreapi.Field```实例列表。

链接标题

自定义分页样式，使用'链接'标题'


---
## HTML分页控件
默认情况下，使用分页类将导致HTML分页控件显示在可浏览的API中。有两种内置显示样式。在```PageNumberPagination```与```LimitOffsetPagination```类显示页码与前一个和下一个列表控件。本```CursorPagination```类显示一个简单的风格，只显示前面和后面的控制。

### 自定义控件
您可以覆盖呈现HTML分页控件的模板。这两种内置式样是：

```
rest_framework/pagination/numbers.html
rest_framework/pagination/previous_and_next.html
```
在全局模板目录中提供具有这些路径的模板将覆盖相关分页类的默认呈现。

或者，您可以通过对现有类进行子类化来完全禁用HTML分页控件，并将其设置template = None为该类的属性。然后您需要配置您的DEFAULT_PAGINATION_CLASS设置键以将您的自定义类用作默认分页样式。

### 低级API
用于确定分页类是否应显示控件的低级API作为display_page_controls分页实例上的属性公开。如果自定义分页类需要显示HTML分页控件，则应True在该paginate_queryset方法中设置自定义分页类。

该```.to_html()```和```.get_html_context()```方法也可以在自定义分页类，以进一步定制控件的呈现方式覆盖。


---

## 第三方软件包
以下第三方包也可用。

### DRF-扩展
该[DRF-extensions](https://chibisov.github.io/drf-extensions/docs/)软件包包含一个```PaginateByMaxMixinmixin```类，允许您的API客户端指定?page_size=max获取允许的最大页面大小。

### DRF-代理分页
该[drf-proxy-pagination](https://github.com/tuffnatty/drf-proxy-pagination)软件包包含一个```ProxyPagination```允许使用查询参数选择分页类的类。

### 链路报头分页
该[django-rest-framework-link-header-pagination](https://github.com/tbeadle/django-rest-framework-link-header-pagination)软件包包含一个LinkHeaderPagination类，该类通过Github的开发人员文档中描述的HTTP Link头提供分页。

