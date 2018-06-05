# 权限
验证或识别本身通常不足以获取信息或代码。为此，请求访问的实体必须具有授权。

- Apple开发者文档

--------

与身份验证和限制一起，权限决定是应该授予还是拒绝访问请求。

在任何其他代码被允许继续之前，权限检查总是在视图的开始处运行。权限检查通常会使用```request.user```和```request.auth```属性中的身份验证信息来确定是否允许传入请求。

权限用于授予或拒绝不同类别的用户访问API的不同部分。

最简单的权限方式是允许访问任何经过身份验证的用户，并拒绝任何未经身份验证的用户的访问。这对应```IsAuthenticated```于REST框架中的类。

稍微不太严格的权限将允许完全访问经过身份验证的用户，但允许对未经身份验证的用户进行只读访问。这对应```IsAuthenticatedOrReadOnly```于REST框架中的类。


## 如何确定权限
REST框架中的权限总是被定义为权限类的列表。

在运行视图的主体之前，检查列表中的每个权限。如果有任何权限检查失败的情况```exceptions.PermissionDenied```或```exceptions.NotAuthenticated```将引发异常，并且视图的主体将无法运行。

当权限检查失败时，根据以下规则，将返回“403 Forbidden”或“401 Unauthorized”响应：

1. 该请求已成功通过身份验证，但权限被拒绝。 - 将返回HTTP 403 Forbidden响应。
2. 该请求未成功通过身份验证，并且最高优先级的身份验证类不使用```WWW-Authenticate```标头。 - 将返回HTTP 403 Forbidden响应。
3. 请求未成功验证，并且最高优先级认证类不使用```WWW-Authenticate```头。- HTTP 401未经授权的响应，并WWW-Authenticate返回一个适当的头文件。

## 对象级权限
REST框架权限还支持对象级权限。对象级权限用于确定是否允许用户对特定对象进行操作，该特定对象通常是模型实例。

对象级权限在```.get_object()```调用时由REST框架的通用视图运行。与视图级权限一样，```exceptions.PermissionDenied```如果不允许用户对给定对象进行操作，则会引发异常。

如果您正在编写自己的视图并希望强制执行对象级权限，或者如果您```get_object```在通用视图上重写该方法，则需要```.check_object_permissions(request, obj)```在检索到视图的位置显式调用该方法目的。

这要么提出一个```PermissionDenied```或```NotAuthenticated```异常，或者简单地返回如果该视图有相应的权限。

例如：

```python
def get_object(self):
    obj = get_object_or_404(self.get_queryset(), pk=self.kwargs["pk"])
    self.check_object_permissions(self.request, obj)
    return obj
```
    
### 对象级权限的限制
出于性能原因，通用视图在返回对象列表时不会自动将对象级权限应用于查询集中的每个实例。

通常，当您使用对象级权限时，您还需要适当地过滤查询集，以确保用户只能看到他们被允许查看的实例。 



## 设置权限策略
使用该DEFAULT_PERMISSION_CLASSES设置可以全局设置默认权限策略。例如。

```python
REST_FRAMEWORK = {
    'DEFAULT_PERMISSION_CLASSES': (
        'rest_framework.permissions.IsAuthenticated',
    )
}
```
如果未指定，则此设置默认为允许无限制访问：

```python
'DEFAULT_PERMISSION_CLASSES': (
   'rest_framework.permissions.AllowAny',
)
```
您还可以使用APIView基于类的视图在每个视图或每个视图的基础上设置身份验证策略。

```python
from rest_framework.permissions import IsAuthenticated
from rest_framework.response import Response
from rest_framework.views import APIView

class ExampleView(APIView):
    permission_classes = (IsAuthenticated,)

    def get(self, request, format=None):
        content = {
            'status': 'request was permitted'
        }
        return Response(content)
```        
        
或者，如果您将@api_view装饰器与基于功能的视图一起使用。

```python
from rest_framework.decorators import api_view, permission_classes
from rest_framework.permissions import IsAuthenticated
from rest_framework.response import Response

@api_view(['GET'])
@permission_classes((IsAuthenticated, ))
def example_view(request, format=None):
    content = {
        'status': 'request was permitted'
    }
    return Response(content)
 
```
    
