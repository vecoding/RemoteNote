# Gin 路由与路由分组学习笔记

> 来源：`D:\ALLDocument\Code\go\gin9`
> 学习重点：同一路径不同请求方法（GET/POST）、`r.Any` 处理所有方法、`r.NoRoute` 自定义 404、`r.Group` 路由分组与嵌套分组
> 环境：Go 1.26.3、gin v1.12.0

---

## 一、项目结构

| 文件 | 作用 |
| --- | --- |
| `main.go` | 全部路由注册与请求处理逻辑 |
| `go.mod` / `go.sum` | Go module 与依赖信息（gin v1.12.0） |

本项目只有一个 `main.go`，内容围绕"路由怎么注册、方法怎么区分、路由多了怎么组织"展开，非常适合入门路由与分组。

---

## 二、原项目代码

`main.go` 原文：

```go
package main

import (
	"net/http"

	"github.com/gin-gonic/gin"
)

func main() {
	r := gin.Default()
	r.GET("/index", func(c *gin.Context) {
		c.JSON(200, gin.H{
			"message": "GET",
		})
	})
	r.POST("/index", func(c *gin.Context) {
		c.JSON(200, gin.H{
			"message": "POST",
		})
	})
	//Any方法可以处理所有请求方法 通过switch 来判断请求方法
	r.Any("/user", func(c *gin.Context) {
		switch c.Request.Method {
		case http.MethodGet:
			c.JSON(200, gin.H{
				"message": "GET",
			})
		case http.MethodPost:
			c.JSON(200, gin.H{
				"message": "POST",
			})
		}
	})
	//访问不存在的路由处理
	r.NoRoute(func(c *gin.Context) {
		c.JSON(404, gin.H{
			"message": "404",
		})
	})
	r.GET("/video/index", func(c *gin.Context) {
		c.JSON(200, gin.H{
			"message": "/video/index GET",
		})
	})
	r.GET("/video/xx", func(c *gin.Context) {
		c.JSON(200, gin.H{
			"message": "/video/xx GET",
		})
	})
	r.GET("/video/oo", func(c *gin.Context) {
		c.JSON(200, gin.H{
			"message": "/video/oo GET",
		})
	})
	//路由组使用方法
	ShopGroup := r.Group("/shop")
	{
		ShopGroup.GET("/index", func(c *gin.Context) {
			c.JSON(200, gin.H{
				"message": "/shop/index GET",
			})
		})
		ShopGroup.GET("/xx", func(c *gin.Context) {
			c.JSON(200, gin.H{
				"message": "/shop/xx GET",
			})
		})
		ShopGroup.GET("/oo", func(c *gin.Context) {
			c.JSON(200, gin.H{
				"message": "/shop/oo GET",
			})
		})
		//同样的，路由支持嵌套
		pay := ShopGroup.Group("/Pay")
		pay.GET("/WeChat", func(c *gin.Context) {
			c.JSON(200, gin.H{
				"message": "WeChat Pay",
			})
		})
		pay.GET("/AliPay", func(c *gin.Context) {
			c.JSON(200, gin.H{
				"message": "Ali Pay",
			})
		})
	}
	r.Run()
}
```

这段代码可以正常运行，`r.Run()` 默认监听 `:8080`。

---

## 三、路由一览表

把项目里所有"方法 + 路径"整理出来：

| 方法 | 路径 | 响应 | 说明 |
| --- | --- | --- | --- |
| GET | `/index` | `{"message":"GET"}` | 独立注册 |
| POST | `/index` | `{"message":"POST"}` | 独立注册，与 GET 共存 |
| 任意 | `/user` | GET/POST 有响应，其他方法 200 空响应 | `r.Any` 注册 |
| 任意 | 未匹配路径 | 404 `{"message":"404"}` | `r.NoRoute` 兜底 |
| GET | `/video/index` `/video/xx` `/video/oo` | 各自返回路径信息 | 不分组，重复写前缀 |
| GET | `/shop/index` `/shop/xx` `/shop/oo` | 各自返回路径信息 | 用 `r.Group` 分组 |
| GET | `/shop/Pay/WeChat` `/shop/Pay/AliPay` | `{"message":"WeChat Pay"}` / `{"message":"Ali Pay"}` | 嵌套分组：`/shop` 组下的 `/Pay` 组 |

