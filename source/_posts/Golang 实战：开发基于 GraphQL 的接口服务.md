---
title: Golang 实战：开发基于 GraphQL 的接口服务
date: 2020-08-20T06:58:23Z
url: https://github.com/gwuhaolin/blog/issues/26
tags:

---

本文首发于[IBMDev社区](https://developer.ibm.com/zh/articles/develop-graphql-services-using-golang/)

---

Golang以其高效、稳定、简单吸引了大量开发者使用，越来越多公司和云计算平台开始选择Golang作为后端服务开发语言。
Golang对比目前主流的后端开发语言Java具有以下优势：

- 性能好：编译为机器码的静态类型语言，能以更少的资源提供同样量级的访问量，节省云服务开支；
- 上手快：语法更简洁上手快，标准库完善设计优秀，自带垃圾回收，开发效率高；
- 并发友好：语言层面支持并发，可以充分利用多核，轻松应对高并发服务。


本文将以打造一个电影网站的后端服务为例，一步步教你如何用Golang开发GraphQL服务，内容涵盖：

- 流行Golang HTTP框架对比与Echo框架教程；
- 流行Golang GraphQL框架对比与graphql-go框架教程；
- 使用Docker部署Golang应用；

搭建出的服务整体架构如图：
![服务整体架构](https://user-images.githubusercontent.com/5773264/90727363-c8af0680-e2f5-11ea-8c1c-f303fcead9d8.png)

> 在阅读本文前你需要有一定的Golang基础，你可以[阅读免费电子书](http://go.wuhaolin.cn/)入门。

# HTTP框架
Golang标准库内置的[net/http包](https://golang.org/pkg/net/http/)能快速实现一个HTTP服务器：
```golang
import (
	"fmt"
	"net/http"
)

func main() {
	http.HandleFunc("/hello", func(writer http.ResponseWriter, request *http.Request) {
		fmt.Fprintf(writer, "Hello, World!")
	})
	http.ListenAndServe(":8080", nil) // HTTP服务监听在8080端口
}
```

但其功能太基础，要用在实际项目中还需要自己补充大量常用基础功能，例如：

- 路由参数：从URL `/movie/:id` 中提取id参数；
- 静态资源托管：暴露 `/static` 目录下的所有文件；
- 响应格式：返回HTML、JSON、XML等格式响应，需要设置对应HTTP响应头；
- 日志：记录请求和响应日志；
- CORS：支持跨域请求；
- HTTPS：配置HTTPS证书；

好在Golang社区中已有多款成熟完善的HTTP框架，例如[Gin](https://github.com/gin-gonic/gin)、[Echo](https://echo.labstack.com/)等。
Gin和Echo功能相似，但Echo文档更齐全性能更好，因此本文选择Echo作为HTTP框架，接下来详细介绍Echo的用法。

## Echo教程

Echo封装的简约但不失灵活，只需以下代码就能快速实现一个高性能HTTP服务：
```go
import (
	"net/http"
	
	"github.com/labstack/echo/v4"
)

func main() {
	e := echo.New()
	e.GET("/hello", func(context echo.Context) error {
		return c.String(http.StatusOK, "Hello, World!")
	})
	e.Start(":8080") // HTTP服务监听在8080端口
}
```
要实现需要响应JSON也非常简单：
```go
    // 响应map类型JSON
	e.GET("/map", func(context echo.Context) error {
		return context.JSON(http.StatusOK, map[string]interface{}{"Hello": "World"})
	})

	// 响应数组类型JSON
	e.GET("/array", func(context echo.Context) error {
		return context.JSON(http.StatusOK, []string{"Hello", "World"})
	})

	// 响应结构体类型JSON
	type Hi struct {
		Hello string `json:"Hello"`
	}
	e.GET("/struct", func(context echo.Context) error {
		return context.JSON(http.StatusOK, Hi{
			Hello: "World",
		})
	})
```

### Echo获取请求参数
如果请求中带有参数，Echo能方便的帮你解析出来：
```go
    e.GET("/params/:operationName", func(context echo.Context) error {
        email := c.QueryParam("email") // 从URL params?email=abc 中提取email字段的值
        operationName := c.Param("operationName") // 从URL params/:abc 中提取operationName字段的值
		variables := c.FormValue("variables") // 从POST Form请求的body中提取variables字段的值
	})
```
Echo还提供更强大的Bind功能，能根据请求自动的提取结构化的参数，同时还能校验参数是否合法：
```go
    // 定义参数的结构
    type Params struct {
        Email         string                 `validate:"required,email"` // 改字段必填，并且是email格式
        // 从JSON和Form请求中提取的字段名称是operationName，从URL中提取的字段名称是operation_name
        OperationName string                 `json:"operationName" form:"operationName" query:"operation_name"`
        Variables     map[string]interface{}
    }
    e.GET("/structParams", func(context echo.Context) (err error) {
		params:= Params{}
        // Bind将自动根据请求类型，从URL、Body中提取参数转换为Params struct中定义的结构
        err = context.Bind(&params)
        // 如果校验失败，err将非空表示校验失败信息
        if err != nil {
            retuen
        }
	})
```

### Echo错误处理
在获取响应给客户端的数据时可能会发生异常，这时候需要HTTP服务作出响应，Echo的错误处理设计的很优雅：
```go
    e.GET("/movie", func(context echo.Context) (err error) {
        // 获取电影数据，可能发生错误
        movie, err := getMovie()
        // 如果获取电影失败，直接返回错误
        if err != nil {
            // 客户端将收到HTTP 500响应码，内容为：{"message": "err.Error()对应的字符串"}
            retuen
        }
		return context.JSON(http.StatusOK, movie)
	})
```

如果你不想返回默认的500错误，例如没有权限，可以自定义错误码：
```go
    e.GET("/movie", func(context echo.Context) (err error) {
        movie, err := getMovie()
        if err != nil {
            // 客户端将收到HTTP 401响应码，内容为：{"message": "err.Error()对应的字符串"}
            retuen echo.NewHTTPError(http.StatusUnauthorized, err.Error())
        }
	})
```

如果你不想在出错时响应JSON，例如需要响应HTTP，可以自定义错误渲染逻辑：
```go
e.HTTPErrorHandler = func(err error, context echo.Context) {
    return context.HTML(http.StatusUnauthorized, err.Error())
}
```

### Echo常用中间件
Echo内置了大量实用的中间件，例如：
```go
import (
	"github.com/labstack/echo/middleware"
)

// 采用Gzip压缩响应后能传输更少的字节，如果的HTTP服务没有在Nginx背后建议开启
e.Use(middleware.Gzip())

// 支持接口跨域请求
e.Use(middleware.CORS())

// 记录请求日志 
e.Use(middleware.Logger())
```
# 用Golang实现GraphQL

## graphql协议介绍
GraphQL作为一种全新的api设计思想，把前端所需要的api用类似图数据结构的方式结构清晰地展现出来，让前端很方便的获取所需要的数据。
GraphQL可用来取代目前用的最多的restful规范，相比于restful有如下优势：

- 数据的关联性和结构化更好，接口即文档，节省手动维护接口文档的精力；
- 更健壮的接口：静态类型Schema约束，不仅能校验前端发送的参数是否符合格式，还能自动生成TypeScript和C等语言中的类型定义；
- [易于前端缓存数据](https://graphql.org/learn/caching/)：借助结构化，社区中很多GraphQL客户端内置了改功能；
- 按需选择：前端根据自己的场景选择部分字段返回，节省计算和网络传输；

自从FaceBook2012公布了GraphQL规范后，引起了很多大公司和社区关注，逐渐有公司开始使用GraphQL作为API规范。
在Golang社区中也涌现了多个GraphQL服务端框架，例如：

1. [graphql](https://github.com/graphql-go/graphql)：目前用户最多，但缺点是完全通过Golang代码描述字段、结构、取数据逻辑，代码臃肿
2. [gqlgen](https://github.com/99designs/gqlgen)：需要先定义GraphQL Schema，然后通过工具生成大量模版代码，再用Golang代码填充取数据逻辑
3. [graphql-go](https://github.com/graph-gophers/graphql-go)：需要先定义GraphQL Schema，再加Golang代码描述取数据逻辑，代码简约

本文将选择第三个graphql-go作为GraphQL服务端框架，接下来介绍如何使用它。


### 定义GraphQL Schema
假如我们需要实现一个搜索电影的服务，我们需要先定义接口暴露的Schema
```graphql
schema {
    query: Query
}

type Query {
    search(offset: Int,size: Int,q: String): [Movie!] # 通过关键字搜索电影
}

type Movie {
   	id: Int!
   	title: String!
	casts: [Cast!]! # 一个电影有多个演员
	image: String!
}

type Cast {
	id: Int!
	name: String!
	image: String!
}
```

客户端在调用接口时只需要发送以下请求：
```graphql
{
    search(q:"你好"){
        title
        image
        casts{
            name
        }
    }
}
```

### 定义取值逻辑
实现根query的取值逻辑：
```go
import (
	"net/http"

	"github.com/gwuhaolin/echo_graphql"
	"github.com/labstack/echo"
	"github.com/graph-gophers/graphql-go"
)

// 定义筛选参数结构，对应Schema中定义的search方法的参数
type MovieFilter struct {
	Offset   *int32
	Size     *int32
	Q        *string
}

type QueryResolver struct {
}

// 对应Schema中定义的search方法，如果方法的error不为空，将响应500错误码
func (r *QueryResolver) Search(ctx context.Context, args model.MovieFilter) ([]*MovieResolver, error) {
	ms, e := db.SearchMovies(args)
	return WrapMovies(ms), e
}
```

实现获取电影信息的取值逻辑：
```go
type MovieResolver struct {
	*model.Movie
}

func (r *MovieResolver) ID() int32 {
	return r.Movie.ID
}

func (r *MovieResolver) Title() string {
	return r.Movie.Title
}

func (r *MovieResolver) Image() string {
	return r.Movie.Image
}

func (r *MovieResolver) Casts() ([]*CastResolver, error) {
	cs, err := db.Casts(r.Movie.ID)
    // 把返回的Cast数组包裹为CastResolver数组
	return WrapCasts(cs), err
}

// 把返回的Movie数组包裹为MovieResolver数组
func WrapMovies(movies []*model.Movie) []*MovieResolver {
	msr := make([]*MovieResolver, 0)
	for i := range movies {
		msr = append(msr, &MovieResolver{movies[i]})
	}
	return msr
}
```
演员信息的取值实现逻辑和电影的非常相似就不再复述。

定义的Schema和Golang代码之间有一个很清晰的映射，包括下钻的嵌套字段，如图：
![嵌套字段映射图](https://user-images.githubusercontent.com/5773264/90727534-090e8480-e2f6-11ea-9df8-80e6f69959e9.png)

# 打通Echo和graphql-go
graphql-go暴露了一个Exec函数用于执行GraphQL语句，改函数入参为上下文和请求体返回为获取到的数据，用发如下：
```go
schema := graphql.MustParseSchema(`上面定义的Schema`, QueryResolver{}, graphql.UseFieldResolvers())
data := schema.Exec(context.Request().Context(), params.Query, params.OperationName, params.Variables)
```
其中Exec的入参都可以通过Echo拿到：
```go
// graphql请求体的标准格式
type Params struct {
	Query         string                 `json:"query"`
	OperationName string                 `json:"operationName"`
	Variables     map[string]interface{} `json:"variables"`
}

// 在Echo中注册graphql路由
e.Any("/graphql", func(context echo.Context) (err error) {
	params := Params{}
	err = context.Bind(&params)
	if err != nil {
		return
	}
	data := schema.Exec(context.Request().Context(), params.Query, params.OperationName, params.Variables)
	return context.JSON(http.StatusOK, data)
})
```
以上就开发完了一个基于Golang的GraphQL服务。

# 使用Docker部署GraphQL服务
使用Docker部署服务能抹去大量繁琐易错的手工操作，使用Docker部署的第一步是需要把我们上面开发完的GraphQL服务构建成一个镜像，
为此需要写一个Dockerfile：
```Dockerfile
FROM golang:latest as builder
WORKDIR /app
COPY . .
RUN go mod download
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o main ./http

FROM alpine:latest
COPY --from=builder /app/main .
EXPOSE 80
EXPOSE 443
CMD ["./main"]
```
同时你可以定义一个Github Action来自动构建和发布镜像，新增Action配置文件`.github/workflows/docker.yml`如下：
```yaml
name: release
on: [push]
jobs:
  dy-server:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v1
      - name: Docker dy-server release
        uses: elgohr/Publish-Docker-Github-Action@master
        with:
          name: gwuhaolin/projectname/http-server
          username: gwuhaolin
          password: ${{ github.token }}
          registry: docker.pkg.github.com
          dockerfile: http/Dockerfile
          workdir: ./
```
每次你向Github推送代码后都会自动触发构建生产最新的GraphQL服务镜像，有了镜像你可以直接通过docker运行服务：
```shell script
docker run -d --name http-server -p 80:80 -p 443:443 docker.pkg.github.com/gwuhaolin/projectname/http-server:latest
```

# 总结
自从2009年发布Golang到现在，Golang社交已发展的非常成熟，你可以在开源社区找到几乎所有的现成框架。
使用Golang开发出的Graphql服务不仅能支撑高并发量，编译出的产物也非常小，
由于不依赖虚拟机，搭配上Docker带来的自动化部署给开发者节省成本的同时又带来稳定和便利。

虽然Golang能开发出小巧高效的Graphql服务，但可以看出在实现GraphQL取数逻辑那块有大量繁琐重复的工作，
这归咎于Golang语法太过死板无法给框架开发者发挥空间来实现使用更便利的框架，希望后续Golang2能提供更灵活的语法来优化这些不足。

