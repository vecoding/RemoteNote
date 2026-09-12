# Gin 参数绑定 ShouldBind 学习笔记

> 来源：`D:\ALLDocument\Code\go\gin6`，基于 Gin 框架学习 `ShouldBind` 在 query、JSON、form 三种场景下的使用。

## 一、核心结论

- `c.ShouldBind(&u)` 可以自动根据请求类型完成参数绑定：
  - `GET` 请求：绑定 URL 查询参数（query）。
  - `POST` 请求 + `application/x-www-form-urlencoded`：绑定表单数据（form）。
  - `POST` 请求 + `application/json`：绑定 JSON 请求体。
- 三种场景下调用方式完全一致，都是 `ShouldBind` + 结构体地址。
- 绑定必须传**结构体指针**（`&u`），因为绑定过程要修改结构体字段。
- 结构体字段必须**首字母大写（导出字段）**，否则无法接收数据。

## 二、示例代码

### 1. 结构体定义

```go
type UserInfo struct {
	Username string `json:"user" form:"username"`
	Password string `json:"pwd" form:"password"`
}
```

说明：

- `form:"username"`：用于绑定 query 参数和表单字段，对应 `?username=xxx` 或 `<input name="username">`。
- `json:"user"`：用于绑定 JSON 请求体，对应 `{"user": "xxx"}`。
- JSON 字段名和 form 字段名可以不一样，Gin 会按照 tag 进行映射。

### 2. query 参数绑定（GET /user）

```go
r.GET("/user", func(c *gin.Context) {
	var u UserInfo
	err := c.ShouldBind(&u) // 注意是地址传递
	if err != nil {
		c.JSON(http.StatusBadRequest, gin.H{
			"error": err.Error(),
		})
	} else {
		fmt.Printf("%#v\n", u)
		c.JSON(200, gin.H{
			"message": "ok",
		})
	}
})
```

测试：

```text
GET http://localhost:8080/user?username=zhangsan&password=123456
```

在示例代码中，方式一是逐个使用 `c.Query("username")`、`c.Query("password")`，方式二就是直接用 `ShouldBind` 一次绑定到结构体，更加简洁。

### 3. form 表单绑定（POST /form）

```go
r.POST("/form", func(c *gin.Context) {
	var u UserInfo
	err := c.ShouldBind(&u) // 注意是地址传递
	if err != nil {
		c.JSON(http.StatusBadRequest, gin.H{
			"error": err.Error(),
		})
	} else {
		fmt.Printf("%#v\n", u)
		c.JSON(200, gin.H{
			"message": "ok",
		})
	}
})
```

对应 `index.html` 中的表单：

```html
<form action="/form" method="post">
	用户名:
	<input type="text" name="username">
	密码:
	<input type="password" name="password">
	<input type="submit" value="提交">
</form>
```

表单 `input` 的 `name` 要和结构体的 `form` tag 对应：

- `name="username"` → `form:"username"`
- `name="password"` → `form:"password"`

### 4. JSON 绑定（POST /json）

```go
r.POST("/json", func(c *gin.Context) {
	var u UserInfo
	err := c.ShouldBind(&u) // 注意是地址传递
	if err != nil {
		c.JSON(http.StatusBadRequest, gin.H{
			"error": err.Error(),
		})
	} else {
		fmt.Printf("%#v\n", u)
		c.JSON(200, gin.H{
			"message": "ok",
		})
	}
})
```

请求时需要在 Header 中设置 `Content-Type: application/json`，请求体示例：

```json
{
	"user": "zhangsan",
	"pwd": "123456"
}
```

JSON 的 key 对应结构体的 `json` tag。

## 三、容易踩坑的点

### 1. 必须传地址

```go
err := c.ShouldBind(&u)
```

如果写成 `c.ShouldBind(u)`，Gin 无法把解析出来的值写回结构体，绑定会失败。

### 2. 字段必须导出（首字母大写）

```go
type UserInfo struct {
	username string // 错误：无法接收数据
	Password string // 正确：导出字段
}
```

Gin 的绑定依赖反射来设置字段值，只有导出字段才能被反射写入。

### 3. Content-Type 会影响绑定方式

`ShouldBind` 是“自动选择”绑定器：

- `GET` 没有请求体时，绑定 query 参数。
- `POST` 表单时，绑定 form 数据。
- `POST` JSON 时，绑定 JSON 数据。

如果请求方式或 `Content-Type` 不符合预期，可能出现绑定不到数据的情况。

## 四、与更具体的绑定方法对比

| 方法 | 适用场景 |
| --- | --- |
| `ShouldBind` | 自动根据请求类型选择绑定方式，通用 |
| `ShouldBindQuery` | 明确只绑定 query 参数 |
| `ShouldBindJSON` | 明确只绑定 JSON 请求体 |
| `ShouldBindForm` | 明确只绑定 form 表单数据 |

日常学习中用 `ShouldBind` 最方便；当路由同时有 query、JSON、form 多种来源时，也可以改用更具体的方法避免歧义。

## 五、运行方式

```bash
cd D:\ALLDocument\Code\go\gin6
go run main.go
```

服务默认运行在 `http://localhost:8080`：

- `GET /user`：query 参数绑定
- `GET /index`：展示测试表单
- `POST /form`：form 表单绑定
- `POST /json`：JSON 绑定（可用 Postman 测试）

## 六、总结

`ShouldBind` 是 Gin 中很实用的参数绑定入口，同一个结构体、同一段绑定代码，就能同时处理 query、JSON、form 三种数据来源。关键点只有三个：**结构体字段首字母大写**、**传结构体地址**、**按需设置 `form` / `json` tag**。
