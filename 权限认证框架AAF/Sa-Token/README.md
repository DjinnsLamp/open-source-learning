# 权限认证框架sa-token

## 开源项目资料

- [官网首页：http://sa-token.dev33.cn/](http://sa-token.dev33.cn/)

- [在线文档：http://sa-token.dev33.cn/doc/index.html](http://sa-token.dev33.cn/doc/index.html)

## Sa-Token是什么？
sa-token是一个轻量级Java权限认证框架，主要解决: 登录认证、权限认证、Session会话 等一系列权限相关问题

在架构设计上，`sa-token`拒绝引入复杂的概念，以实际业务需求为第一目标，业务上需要什么，sa-token就做什么，例如踢人下线、自动续签、同端互斥登录等常见业务在框架内**均可以一行代码调用实现**，简单粗暴，拒绝复杂！

对于传统Session会话模型的N多难题，例如难以分布式、水平扩展性差，难以兼容前后台分离环境，多会话管理混乱等，
`sa-token`独创了以账号为主的`User-Session`模式，同时又兼容传统以token为主的`Token-Session`模式，两者彼此独立，互不干扰，
让你在进行会话管理时如鱼得水，在`sa-toekn`的强力加持下，权限问题将不再成为业务逻辑的瓶颈！

总的来说，与其它权限认证框架相比，`sa-token`具有以下优势：
1. **更简单的上手步骤** ：可零配置启动框架，能自动化的均已自动化，不让你费脑子
2. **更全面的功能示例** ：目前已集成几十项权限相关特性，涵盖了大部分业务场景的解决方案
3. **更易用的API调用** ：同样的一个功能，可能在别的框架中需要上百行代码，但是在sa-token中统统一行代码调个方法即可解决
4. **更高的扩展性** ：框架中几乎所有组件都提供了对应的扩展接口，90%以上的逻辑都是可以被按需重写的

有了sa-token，你所有的权限认证问题，都不再是问题！

## 涵盖功能
- **登录验证** —— 轻松登录鉴权，并提供五种细分场景值
- **权限验证** —— 适配RBAC模型，不同角色不同授权
- **Session会话** —— 专业的数据缓存中心
- **踢人下线** —— 将违规用户立刻清退下线
- **持久层扩展** —— 可集成redis、MongoDB等专业缓存中间件
- **多账号认证体系** —— 比如一个商城项目的user表和admin表分开鉴权
- **无Cookie模式** —— APP、小程序等前后台分离场景 
- **注解式鉴权** —— 优雅的将鉴权与业务代码分离
- **路由拦截式鉴权** —— 设定全局路由拦截，并排除指定路由
- **模拟他人账号** —— 实时操作任意用户状态数据
- **临时身份切换** —— 将会话身份临时切换为其它账号
- **花式token生成** —— 内置六种token风格，还可自定义token生成策略
- **自动续签** —— 提供两种token过期策略，灵活搭配使用，还可自动续签
- **同端互斥登录** —— 像QQ一样手机电脑同时在线，但是两个手机上互斥登录
- **组件自动注入** —— 零配置与Spring等框架集成
- **会话治理** —— 提供方便灵活的会话查询接口
- **更多功能正在集成中...** —— 如有您有好想法或者建议，欢迎加群交流