---
title: 【项目笔记】通过JWT实现认证与授权
date: 2025-03-22 21:30:00
categories: 项目笔记
---

## 什么是JWT？

简单来说，JWT是JSON Web Token的缩写，也就是用JSON作为Web应用中的令牌，用于在各方之间安全地将信息作为JSON对象传输。在数据传输过程中还可以完成数据加密、签名等相关处理。

## JWT的优点

1. **无状态**：JWT自身包含了身份验证所需要的所有信息，因此，我们的服务器不需要存储JWT信息。这显然增加了系统的可用性和伸缩性，大大减轻了服务端的压力。可以看出，JWT更符合设计RESTful API时的「Stateless（无状态）」原则 。
2. **避免CSRF攻击**：CSRF（Cross Site Request Forgery） 一般被翻译为跨站请求伪造。而使用JWT进行身份验证不需要依赖Cookie ，因此可以避免CSRF攻击。

## JWT应用举例

![](/images/【项目笔记】通过JWT实现认证与授权/image.png)

## JWT应用进阶

1. 将JWT存放在localStorage中，放在Cookie中会有CSRF的风险。

2. **JWT并不是完美的**，使用JWT实际上会给“**退出登录**”这个功能带来一点小麻烦，即退出登陆后JWT仍然有效。

这个问题不存在于Session认证方式中，因为在Session认证方式中，遇到这种情况的话服务端删除对应的Session记录即可。但是，使用JWT认证的方式就不好解决了，JWT是自验证的，一旦派发出去，如果后端不增加其他逻辑的话，它在失效之前都是有效的。

目前常见的一种打补丁的方式就是，将JWT存入Redis，并利用JWT的有效时间设置Redis中key的过期时间。每次使用JWT都要先从Redis中查询JWT是否存在，JWT过期后会被Redis自动删除，如果需要让某个JWT失效就直接从Redis中删除这个JWT即可。

虽然这种思路违背了JWT的无状态原则，但是一般实际项目中我们还是会使用。

>> Redis（**Re**mote **Di**ctionary **S**erver）是一个开源的、高性能的 **内存数据库**（In-Memory Database），常被用作缓存、消息队列或实时数据存储系统。它支持多种数据结构，以键值对（Key-Value）的形式存储数据，并因其极快的读写速度（微秒级响应）而广受欢迎。

## 参考资料

1. [Introduction to JSON Web Tokens](ttps://jwt.io/introduction)
2. [JWT 基础概念详解](https://javaguide.cn/system-design/security/jwt-intro.html)
3. [JWT 身份认证优缺点分析](https://javaguide.cn/system-design/security/advantages-and-disadvantages-of-jwt.html)
