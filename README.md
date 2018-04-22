## 介绍

---

注意: 本文档针对的是REST framework的版本3。

编写本文档时，所用的版本号：
- Django(1.11.10)
- djangorestframework(3.7.7)
- Python(3.6.4)

---

![](./images/logo.png)


Django REST framework 是一个功能强大的灵活的构建Web APIs的工具包。

为什么要使用REST framework ?
- 基于Web 浏览器的API 可视化，对于你的开发将会有很大的帮助
- 身份认证策略包含```OAuth1a```和```OAuth2```
- 同时支持ORM和非ORM的数据源的序列化
- 完整的REST API 功能支持, 包括认证、权限、限流、分页等
- 可定制化 - 如果不需要功能强大的特性，那么可以基于基础的功能类(```function-based```)进行开发
- 文档完善，社区活跃
- Mozilla, Red Hat 等公司正在使用REST framework

---

#### 什么人适合本文档？

阅读本文档之前，至少要对Django有一定的了解。

---

#### 项目源码

项目源码存放于Github上，[https://github.com/bigtree6688/djangorestframework-book](https://github.com/bigtree6688/djangorestframework-book)。

---

#### 在线阅读

可以通过[GitBook]()或者[Github](https://github.com/bigtree6688/djangorestframework-book)来在线阅读。

---

## 目录

- 安装
  1. [介绍](./README.md)
  2. [安装](./home/install.md)
  3. [举例](./home/example.md)
- 快速上手
  1. [创建项目](./quickstart/project.md)
  2. [序列化](./quickstart/serializers.md)
  3. [视图](./quickstart/views.md)
  4. [路由](./quickstart/urls.md)
  5. [Settings](./quickstart/settings.md)
  6. [测试API](./quickstart/testing.md)
- 完整教程
  1. [序列化](./tutorial/serialization.md)
  2. [Requests & Responses](./tutorial/req-resp.md)
  3. [类视图](./tutorial/classview.md)
  4. [认证和权限](./tutorial/auth-perms.md)
  5. [关系和超链接](./tutorial/hyperlink.md)
  6. [视图集合和路由](./tutorial/routers.md)
  7. [coreapi](./tutorial/coreapi.md)
- API指南
  1. [Requests](./api/requests.md)
  2. [Responses](./api/responses.md)
  3. [视图Views](./api/views.md)
  4. [通用视图Generic views](./api/gviews.md)
  5. [视图组Viewsets](./api/viewsets.md)
  6. [路由器Routers](./api/routers.md)
  7. [解析器Parsers](./api/parsers.md)
  8. [渲染器Renderers](./api/renderers.md)
  9. [序列化器Serializers](./api/serializers.md)
  10. [序列化器字段Serializer fields](./api/serializersfield.md)
  11. [Serializer relations](./api/serializersrelat.md)
  12. [校验器Validators](./api/validators.md)
  13. [身份验证](./api/authentication.md)
  14. [权限](./api/permissions.md)
  15. [限流Throttling](./api/throttling.md)
  16. [过滤](./api/filtering.md)
  17. [分页](./api/pagination.md)
  18. [API 版本化](./api/versioning.md)
  19. [内容协商Content negotiation](./api/cnegotiation.md)
  20. [元数据](./api/metadata.md)
  21. [Schemas](./api/schemas.md)
  22. [Format suffixes](./api/formatsuffixes.md)
  23. [Returning URLs](./api/urls.md)
  24. [异常处理](./api/exceptions.md)
  25. [状态码](./api/statuscodes.md)
  26. [测试](./api/testing.md)
  27. [Settings](./api/settings.md)
- 最佳实践
  1. [API文档生成](./topics/docs.md)
  2. [HTML & Forms](../topics/forms.md)
  3. [AJAX & CSRF & CORS](topics/cors.md)