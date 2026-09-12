# Gin 路径参数学习笔记

> 来源：`D:\ALLDocument\Code\go\gin5`
> 学习重点：用 `c.Param` 获取 URL 路径中的参数，理解 Gin 路由匹配的"静态优先"规则
> 环境：Go 1.26.3、gin v1.12.0

---

## 一、什么是路径参数

URL 传参有两种常见方式：

| 方式 | 示例 | 获取方法 |
| --- | --- | --- |
| 查询参数 | `/user?name=tom&age=25` | `c.Query("name")` |
| 路径参数 | `/tom/25` | `c.Param("username")` |

路径参数就是把值直接"嵌"在路径里，路由中用 `:参数名` 占位：

- `/:username/:age` 可以匹配 `/tom/25`，其中 `username=tom`、`age=25`
- `/blog/:username` 可以匹配 `/blog/tom`，其中 `username=tom`

获取方式固定一句：

```go
value := c.Param("参数名")
```

---

## 二、原项目代码

`main.go` 原文：

```go
package main

import (
	"github.com/gin-gonic/gin"
)

func main() {
	r := gin.Default()
	r.GET("/:username/:age", func(c *gin.Context) {
		//获取路径参数
		name := c.Param("username")
		age := c.Param("age")
		c.JSON(200, gin.H{
			"name": name,
			"age":  age,
		})
	})
	//新版本为静态地址优先，当:username = blog时，会匹配到blog，而不是username
	r.GET("/blog/:username", func(c *gin.Context) {
		//获取路径参数
		name := c.Param("username")
		c.JSON(200, gin.H{
			"name": name,
		})
	})

	r.GET("/blog/:year/:month", func(c *gin.Context) {
		//获取路径参数
		year := c.Param("year")
		month := c.Param("month")
		c.JSON(200, gin.H{
			"year":  year,
			"month": month,
		})
	})
	r.Run(":8080")
}
```

⚠️ 注意：这段代码**直接运行会 panic、无法启动**，原因是第三个路由和第二个路由存在通配符冲突。建议先读第五部分，再运行第六部分的修复版。

---

## 三、第一个路由：`c.Param` 基本用法

`/:username/:age` 是最简单的路径参数用法：路由里写几个 `:xxx`，处理器里就取几个。

```go
name := c.Param("username")
age := c.Param("age")
```

要点：

- 参数名必须与路由里的占位符**完全一致**（大小写敏感）
- 返回值永远是 `string`，`age` 取出来是 `"25"` 而不是数字 `25`

实测请求（gin v1.12.0）：

| 请求 | 响应 |
| --- | --- |
| `GET /tom/25` | `{"age":"25","name":"tom"}` |
| `GET /tom` | 404（段数不够，该路由要求两段） |
| `GET /tom/25/extra` | 404（段数太多） |

需要数字时自行转换：

```go
ageStr := c.Param("age")
age, err := strconv.Atoi(ageStr)
if err != nil {
	age = 0
}
```

---

## 四、第二个路由：静态地址优先

`/:username/:age` 与 `/blog/:username` 同时注册后，`/blog/...` 开头会被谁匹配？实测结果：

| 请求 | 响应 | 说明 |
| --- | --- | --- |
| `GET /blog/tom` | `{"name":"tom"}` | 走 `/blog/:username` |
| `GET /blog/2026` | `{"name":"2026"}` | 走 `/blog/:username`，而不是把 `blog` 当作 `:username` |
| `GET /tom/25` | `{"age":"25","name":"tom"}` | 走 `/:username/:age` |

原理：Gin 的路由树（radix tree）中，**静态段优先于参数段**。虽然 `/blog/2026` 也能被 `/:username/:age` 匹配成 `username=blog, age=2026`，但 Gin 优先匹配静态段 `blog`，所以 `blog` 不会被当成 `:username` 的值。这就是代码注释里"静态地址优先"的含义。

⚠️ 更准确地说：静态分支一旦被选中，匹配失败也**不会回退**去尝试参数分支。实测只有 `/archive/:year/:month` 和 `/:username/:age` 两个路由时：

| 请求 | 响应 |
| --- | --- |
| `GET /archive/2026/08` | `{"month":"08","year":"2026"}` |
| `GET /archive/2026` | 404（静态分支缺少一段；不会回退成 `username=archive, age=2026`） |

---

## 五、第三个路由：通配符名冲突（本项目最大的坑）

第三个路由 `r.GET("/blog/:year/:month", ...)` 在 gin v1.12.0 下启动即 panic：

```text
panic: ':year' in new path '/blog/:year/:month' conflicts with existing wildcard ':username' in existing prefix '/blog/:username'
```

原因：`/blog/:username` 和 `/blog/:year/:month` 在 `/blog/` 之后的**同一位置（第二段）**都使用了通配符，但参数名不同（`username` vs `year`）。Gin 的路由树要求同一位置的通配符**参数名必须一致**，否则注册路由时直接 panic。