可以看到 `/video/*` 和 `/shop/*` 效果完全一样，区别只在于**写法**：一个把前缀重复写三次，一个用分组把前缀抽出来。这正是路由分组要解决的问题；而 `/shop/Pay/*` 则演示了分组还能继续嵌套，前缀一层层叠加。

---

## 四、知识点 1：同一路径，不同请求方法

在 Gin 中，一条"路由"由**请求方法 + 路径**共同决定。`/index` 这个路径同时注册 GET 和 POST，是两个互不干扰的路由：

```go
r.GET("/index", func(c *gin.Context) {
	c.JSON(200, gin.H{"message": "GET"})
})
r.POST("/index", func(c *gin.Context) {
	c.JSON(200, gin.H{"message": "POST"})
})
```

实测结果（gin v1.12.0）：

| 请求 | 状态码 | 响应体 |
| --- | --- | --- |
| `GET /index` | 200 | `{"message":"GET"}` |
| `POST /index` | 200 | `{"message":"POST"}` |
| `PUT /index` | 404 | `{"message":"404"}`（走 NoRoute） |

要点：

- 方法不同、路径相同，不算冲突，可以各自注册各自的处理器。
- 常见的注册方法还有 `r.PUT`、`r.DELETE`、`r.PATCH`、`r.HEAD`、`r.OPTIONS`。
- 如果只注册了 `r.GET("/index")`，用 POST 访问会走 404，而不是"405 方法不允许"。

---

## 五、知识点 2：`r.Any` 处理所有请求方法

```go
r.Any("/user", func(c *gin.Context) {
	switch c.Request.Method {
	case http.MethodGet:
		c.JSON(200, gin.H{"message": "GET"})
	case http.MethodPost:
		c.JSON(200, gin.H{"message": "POST"})
	}
})
```

`r.Any` 会把 GET、POST、PUT、DELETE、PATCH、HEAD、OPTIONS 等**所有请求方法**都注册到同一个处理器。处理器内部用 `c.Request.Method` 拿到实际方法，再 `switch` 分发。

实测结果：

| 请求 | 状态码 | 响应体 |
| --- | --- | --- |
| `GET /user` | 200 | `{"message":"GET"}` |
| `POST /user` | 200 | `{"message":"POST"}` |
| `PUT /user` | 200 | 空响应（`Content-Length: 0`） |
| `DELETE /user` | 200 | 空响应（`Content-Length: 0`） |

⚠️ 本项目最大的"隐藏坑"：`switch` 里只处理了 GET 和 POST，**没有 `default` 分支**。所以 PUT、DELETE 等请求也能进入处理器（状态码 200），但什么都不写、返回空响应。从调用方角度看，这是一个"成功但没内容"的接口，容易让前端困惑。

建议改成显式兜底：

```go
r.Any("/user", func(c *gin.Context) {
	switch c.Request.Method {
	case http.MethodGet:
		c.JSON(200, gin.H{"message": "GET"})
	case http.MethodPost:
		c.JSON(200, gin.H{"message": "POST"})
	default:
		c.JSON(405, gin.H{"message": "Method Not Allowed"})
	}
})
```

如果只是想对同一个路径分别处理 GET 和 POST，其实直接注册两个路由更清晰，`r.Any` 更适合"所有方法都要走统一逻辑（如登录校验）"的场景。

---

## 六、知识点 3：`r.NoRoute` 自定义 404

```go
r.NoRoute(func(c *gin.Context) {
	c.JSON(404, gin.H{
		"message": "404",
	})
})
```

任何**没有匹配到**的方法 + 路径组合，都会统一进入 `r.NoRoute` 注册的处理器。实测：

| 请求 | 状态码 | 响应体 |
| --- | --- | --- |
| `GET /notexist` | 404 | `{"message":"404"}` |
| `PUT /index` | 404 | `{"message":"404"}` |
| `POST /shop/index` | 404 | `{"message":"404"}`（分组里没注册 POST） |

要点：

- 不注册 `r.NoRoute` 时，Gin 返回默认的纯文本 `404 page not found`；注册后可以统一返回 JSON，方便前端解析。
- `r.NoRoute` 对**任意方法**生效，不只是 GET。
- 适合做统一错误响应格式，例如返回 `{"code": 404, "message": "not found"}`。

---

## 七、知识点 4：路由分组 `r.Group`

### 7.1 分组前：重复写前缀

```go
r.GET("/video/index", ...)
r.GET("/video/xx", ...)
r.GET("/video/oo", ...)
```

### 7.2 分组后：前缀只写一次