注意：当您通过类属性或装饰器设置新的权限类时，您要通知视图忽略settings.py文件中设置的默认列表。


--------

# API参考
## AllowAny
该AllowAny许可类将允许不受限制的访问，不管请求被认证或未认证的。

此权限并非严格要求，因为您可以通过为权限设置使用空列表或元组来获得相同的结果，但是您可能会发现指定此类是有用的，因为它会使意图变得明确。

## IsAuthenticated
该```IsAuthenticated```许可类将拒绝允许任何未认证用户，并允许许可，否则。

如果您希望您的API只能由注册用户访问，则此权限是适合的。

## IsAdminUser
所述```IsAdminUser```许可类将拒绝许可给任何用户，除非```user.is_staff```是```True```在这种情况下的许可将被允许。

如果您希望您的API只能被部分受信任的管理员访问，则此权限是适合的。

## IsAuthenticatedOrReadOnly
在```IsAuthenticatedOrReadOnly```将允许被授权的用户进行任何请求。只有当请求方法是“安全”方法之一时，才允许未经授权的用户请求; ```GET```，```HEAD```或者```OPTIONS```。

如果您希望您的API允许匿名用户读取权限，并且只允许对已通过身份验证的用户进行写入权限，则此权限是适合的。

## DjangoModelPermissions
此权限类与Django的标准```django.contrib.auth``` 模型权限相关联。此权限只能应用于具有```.queryset```属性设置的视图。只有在用户通过身份验证并分配了相关模型权限的情况下，授权才会被授予。

```POST```请求要求用户拥有add模型的权限。
```PUT```并且```PATCH```请求要求用户```change```获得模型的许可。
```DELETE```请求要求用户拥有```delete```模型的权限。
默认行为也可以被重写以支持自定义模型权限。例如，您可能想要包含请求的view模型权限GET。

要使用自定义模型权限，请覆盖```DjangoModelPermissions```并设置```.perms_map```属性。有关详细信息，请参阅源代码。

使用不包含```queryset```属性的视图。
如果您将此权限与使用重写```get_queryset()```方法的视图一起使用，则该视图上可能没有```queryset```属性。在这种情况下，我们建议还使用标记查询集标记视图，以便此类可以确定所需的权限。例如：

```python
queryset = User.objects.none()  # Required for DjangoModelPermissions
DjangoModelPermissionsOrAnonReadOnly
```

与此类似```DjangoModelPermissions```，但也允许未经身份验证的用户拥有对该API的只读访问权限。

## DjangoObjectPermissions
此权限类绑定到Django的标准对象权限框架中，该框架允许对模型执行每对象权限。为了使用此权限类，您还需要添加支持对象级权限的权限后端，例如```django-guardian```。

同样```DjangoModelPermissions```，此权限只能应用于具有```.queryset```属性或```.get_queryset()```方法的视图。只有在用户通过身份验证并且具有相关的每个对象权限和相关的模型权限后，授权才会被授予。

1. POST请求要求用户拥有add模型实例的权限。
2. PUT并且PATCH请求要求用户拥有change模型实例的权限。
3. DELETE请求要求用户拥有delete模型实例的权限。

请注意，```DjangoObjectPermissions``` 不要求```django-guardian```包，并且应该支持其他对象级后端同样出色。

与```DjangoModelPermissions```您一样，您可以通过覆盖```DjangoObjectPermissions```和设置```.perms_map```属性来使用自定义模型权限。有关详细信息，请参阅源代码。

注意：如果你需要的对象级别```view```的权限```GET```，```HEAD```并```OPTIONS```请求，你要还考虑增加的```DjangoObjectPermissionsFilter```类，以确保该列表只端点返回结果包括用户拥有适当的查看权限的对象。   



--------

# 自定义权限
要实现自定义权限，请覆盖BasePermission并实现以下方法之一或两者：

