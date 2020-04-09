Golang以其高效、稳定、简单吸引了大量开发者使用，越来越多公司和云计算平台开始选择Golang作为后端服务开发语言。
Golang对比目前主流的后端开发语言Java具有以下优势：

- 性能好：编译为机器码的静态类型语言，能以更少的资源提供同样量级的访问量，节省云服务开支；
- 上手快：语法更简洁上手快，标准库完善设计优秀，自带垃圾回收，开发效率高；
- 并发友好：语言层面支持并发，可以充分利用多核，轻松应对高并发服务。


本文将以打造一个电影网站的后端服务为例，一步步教你如何用Golang开发GraphQL服务，内容涵盖：

- 流行Golang HTTP框架对比与Echo框架教程；
- 流行Golang GraphQL框架对比与graphql-go框架教程；
- Golang操作PostgresSQL数据库；
- 使用Docker部署Golang应用；
- 接入GitHub Actions实现开发流程自动化；

> 在阅读本文前你需要有一定的Golang基础，你可以[阅读免费电子书](http://go.wuhaolin.cn/)入门。

# HTTP框架
虽然Golang标准库内置的[net/http包](https://golang.org/pkg/net/http/)能快速实现一个HTTP服务器，但其功能太基础要用在实际项目中还需要我们补充大量功能。
好在Golang社区中已有多款成熟完善的HTTP框架，比如[Gin](https://github.com/gin-gonic/gin)、[Echo](https://echo.labstack.com/)、[iris](https://github.com/kataras/iris)
echo和其它go http框架对比
中间件
常用中间件

# graphql-go

### graphql协议介绍
graphql作为一种全新的api设计思想，把前端所需要的api用类似图数据结构的方式结构清晰地展现出来，让前端很方便的获取所需要的数据。
graphql可用来取代目前用的最多的restful规范，相比于restful有如下优势：

 https://jerryzou.com/posts/10-questions-about-graphql/
 
- 数据的关联性和结构化更好
- 更易于前端缓存数据
  这个一般像 Relay 和 apollo-client 都替你做好了，如果你想了解它的缓存原理，请移步 [GraphQL Caching](https://graphql.org/learn/caching/)
- 更健壮的接口
    不用再因为在缺乏沟通的情况下修改接口，而为系统埋下不稳定的定时炸弹。一切面向前端的接口都有强类型的 Schema 做保证，且完整类型定义因 introspection 完全对前端可见，一旦前端发送的 query 与 Schema 不符，能快速感知到产生了错误。
- 优点就是后端可以少招几个写接口的🐶，可能会节约成本
- 前后端一起开发，节约工期
- 较少维护api文档，节省精力
- 说了这些，其实单对于前端来说，帮助不算特别大🐶

缺点和难推广的地方

- 后端或者中间层把gql封装相应业务对接数据库是难点，需要高端人力
- 需要前端多少学一点类sql语句，不过大部分场景可以封装好固定的sql语句
- 封装gql不好会产生sql性能问题，三级嵌套联查还有n+1的老问题又会冒出来，需要持续优化
- 前端排除bug需要一定的后端知识，前后端架构多少了解一些

自从facebook2012公布了GraphQL规范后，引起了很多大公司和社区关注，落地了很多规范和框架 ，逐渐有公司开始使用graphql作为API规范。
- 客户端net库
- 服务端实现框架

### 对其它graphql go实现对比
- https://github.com/graphql-go/graphql
- https://github.com/graph-gophers/graphql-go
- https://github.com/99designs/gqlgen
- https://github.com/samsarahq/thunder

- echo_graphql echo和graphql对接
- lfucache 高速缓存
- 定义schema

# go-pg

- 定义model
- 生成Table
- 数据库的增删改
- resolve写法打通DB和

## postgres中文检索与优化
- pgtrgm介绍和各函数用法
- 索引gin btree gist介绍和场景实践
- 全文检索tsvet tsquery与中文分词实践
- lfu缓存DB

# docker集成部署
- docker打包HTTP Server
- docker compose部署
- GitHub actions实战



  