```go
ShopGroup := r.Group("/shop")
{
	ShopGroup.GET("/index", ...)
	ShopGroup.GET("/xx", ...)
	ShopGroup.GET("/oo", ...)
}
```

`r.Group("/shop")` 返回一个路由组，之后在组上注册的路由都会自动加上 `/shop` 前缀：

| 分组内注册 | 实际路由 |
| --- | --- |
| `ShopGroup.GET("/index")` | `GET /shop/index` |
| `ShopGroup.GET("/xx")` | `GET /shop/xx` |
| `ShopGroup.GET("/oo")` | `GET /shop/oo` |

实测：`GET /shop/index` → 200 `{"message":"/shop/index GET"}`，与直接注册 `r.GET("/shop/index")` 完全等价。

### 7.3 关于外层 `{}` 大括号

```go
ShopGroup := r.Group("/shop")
{
	...
}
```

这个 `{}` **不是 Gin 的语法**，Go 里 `{}` 只是一个代码块（作用域），用来把同一组的注册代码"视觉上"框在一起，让结构更清晰。去掉它、把三行 `ShopGroup.GET(...)` 平铺写，效果完全一样。真正起作用的是 `ShopGroup := r.Group("/shop")` 这一句。

### 7.4 分组的好处

| 对比项 | 不分组（`/video`） | 分组（`/shop`） |
| --- | --- | --- |
| 前缀写几次 | 每个路由写一次 | 只写一次 |
| 增加新路由 | 复制粘贴前缀，容易写错 | 在组里加一行 |
| 加中间件 | 每个路由单独加 | 整组统一加 |
| 模块边界 | 不明显 | 清晰 |

---

## 八、分组进阶：嵌套、任意方法、中间件

### 8.1 分组内可以注册任意方法

```go
ShopGroup := r.Group("/shop")
{
	ShopGroup.GET("/index", ...)
	ShopGroup.POST("/order", ...)   // POST /shop/order
	ShopGroup.Any("/hello", ...)    // 所有方法 /shop/hello
}
```

分组只是加前缀，注册方法的选择不受影响。

### 8.2 嵌套分组（多级前缀）

项目里已经演示了嵌套分组：`pay` 组挂在 `ShopGroup` 之下，路径前缀一层层叠加：

```go
ShopGroup := r.Group("/shop")
{
	// 子分组：在 /shop 下再挂一层 /Pay
	pay := ShopGroup.Group("/Pay")
	pay.GET("/WeChat", ...)   // GET /shop/Pay/WeChat
	pay.GET("/AliPay", ...)   // GET /shop/Pay/AliPay
}
```

| 分组内注册 | 实际路由 | 响应 |
| --- | --- | --- |
| `pay.GET("/WeChat")` | `GET /shop/Pay/WeChat` | `{"message":"WeChat Pay"}` |
| `pay.GET("/AliPay")` | `GET /shop/Pay/AliPay` | `{"message":"Ali Pay"}` |

实测：`GET /shop/Pay/WeChat` → 200 `{"message":"WeChat Pay"}`；⚠️ 路径**区分大小写**，`GET /shop/pay/wechat` → 404。

嵌套分组最常见的用途是 API 版本管理：

```go
api := r.Group("/api")
{
	v1 := api.Group("/v1")
	{
		v1.GET("/users", ...)   // GET /api/v1/users
	}
	v2 := api.Group("/v2")
	{
		v2.GET("/users", ...)   // GET /api/v2/users
	}
}
```

外层 `/api`、内层按版本分，与项目里 `/shop` + `/Pay` 是同一套机制：**子分组的完整前缀 = 父分组前缀 + 子分组路径**。

### 8.3 分组中间件

```go
// 方式一：创建分组时挂中间件
ShopGroup := r.Group("/shop", AuthMiddleware())

// 方式二：创建后挂中间件
ShopGroup.Use(AuthMiddleware())
```

挂上后，组内**所有路由**都会先执行中间件，适合登录校验、日志、限流等统一逻辑。这是分组比"手写前缀"最大的优势。

---

## 九、运行与测试

```text
cd D:\ALLDocument\Code\go\gin9
go run .
```

启动后监听 `http://127.0.0.1:8080`。想改端口，把 `r.Run()` 改成 `r.Run(":9090")` 即可。

curl 测试命令：

