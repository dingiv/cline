# 大模型应用开发第二弹

## Cline 讲解


## 应用层软件三层架构

View-Service-Model

![](./arch.dio.svg)

View 对其前端/客户端呈现接口
+ 监听客户的请求，解析请求，根据规则派发任务
+ 会话管理，该层会创建一个 session 或者 task 对象，然后传递给 Service 层进行处理

Service 处理业务逻辑
+ 具体的业务逻辑


Model

