# Gin 表单接收学习笔记：PostForm / DefaultPostForm / GetPostForm

> 项目位置：`D:\ALLDocument\Code\go\gin4`
> 技术栈：Go + Gin（v1.12.0）
> 学习目标：掌握 Gin 接收 POST 表单数据的三种方式及其区别

## 1. 项目结构

| 文件 | 作用 |
| --- | --- |
| main.go | 路由注册与表单接收逻辑 |
| login.html | 登录表单页，GET /login 时返回 |
| index.html | 提交成功后展示用户名和密码，POST /login 时返回 |
| go.mod / go.sum | Go module 与依赖信息 |

## 2. 运行流程

1. 启动服务：`go run main.go`
2. 浏览器访问 `http://127.0.0.1:8080/login`，得到登录表单
3. 填写 username / password 并提交，浏览器向 `/login` 发送 POST 请求
4. 后端通过 `GetPostForm` 读取表单值
5. 字段缺失时使用兜底值，最终渲染 `index.html` 显示结果

## 3. 核心代码 main.go

```go
package main

import (
	"github.com/gin-gonic/gin"
)

func main() {
	r := gin.Default()
	r.LoadHTMLFiles("./login.html", "./index.html")

	// 显示登录页
	r.GET("/login", func(c *gin.Context) {
		c.HTML(200, "login.html", nil)
	})

	// 接收登录表单
	r.POST("/login", func(c *gin.Context) {
		// 方式三（当前启用）：GetPostForm
		username, ok := c.GetPostForm("username")
		if !ok {
			username = "没脑子"
		}
		password, ok := c.GetPostForm("password")
		if !ok {
			password = "<PASSWORD>"
		}

		c.HTML(200, "index.html", gin.H{
			"username": username,
			"password": password,
		})
	})

	r.Run(":8080")
}
```

## 4. 三种接收方式详解

### 4.1 PostForm —— 最简单，只拿值

```go
username := c.PostForm("username")
password := c.PostForm("password")
```

- 签名：`func (c *Context) PostForm(key string) string`
- 只返回一个字符串。
- 字段不存在时返回 `""`；字段存在但没填值时也是 `""`。
- **缺点：无法区分“字段缺失”和“字段为空”。**

### 4.2 DefaultPostForm —— 字段缺失时给默认值

```go
username := c.DefaultPostForm("username", "没脑子")
password := c.DefaultPostForm("password", "<PASSWORD>")

// 只有表单里根本没有 xxx 这个字段时，才会触发默认值
password = c.DefaultPostForm("xxx", "<PASSWORD>")
```

- 签名：`func (c *Context) DefaultPostForm(key, defaultValue string) string`
- 当 `key` 不存在时，返回 `defaultValue`。
- ⚠️ **重点坑**：如果输入框存在但用户留空，`PostForm` 返回的是 `""`，此时 `DefaultPostForm` 也会返回 `""`，**不会**触发默认值。
- 也就是说，默认值只在“字段完全缺失”时生效。

### 4.3 GetPostForm —— 最严谨，可判断是否存在

```go
username, ok := c.GetPostForm("username")
if !ok {
	username = "没脑子"
}
```

- 签名：`func (c *Context) GetPostForm(key string) (string, bool)`
- 返回 `(值, 是否存在)`。
- `ok == true` 表示表单里**有这个字段**，即使值是空字符串。
- `ok == false` 表示字段缺失，此时可以自己写兜底逻辑。
- 这是当前 `main.go` 使用的方式，也是日常开发中推荐的做法。

## 5. 三种方式对比

| 方法 | 返回值 | 字段缺失 | 字段存在但为空 | 典型场景 |
| --- | --- | --- | --- | --- |
| `PostForm(key)` | `string` | `""` | `""` | 字段一定存在，不关心空值 |
| `DefaultPostForm(key, d)` | `string` | `d`（默认值） | `""`（注意！） | 只处理字段缺失的情况 |
| `GetPostForm(key)` | `(string, bool)` | `("", false)` | `("", true)` | 需要区分缺失与空值，做兜底 |

## 6. 与 Query 的对应关系

`GetPostForm` 和 `GetQuery` 是同一套设计：

- `Query(key)` → `PostForm(key)`
- `DefaultQuery(key, d)` → `DefaultPostForm(key, d)`
- `GetQuery(key)` → `GetPostForm(key)`

一个读 URL 查询参数，一个读 POST 表单，行为模式完全一致。

## 7. 表单页关键点（login.html / index.html）

login.html：

```html
<form action="/login" method="post" novalidate autocomplete="off">
    <input type="text" name="username" id="username">
    <input type="password" name="password" id="password">
    <input type="submit" value="登录">
</form>
```

- `action="/login"`：提交到后端 POST 路由。
- `method="post"`：POST 提交。
- `name="username"`、`name="password"`：**后端读取的 key 就是这里的 name**，改 name 后端的 key 也要跟着改。

index.html：

```html
<span>{{.username}}</span>
<span>{{.password}}</span>
```

- 后端传入 `gin.H{"username": ..., "password": ...}`，模板用 `{{.字段名}}` 输出。

## 8. 实践建议

1. 写 demo 或确定字段一定存在：`PostForm` 最简洁。
2. 字段可能缺失、且允许空值：用 `DefaultPostForm` 之前先想清楚——空输入不会触发默认值。
3. 需要严谨的“存在性判断 + 兜底值”：优先 `GetPostForm` + `if !ok`。
4. 这三个方法针对 `application/x-www-form-urlencoded` 和 `multipart/form-data` 表单；如果是 JSON 请求体，应改用 `ShouldBindJSON` 等绑定方式。