```bash
curl http://127.0.0.1:8080/index
curl -X POST http://127.0.0.1:8080/index
curl http://127.0.0.1:8080/user
curl -X POST http://127.0.0.1:8080/user
curl -X PUT http://127.0.0.1:8080/user
curl http://127.0.0.1:8080/notexist
curl http://127.0.0.1:8080/video/index
curl http://127.0.0.1:8080/shop/index
curl -X POST http://127.0.0.1:8080/shop/index
curl http://127.0.0.1:8080/shop/Pay/WeChat
curl http://127.0.0.1:8080/shop/Pay/AliPay
curl http://127.0.0.1:8080/shop/pay/wechat   # 小写，验证大小写敏感
```

实测结果汇总：

| 请求 | 状态码 | 响应体 |
| --- | --- | --- |
| `GET /index` | 200 | `{"message":"GET"}` |
| `POST /index` | 200 | `{"message":"POST"}` |
| `GET /user` | 200 | `{"message":"GET"}` |
| `POST /user` | 200 | `{"message":"POST"}` |
| `PUT /user` | 200 | 空响应 |
| `GET /notexist` | 404 | `{"message":"404"}` |
| `GET /video/index` | 200 | `{"message":"/video/index GET"}` |
| `GET /shop/index` | 200 | `{"message":"/shop/index GET"}` |
| `POST /shop/index` | 404 | `{"message":"404"}` |
| `GET /shop/Pay/WeChat` | 200 | `{"message":"WeChat Pay"}` |
| `GET /shop/Pay/AliPay` | 200 | `{"message":"Ali Pay"}` |
| `GET /shop/pay/wechat` | 404 | `{"message":"404"}` |

---

## 十、注意事项

1. **路由 = 方法 + 路径**：同路径不同方法互不影响；方法不匹配时走 404（`NoRoute`），不会自动变成 405。
2. `r.Any` 注册所有方法，处理器里要用 `c.Request.Method` 分发，并且**记得写 `default`**，否则未处理的方法会返回 200 空响应，接口行为很迷惑。
3. `r.NoRoute` 对任意方法生效，注册后未匹配请求统一返回自定义 404，适合统一 JSON 错误格式。
4. `r.Group("/shop")` 只是给路由加前缀，与 `r.GET("/shop/index")` 完全等价，不改变路由匹配规则。
5. 分组代码外面那层 `{}` 只是 Go 的代码块，用于视觉分组，**不是语法必需**。
6. 组内注册用 `GroupName.GET/POST/Any`，想注册什么方法就用什么方法。
7. 前缀以 `/` 开头更直观；Gin 拼接路径时会自动处理分隔符，`g.GET("index")` 不写斜杠也能得到 `/shop/index`。
8. 分组适合做中间件和嵌套：权限校验、日志、API 版本（`/api/v1`、`/api/v2`）都用分组实现；本项目 `/shop` 下的 `/Pay` 就是嵌套分组实例。
9. 嵌套分组会逐层拼接前缀：`ShopGroup.Group("/Pay")` + `pay.GET("/WeChat")` = `/shop/Pay/WeChat`。路径**区分大小写**，`/shop/pay/wechat` 会 404。
10. `r.Run()` 默认端口 8080，需要改端口时传参：`r.Run(":9090")`。

---

## 十一、练习

1. 在 `/shop` 分组里新增 `ShopGroup.POST("/index")`，用 `curl -X POST http://127.0.0.1:8080/shop/index` 验证不再 404。
2. 把 `/user` 的 `switch` 补上 `default` 分支，返回 405 JSON，再用 PUT 请求验证。
3. 在 `/shop/Pay` 分组下新增 `pay.GET("/Coupon")`，请求 `/shop/Pay/Coupon`；再试试小写 `/shop/pay/coupon`，观察大小写行为。
4. 给 `/shop` 分组挂一个简单中间件（打印日志），观察三个 `/shop` 路由是否都先执行中间件。
5. 把 `r.Run()` 改成 `r.Run(":9090")`，用新端口访问验证。

---

## 小结

本项目用最少的代码展示了 Gin 路由的核心骨架：**同路径多方法**（GET/POST 共存）、**`r.Any` 通吃所有方法**、**`r.NoRoute` 统一 404**、**`r.Group` 组织同前缀路由**。`/video` 和 `/shop` 两组路由效果相同，却展示了"不分组"和"分组"两种写法；`/shop/Pay` 则进一步演示了**分组嵌套**，前缀逐层叠加出 `/shop/Pay/WeChat`。对比着看就能理解：分组只是把公共前缀抽出来，真正拉开差距的是后续的中间件与嵌套能力。
