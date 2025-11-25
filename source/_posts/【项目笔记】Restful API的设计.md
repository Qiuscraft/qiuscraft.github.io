---
title: 【项目笔记】Restful API的设计
date: 2025-03-02 21:30:00
categories: 项目笔记
---

**1. 可使用的 HTTP 动词：GET, POST, PUT, DELETE。**

PUT和POST的区别在于：PUT是幂等的, 而POST是非幂等的。

**2. 返回结构采用JSON格式。**

**3. 将业务抽象成资源，在接口URI中使用资源的名词复数形式。**

```
GET: /articles/2/
POST: /articles/
...
```

**4. 规范且一致地使用状态码，发生错误时，在返回结构中展示错误详情。**

```json
{
  "error": "Invalid payoad.",
  "detail": {
    "surname": "This field is required."
  }
}
```

**5. 不要嵌套资源，使用query进行筛选资源**

```
GET: /authors/12/articles/
```

上面的接口不如下面的接口好。

```
GET: /articles/?author_id=12
```

**参考资料**

[RESTful API Design: 13 Best Practices to Make Your Users Happy](https://florimond.dev/en/posts/2018/08/restful-api-design-13-best-practices-to-make-your-users-happy)
