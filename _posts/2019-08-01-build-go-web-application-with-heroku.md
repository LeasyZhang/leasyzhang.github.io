---
layout:     post
title:      "构建Go Web应用程序"
subtitle:   "Go Web Application With Heroku"
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

最近在学习用Go写一个Web项目,这里介绍一下如何使用Go Gin框架开发一个Web程序,并且部署到Heroku.

### 搭建环境

- 操作系统: macOS 10.13.6
- 开发语言: Go 1.12.3 
- 开发工具: VsCode
- 插件: Go (VsCode插件)

#### 安装Go
Go语言官网有一个安装教程: [官网教程](https://golang.org/doc/install), 我自己用brew来安装go, 在终端执行安装命令
```bash
brew install go
```
安装完成之后，查看Go版本号
```Bash
$ go version
```
如果打印出版本信息说明Go已经安装成功.
安装之后查看自己的GoPath
```bash
go env GOPATH
```
Go1.8之后会有一个默认的GOPATH,默认值是～/go,我这里使用默认路径。之后需要把GOPATH/bin添加到PATH里面，这样通过go get和go install安装的二进制程序才能够被直接运行.
关于Go WorkSpace官网也有详细的解释[Go WorkSpace](https://golang.org/doc/code.html#Workspaces).
在GoPath路径下创建文件夹src/hello,创建一个文件hello.go，在hello.go文件中添加代码
``` go
package main

import "fmt"

func main() {
    fmt.Println("Hello World.\n")
}
```
完成之后的目录结构
![Hello World目录结构](https://leasyzhang.github.io/img/in-post/post-go-dev/go-hello-folder-structure.jpg)
编译源文件
```bash
$ cd ~/go/src/hello
$ go build
```
执行完上面的命令之后会生成一个可执行文件hello,执行这个文件
```bash
$ ./hello
```
可以看到Hello World.的信息，说明Go语言已经安装成功了。
### 搭建项目结构
我并不了解标准的Web项目是如何设置目录结构的，作为Java开发，我按照原先的开发习惯使用下面的目录结构:
![项目结构](https://leasyzhang.github.io/img/in-post/post-go-dev/go-web-structure.jpg)
目录结构解释:
- api: controller层，负责接收用户请求和发送响应
- bin: 可执行文件
- config: 配置文件
- database: 数据库访问相关代码
    - dao: DAO层，数据库操作方法
    - scripts: SQL脚本，创建数据库表的SQL语句
- middleware: 中间件层，拦截用户请求，做二次处理
- test: 单元测试
- util: 工具类
- go.mod & go.sum: go依赖管理文件
- main.go: 应用程序入口
- Procfile: heroku配置文件(后面会提到)
其他文件和项目关系不大,这是完整的项目结构。

我没有在Go Workspace下面创建项目，是在自定义的目录下面开发，这里用了[go modules](https://blog.golang.org/using-go-modules)。
- 在github创建一个repo,名字是sample-gin-web-application,clone到本地。
- 在根目录下创建两个文件go.mod&go.sum,在go.mod文件中添加
```go
module sample-gin-web-app
```
这表示当前项目的名称，后续部署的时候会用到
- 在根目录下创建main.go文件，填充代码
``` go
package main

import "fmt"

func main() {
    fmt.Println("Hello World.\n")
}
```
在根目录执行
```bash
go build -o bin/sample-gin-web-app -v .
```
命令来编译，编译完成会在bin目录下生成一个sample-gin-web-app可执行文件,执行
```bash
./bin/sample-gin-web-app 
```
可以看到Hello World.输出。
- 创建图片中的其他目录，项目的框架搭建完成。
### 引入Gin框架

[Gin](https://github.com/gin-gonic/gin)是一个用Go编写的Web框架，可以很方便地用Go开发Web应用程序
> Gin is a web framework written in Go (Golang). It features a martini-like API with much better performance, up to 40 times faster thanks to httprouter.

- 安装gin：在项目的根目录下执行
```bash
go get -u github.com/gin-gonic/gin
```

### 创建一个Controller

这里使用了Gin在github页面的教程[快速教程](https://github.com/gin-gonic/gin#quick-start)
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
可以看到响应信息pong.

接下来将func函数抽取出来,我们要实现一个用户处理的功能：在api目录下创建一个userApi.go文件，用来处理user相关的请求
- 定义一个user模型，在database/dao目录下创建user.go文件
```go
package dao

type User struct {
	ID       int32
	UserName string
	Age      int32
	Gender   string
}
```
- 在api目录下创建userApi.go文件，创建一个FindUserByID方法，这个方法可以根据用户ID查询用户信息，这里先用Mock的数据
```go
package api

import (
	"net/http"
	userDao "sample-gin-web-app/database/dao"

	"github.com/gin-gonic/gin"
)

func FindUserByID(c *gin.Context) {
	var currentUser userDao.User
	currentUser.ID = 1
	currentUser.UserName = "joe.zhang"
	currentUser.Age = 1
	currentUser.Gender = "MALE"

	c.JSON(http.StatusOK, gin.H{
		"message": currentUser,
	})
}
```
修改main.go文件
```go
package main

import (
	//userApi相当于给api的包取别名，可以用userApi.xx的方式来调用方法
	userApi "sample-gin-web-app/api"

	"github.com/gin-gonic/gin"
)

func main() {
	r := gin.Default()
	//响应/user/1之类的请求，然后调用userApi.go文件的FindUserByID方法
	r.GET("/user/:id", userApi.FindUserByID)
	r.Run()
}
```
如果应用程序正在运行，先终止应用程序(Ctrl+C).重新运行一次应用程序
```bash
go run main.go
```
在浏览器访问http://localhost:8080/user/1
可以得到如下的响应
```json
{"message":{"ID":1,"UserName":"joe.zhang","Age":1,"Gender":"MALE"}}
```
到这里我们已经可以处理web请求了，接下来引入数据库处理的功能。
### 访问数据库
数据库操作我用了[gorm](https://github.com/jinzhu/gorm)框架，这是一个用Go语言版本数据库orm框架，支持mysql/postgresql/sqlite等主流数据库。这个里使用postgres数据库来演示。
- 安装gorm库
```bash
go get -u github.com/jinzhu/gorm
```
- 在database目录下创建dbConnection.go文件，添加代码
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
var DBName = testDBName

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
- 修改user对象
gorm可以将对象映射到数据库表，通过migrate功能，定义好对象之后，gorm可以在数据库创建一张对应的数据库表。
在user对象添加一个gorm.Model来继承这个对象，这里面包含了id/创建时间/更新时间等字段，然后创建一个tableName方法，gorm会根据这个方法映射到真实的数据库表。
```go
package dao

import "github.com/jinzhu/gorm"

type User struct {
	gorm.Model
	UserName string
	Age      int32
	Gender   string
}

func (User) TableName() string {
	//user对象和simple_user表绑定
	return "simple_user"
}
```

- 在user.go里面添加一个方法，可以根据ID获取用户信息
```go
...
import (
	db "sample-gin-web-app/database"

	"github.com/jinzhu/gorm"
)
...
func GetUserByID(userID int) (target User) {
	var user User
	db.Conn.Where("id = ?", userID).First(&user)
	return user
}
```
- 在userApi中引用这个方法，修改FindUserByID方法
```go
func FindUserByID(c *gin.Context) {
	var currentUser userDao.User
	userID, _ := strconv.Atoi(c.Param("id"))
	currentUser = userDao.GetUserByID(userID)

	c.JSON(http.StatusOK, gin.H{
		"message": currentUser,
	})
}
```
- 在main方法中连接数据库
这里在应用程序启动方法连接数据库，初始化数据库表
```go
import (
	userApi "sample-gin-web-app/api"
	db "sample-gin-web-app/database"
	"sample-gin-web-app/database/dao"
	"github.com/gin-gonic/gin"
)
...
func main() {
	db, err := db.GetDBConnection()

	if err != nil {
		panic("fatal error, failed to connect to database")
	}
	db.AutoMigrate(&dao.User{})
	defer db.Close()

	r := gin.Default()
	r.GET("/user/:id", userApi.FindUserByID)
	r.Run() // listen and serve on 0.0.0.0:8080
}
```
重启应用程序，然后在数据库里面发现会出现一张simple_user表
在里面手动添加一条数据，然后访问http://localhost:8080/user/1
得到如下响应
```json
{
	"message":
	{
		"ID":1,
		"CreatedAt":"2019-08-29T00:00:00+08:00",
		"UpdatedAt":"2019-08-29T00:00:00+08:00",
		"DeletedAt":null,
		"UserName":"joe",
		"Age":12,
		"Gender":"MALE"
	}
}
```
到这里我们已经完成了
- 搭建Go项目
- 处理网络请求
- 连接到数据库

这三个最重要的功能，在本地执行

```bash
go build -o bin/sample-gin-web-app -v .
```

命令，执行./bin/sample-gin-web-app就可以在本地运行了。接下来我把这个项目部署到heroku上。
### 部署到Heroku
heroku支持github集成，可以部署github仓库的代码，所以我用这种方式部署项目。
[heroku官方教程](https://devcenter.heroku.com/articles/github-integration)是一个很好的参考。我介绍我的部署方法：
- 先把先前的代码推送到github仓库
- 注册一个heroku账号，Heroku免费账号可以免费创建两个App
- 创建一个App，名字是sample-gin-web-application
- 在应用程序项目根目录下创建一个名字是Procfile的文件，这个文件可以被heroku识别，当作应用程序启动的入口,在Procfile里面添加
```javascript
web: bin/sample-gin-web-app
```
- 在heroku的App管理界面，在Deploy-->Deploy method选项中选择github,连接到先前创建的仓库
- 在开始部署之前需要创建一个postgres数据库，heroku提供一个免费小容量的postgre数据库，在Resources菜单下的Add-ons选项中搜索postgres，选中"Heroku Postgres",添加一个免费的postgres,这样数据库就连接到当前项目中。
- 现在返回项目中修改数据库链接地址，让其指向我们创建的数据库，heroku在创建一个数据库之后，会添加一个环境变量"DATABASE_URL"，这个环境变量存储的是数据库连接，我们把数据库连接串改成
```go
os.Getenv("DATABASE_URL")
```
- 在Deploy-->Manual Deploy菜单中选择master分支点击Deploy Branch，开始部署。
- 部署完成之后我们向heroku数据库添加一些数据
- 访问https://sample-gin-web-application.herokuapp.com/user/1，成功得到响应
```json
{
	"message":
	{
		"ID":1,
		"CreatedAt":"2019-08-28T00:00:00Z",
		"UpdatedAt":"2019-08-29T00:00:00Z",
		"DeletedAt":null,
		"UserName":"joe",
		"Age":1,
		"Gender":"MALE"
	}
}
```
> Heroku提供自动部署的功能，在Deploy --> Automatic deploys菜单，点击"Enable Automatic Deploys"按钮，选择master分支，这样子所有的master分支变动都会触发一次自动部署。

现在我们的项目已经成功部署到heroku上面了。
项目代码在这里，[项目地址](https://github.com/LeasyZhang/sample-gin-web-application).