1. ```.has_permission(self, request, view)```
2. ```.has_object_permission(self, request, view, obj)```
```True```如果请求应该被授予访问权限，则方法应该返回，False否则返回。

如果您需要测试如果请求是读操作或写操作，你应该检查对常量的请求方法```SAFE_METHODS```，这是一个包含一个元组```'GET'```，```'OPTIONS'```和```'HEAD'```。例如：

```python
if request.method in permissions.SAFE_METHODS:
    # Check permissions for read-only request
else:
    # Check permissions for write request
```
    
***注意***：```has_object_permission```只有在视图级```has_permission```检查已通过时才会调用实例级方法。另请注意，为了运行实例级检查，视图代码应该显式调用```.check_object_permissions(request, obj)```。如果您使用的是通用视图，那么默认情况下会为您处理。（基于函数的视图将需要明确检查对象权限，从而提高```PermissionDenied```失败。）

```PermissionDenied```如果测试失败，自定义权限将引发异常。要更改与异常相关的错误消息，请```message```直接在您的自定义权限上实施属性。否则```default_detail```，```PermissionDenied```将使用来自该属性的属性。

```python
from rest_framework import permissions

class CustomerAccessPermission(permissions.BasePermission):
    message = 'Adding customers not allowed.'

    def has_permission(self, request, view):
         ...
```
         
# 例子

以下是一个权限类的示例，该权限类将入局请求的IP地址与黑名单进行比对，并在IP被列入黑名单时拒绝该请求。

```python
from rest_framework import permissions

class BlacklistPermission(permissions.BasePermission):
    """
    Global permission check for blacklisted IPs.
    """

    def has_permission(self, request, view):
        ip_addr = request.META['REMOTE_ADDR']
        blacklisted = Blacklist.objects.filter(ip_addr=ip_addr).exists()
        return not blacklisted
```
        
除了针对所有传入请求运行的全局权限外，还可以创建对象级权限，这些权限仅针对影响特定对象实例的操作运行。例如：

```python
class IsOwnerOrReadOnly(permissions.BasePermission):
    """
    Object-level permission to only allow owners of an object to edit it.
    Assumes the model instance has an `owner` attribute.
    """

    def has_object_permission(self, request, view, obj):
        # Read permissions are allowed to any request,
        # so we'll always allow GET, HEAD or OPTIONS requests.
        if request.method in permissions.SAFE_METHODS:
            return True

        # Instance must have an attribute named `owner`.
        return obj.owner == request.user
```   
     
请注意，通用视图将检查适当的对象级权限，但如果您正在编写自己的自定义视图，则需要确保检查自己的对象级权限检查。```self.check_object_permissions(request, obj)```一旦你拥有对象实例，你可以通过从视图中调用。APIException如果任何对象级权限检查失败，此调用将引发适当的，否则将简单地返回。

另请注意，通用视图将仅检查检索单个模型实例的视图的对象级权限。如果您需要列表视图的对象级过滤，则需要单独过滤查询集。查看过滤文档以获取更多详细信息。

--------

# 第三方软件包
以下第三方包也可用。

## 组成权限
的组成权限包提供了一种简单的方式来定义复杂和多深度（与逻辑运算符）权限对象，使用小的和可重复使用的部件。

## REST条件
在静止状态下包装的另一个扩展简单和方便的方式构建复杂的权限。该扩展允许您将权限与逻辑运算符组合在一起。

## DRY Rest权限
该DRY休息权限包提供定义单个默认和自定义操作不同的权限的能力。该软件包是为具有从应用程序的数据模型中定义的关系派生的权限的应用程序生成的。它还支持通过API的序列化程序返回到客户端应用程序的权限检查。此外，它还支持向默认和自定义列表操作添加权限，以限制每个用户检索的数据。

## Django Rest框架角色
在Django的REST框架角色包使得它更容易在多个类型的用户参数的API。

## Django Rest框架API密钥
在Django的REST框架API密钥包，您可以确保服务器的每个请求需要一个API密钥头。您可以从django管理界面生成一个。

## Django Rest框架角色过滤器
的Django的休息框架中的作用的过滤器包提供在多个类型的角色简单滤波。