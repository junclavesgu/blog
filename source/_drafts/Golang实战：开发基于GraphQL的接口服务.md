Golang以其高效、稳定、简单吸引了大量开发者使用，越来越多公司和云计算平台开始选择Golang作为后端服务开发语言。
Golang对比目前主流的后端开发语言Java具有以下优势：

- 性能好：编译为机器码的静态类型语言，能以更少的资源提供同样量级的访问量，节省开支；
- 上手快：语法更简洁上手快，标准库完善设计优秀，跨平台编译，自带垃圾回收，开发效率高；
- 并发友好：语言层面支持并发，可以充分利用多核，非常适合访问量大的服务。


本文讲以打造一个电影网站的后端服务为例，一步步教你如何用Golang开发GraphQL服务，内容涵盖：

- 流行Golang HTTP框架对比与Echo框架教程；
- 流行Golang GraphQL框架对比与graphql-go框架教程；
- Golang操作PostgresSQL数据库；
- 使用Docker容器部署Golang应用；
- 接入GitHub Actions实现开发流程自动化；

# Echo
echo和其它go http框架对比
中间件
常用中间件

# graphql-go

- graphql协议介绍，对比RESTful优缺点
- 对其它graphql go实现对比
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



  

