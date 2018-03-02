## 基于类的视图

> Django的基于类的视图和老式视图相比是更受欢迎的。
——Reinout van Rees

REST framework提供了一个```APIView```类，是Django的```View```类的子类。

相比于常规的```View```类，REST framework的```APIView```有以下几点不同：

- 
- 
- 
- 

## 基于函数的视图

> 基于类的视图是一个优越的解决方案，这是一个误解。
——Nick Coghlan

REST framework对于基于函数的视图同样也能非常好的工作。REST framework提供了一组简单的装饰器，用来包装你的基于函数的视图；来保证你的视图接收到的请求是```Request```的实例(而不是Django的HttPRequest)，并且返回的响应是```Response```的实例(而不是Django的HttpResponse)，并且允许您配置如何处理请求。