这不是运行时的 404，而是**启动崩溃**；先注册哪个、后注册哪个都一样，只要同时存在就会炸。

修复方案（任选其一）：

### 方案一：换独立静态前缀（推荐）

```go
r.GET("/blog/:username", ...)
r.GET("/archive/:year/:month", ...)
```

实测：`GET /archive/2026/08` → `{"month":"08","year":"2026"}`

### 方案二：统一参数名

```go
r.GET("/blog/:username", ...)
r.GET("/blog/:username/:month", ...) // 第二段参数名统一为 username
```

实测：`GET /blog/2026/08` → `{"year":"2026","month":"08"}`（`year` 的值来自 `username` 占位）

### 方案三：删掉第三个路由

如果暂时不需要"一个静态前缀下多个参数"的演示，直接删除第三个路由即可启动。

---

## 六、修复后的完整可运行代码

以"独立静态前缀"方案为例：

```go
package main

import (
	"github.com/gin-gonic/gin"
)

func main() {
	r := gin.Default()
	// 1. 多个路径参数
	r.GET("/:username/:age", func(c *gin.Context) {
		name := c.Param("username")
		age := c.Param("age")
		c.JSON(200, gin.H{
			"name": name,
			"age":  age,
		})
	})

	// 2. 静态地址优先：/blog/xxx 固定走这里，blog 不会被当作 :username
	r.GET("/blog/:username", func(c *gin.Context) {
		name := c.Param("username")
		c.JSON(200, gin.H{
			"name": name,
		})
	})

	// 3. 多参数放到独立静态前缀，避免与 /blog/:username 参数名冲突
	r.GET("/archive/:year/:month", func(c *gin.Context) {
		year := c.Param("year")
		month := c.Param("month")
		c.JSON(200, gin.H{
			"year":  year,
			"month": month,
		})
	})

	r.Run(":8080")
}
```

---

## 七、运行与测试

```text
cd D:\ALLDocument\Code\go\gin5
go run .
```

浏览器直接访问，或用 curl：

```text
GET http://127.0.0.1:8080/tom/25
GET http://127.0.0.1:8080/blog/tom
GET http://127.0.0.1:8080/blog/2026
GET http://127.0.0.1:8080/archive/2026/08
```

```bash
curl http://127.0.0.1:8080/tom/25
```

⚠️ 原代码需要先按第五、六部分修复后才能启动。

---

## 八、注意事项

1. `c.Param` 的返回值永远是 `string`，需要数字时用 `strconv.Atoi` 转换。
2. 参数名大小写敏感，且必须与路由占位符一致；取一个不存在的参数名会得到空字符串 `""`。
3. 路径参数是"路径的一部分"：少一段或多一段都会 404；查询参数缺了则返回空字符串，两者行为不同。
4. 静态段优先于参数段，且静态分支匹配失败不会回退到参数分支。
5. 同一位置的通配符参数名必须一致，否则启动时 panic。
6. 同一位置可以同时存在静态段和参数段（如 `/user/new` 与 `/user/:id`），静态段优先，两者互不冲突。
7. 中文等特殊字符在 URL 中被编码后会自动解码，但 `%2F`（编码的斜杠）仍可能被当作路径分隔符，不建议放在路径参数值里。
8. 路由冲突会在启动时立刻暴露，是好事——尽早发现问题，而不是靠运行后的 404 慢慢排查。

---

## 九、相关扩展

| 方法/话题 | 作用 | 参考笔记 |
| --- | --- | --- |
| `c.Query` 系列 | 获取 URL 查询参数 | `gin-query-params.md` |
| `c.PostForm` 系列 | 获取表单参数 | `Gin表单接收学习笔记.md` |
| `c.ShouldBind(&u)` | 自动绑定到结构体 | `Gin参数绑定ShouldBind学习笔记.md` |
| `c.ShouldBindUri(&u)` | 把路径参数绑定到结构体 | 同上 |

路径参数也可以一次性绑定到结构体：

```go
type User struct {
	Username string `uri:"username"`
	Age      string `uri:"age"`
}

var u User
c.ShouldBindUri(&u)
```

---

## 十、练习

1. 修改 `/archive/:year/:month`，把 `year`、`month` 转成整数后再返回。
2. 新增 `/user/:id` 路由，返回 `{"id": "xxx"}`；再注册 `/user/new`，请求 `/user/new` 和 `/user/123`，观察静态优先的效果。
3. 把 `/blog/:username` 改成 `/blog/:year`，同时保留 `/blog/:year/:month`，看能否正常启动，理解"同一位置参数名必须一致"。

---

## 小结

路径参数用 `c.Param("参数名")` 获取，是最直观的 URL 传参方式。本项目两个核心知识点：一是**静态段优先于参数段**，匹配失败也不回退；二是**同一位置的通配符参数名必须一致**，否则启动 panic。修好第三个路由后，把四种请求各测一遍，就能完整理解 Gin 的路径匹配规则。
