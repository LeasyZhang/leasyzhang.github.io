---
layout:     post
title:      "构建Go Web应用程序"
subtitle:   "Go With Heroku"
date:       2019-08-01
author:     "leasy"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - Go
    - Gin
    - Heroku
    - Github
---

> "Go Web Application with Gin"

最近在用Go写一个Web项目，记录一下开发的过程。

### 搭建环境

- 操作系统 macOS 10.13.6
- 安装Go，安装教程参考Go语言[官网教程](https://golang.org/doc/install),我安装的Go语言版本是1.12.3
- 我是用VSCode当作开发工具，在VSCode中安装了Go语言插件，这个插件可以帮助我发现语法问题，还可以自动补全

安装完成之后，在终端用
```Bash
$ go version
```
如果打印出版本信息说明Go安装成功
然后在[Go WorkSpace](https://golang.org/doc/code.html#Workspaces)路径下创建文件夹src/hello,创建一个文件hello.go，然后在hello.go文件中添加代码
```go
package main

import "fmt"

func main() {
    fmt.Println("Hello World.\n")
}
```
然后用Go工具编译文件
```bash
$ cd $HOME/go/src/hello
$ go build
```
执行完上面的命令之后会生成一个hello的可执行文件，执行这个文件可以看到输出语句
```bash
$ ./hello
Hello World.
```
如果看到Hello World.的信息，说明Go语言已经安装成功了。
### 搭建项目结构
我并不了解标准的Web项目是如何设置目录结构的，作为Java开发，我按照原先的开发习惯使用下面的目录结构:
![项目结构]((https://leasyzhang.github.io/img/index-go-web-application.jpg))
目录结构解释:
- api: controller层，负责接收用户请求和发送响应
- bin: 可执行文件
- config: 配置文件
- constant: 常量代码存放位置
- database: 数据库访问相关代码
    - dao: DAO层，数据库操作方法
    - entity: Model层，和数据库表映射的类
    - scripts: SQL脚本，创建数据库表的SQL语句
    - dbConnection.go: 数据库连接管理类
- dto: 自定义对象
- middleware: 中间件层，拦截用户请求，做二次处理
- service: 业务逻辑层
- test: 单元测试
- util: 工具类
- go.mod & go.sum: go依赖管理文件
- main.go: 应用程序入口
- Procfile: heroku配置文件(后面会提到)
其他文件和项目关系不大,这是完整的项目结构，我会讲解如何

我没有在Go Workspace下面写代码，是在自己自定义的目录下面开发，所以用了[go modules](https://blog.golang.org/using-go-modules)。
### 引入Gin框架

[Gin](https://github.com/gin-gonic/gin)是一个用Go编写的Web框架
> Gin is a web framework written in Go (Golang). It features a martini-like API with much better performance, up to 40 times faster thanks to httprouter.

用Gin可以方便地开发一个Web应用程序，
- 安装gin
```bash
go get -u github.com/gin-gonic/gin
```
- 在代码中引入gin
```go
import "github.com/gin-gonic/gin" 
```

### 搭建简单的Controller

在Gin的github页面有一个[快速教程](https://github.com/gin-gonic/gin#quick-start)
在main.go文件添加代码
```go
package main

import "github.com/gin-gonic/gin"

func main() {
	r := gin.Default()
	r.GET("/ping", func(c *gin.Context) {
		c.JSON(200, gin.H{
			"message": "pong",
		})
	})
	r.Run() // listen and serve on 0.0.0.0:8080
}
```
运行代码然后访问http://localhost:8080/ping
```bash
$ go run main.go

# run in another terminal
$ curl localhost:8080/ping
```
在终端中可以可以看到响应信息pong.

不过实际开发的时候我会将func函数抽取出来：
```go
func Hello(c *gin.Context) {
    c.JSON(200, gin.H{
			"message": "pong",
		})
}

r.GET("/ping", Hello)
```
### 访问数据库
数据库访问我用了[gorm](https://github.com/jinzhu/gorm)框架，这是一个用Go语言编写的数据库orm框架，支持mysql/postgresql/sqlite等主流数据库。
在database目录下创建dbConnection.go文件，将QuickStart代码拷贝进去测试
```go
package main

import (
  "github.com/jinzhu/gorm"
  _ "github.com/jinzhu/gorm/dialects/sqlite"
)

type Product struct {
  gorm.Model
  Code string
  Price uint
}

func main() {
  db, err := gorm.Open("sqlite3", "test.db")
  if err != nil {
    panic("failed to connect database")
  }
  defer db.Close()

  // 创建
  db.Create(&Product{Code: "L1212", Price: 1000})

  // 查询
  var product Product
  db.First(&product, 1) // 查询 id是1的product数据
  db.First(&product, "code = ?", "L1212") // 查询 code 是l1212的数据

  // 更新 - 将product对象的price字段更新为2000
  db.Model(&product).Update("Price", 2000)

  // 删除 - 删除对象
  db.Delete(&product)
}
```
go orm会将对象和数据库表映射起来，通过go run dbConnection.go来验证结果。
我用的数据库是Postgres,所以我的数据库连接方式如下:
```go
package database

import (
	"os"

	"github.com/jinzhu/gorm"
	_ "github.com/jinzhu/gorm/dialects/postgres"
)

//数据库连接，获取数据库连接就可以操作数据库表
var Conn *gorm.DB

//数据库连接串，这里屏蔽掉生产的数据库
var testDBName = "host=localhost port=5432 user=joe.zhang dbname=mydb password=19950209 sslmode=disable"
var prodDBName = os.Getenv("DATABASE_URL")
var DBName = prodDBName

var DBEngine = "postgres"

func GetDBConnection() (*gorm.DB, error) {
	db, err := gorm.Open(DBEngine, DBName)
	if err != nil {
		panic("Failed to connect to database " + err.Error())
	}
	db.DB().SetMaxIdleConns(10)
	db.DB().SetMaxOpenConns(20)

	Conn = db
	return db, err
}
```
应用程序在main函数中先调用GetDBConnection方法连接数据库然后通过Conn变量来操作数据库表。
```go
func GetBookByID(bookID int) (book entity.Book, bookError *dto.BookErrorResponse) {
	var bookRsp entity.Book

	errors := db.Conn.Where("id = ?", bookID).First(&bookRsp).GetErrors()

	for _, err := range errors {
		if gorm.IsRecordNotFoundError(err) {
			return bookRsp, &dto.BookErrorResponse{Error: constant.BookNotFound, ErrorCode: constant.BookNotFoundCode}
		}
		return bookRsp, &dto.BookErrorResponse{Error: constant.InternalServerError, ErrorCode: constant.InternalServerErrorCode}
	}

	if bookRsp.ID <= 0 {
		return bookRsp, &dto.BookErrorResponse{Error: constant.BookNotFound, ErrorCode: constant.BookNotFoundCode}
	}

	return bookRsp, nil
}
```
### 部署到Heroku
项目在本地运行通过之后，需要部署到生产环境，我用Heroku来作为生产环境服务器。
因为heroku支持拉取github上面的代码然后部署，所以我就用了这种方式。
[heroku官方教程](https://devcenter.heroku.com/articles/github-integration)是一个很好的参考。在部署之前我在项目根目录下添加了一个Procfile的文件，这个文件可以告知heroku应用程序启动的入口。如果配置如下
```javascript
web : your-application
```
那么heroku会先执行build命令，然后尝试执行下面的命令来启动应用程序。
```bash
./your-application
```

项目代码已经托管在github上，[项目地址](https://github.com/LeasyZhang/reading-club-backend).