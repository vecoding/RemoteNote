# Gin 重定向 Redirect 与内部转发 HandleContext 学习笔记

> 来源：`D:\ALLDocument\Code\go\gin8`
> 学习重点：`c.Redirect` 实现 HTTP 重定向；`r.HandleContext` 实现服务器内部转发
> 环境：Go 1.26.3、gin v1.12.0

---

## 一、两个概念先分清

| 对比项 | 重定向 Redirect | 内部转发 HandleContext |
| --- | --- | --- |
| 本质 | 服务器返回 3xx 状态码 + `Location` 响应头 | 服务器内部把同一个请求重新交给路由匹配一次 |
| 地址栏 URL | 变成新地址 | 不变 |
| 网络请求次数 | 2 次（浏览器收到 3xx 后自动再发一次） | 1 次 |
| 客户端能否感知 | 能感知（收到 301/302） | 完全无感知 |
| 典型场景 | 域名迁移、登录后跳转、跳外链 | 内部路由分发、统一入口转发 |

一句话总结：

- **Redirect**：服务器对浏览器说"你要的东西不在这里，去新地址找"；
- **HandleContext**：服务器自己说"这个请求换个 handler 处理"，浏览器全程不知情。

---

## 二、原项目代码

`main.go` 原文（原文件注释为 GBK 编码，用 UTF-8 打开会显示乱码，以下按原意整理为中文）：

```go
package main

import (
	"net/http"

	"github.com/gin-gonic/gin"
)

func main() {
	r := gin.Default()
	// 路由重定向
	r.GET("/index", func(c *gin.Context) {
		c.Redirect(http.StatusMovedPermanently, "https://www.doubao.com")
		// 临时重定向，浏览器不会缓存
		// c.Redirect(http.StatusFound, "https://www.baidu.com")
		// 注意学习各个状态码的含义
	})
	// 服务器内部转发，地址栏的 URL 不变
	r.GET("/a", func(c *gin.Context) {
		// 跳转到 b 对应路由
		c.Request.URL.Path = "/b"
		r.HandleContext(c)
	})
	r.GET("/b", func(c *gin.Context) {
		c.JSON(http.StatusOK, gin.H{
			"status": "b"})
	})

	r.Run(":8080")
}
```

项目很小，只注册了三个路由：

| 路由 | 行为 |
| --- | --- |
| `GET /index` | 301 永久重定向到 `https://www.doubao.com` |
| `GET /a` | 内部转发给 `/b` 处理 |
| `GET /b` | 返回 JSON `{"status":"b"}` |

---

## 三、c.Redirect：HTTP 重定向

### 3.1 基本用法

```go
c.Redirect(状态码, 目标地址)
```

项目中的两行：

```go
c.Redirect(http.StatusMovedPermanently, "https://www.doubao.com") // 301 永久重定向
// c.Redirect(http.StatusFound, "https://www.baidu.com")           // 302 临时重定向（注释掉了）
```

`http.StatusMovedPermanently` 就是 301，`http.StatusFound` 就是 302。`net/http` 包把常用状态码都定义成了常量，建议写常量而不是裸数字。

实际响应长这样（`curl -i` 可以看到）：

```text
HTTP/1.1 301 Moved Permanently
Content-Type: text/html; charset=utf-8
Location: https://www.doubao.com
Date: ...
Content-Length: ...

<a href="https://www.doubao.com">Moved Permanently</a>.
```

要点：

- 核心是响应头 `Location`，浏览器看到 3xx + Location 就会自动发起第二次请求；
- 正文是一个很短的 HTML 链接，兼容老浏览器；
- 目标地址可以是绝对地址（如 `https://www.doubao.com`），也可以是相对地址（如 `/b`），Go 会自动把相对地址拼成绝对地址。

### 3.2 301 与 302 的区别（本项目重点）

| 状态码 | 含义 | 浏览器/搜索引擎行为 |
| --- | --- | --- |
| 301 Moved Permanently | 永久重定向 | 会**缓存**，下次直接访问新地址，不再请求旧地址 |
| 302 Found | 临时重定向 | **不会缓存**，每次都会先访问旧地址再被"踢"到新地址 |

项目注释里"临时重定向，浏览器不会缓存"说的就是这个区别。

调试建议：**改代码阶段先用 302**。301 一旦被浏览器缓存，改了代码再访问旧地址，浏览器可能根本不会发请求到你的服务器，容易产生"改了没生效"的错觉。

### 3.3 其他常见重定向状态码

| 状态码 | 名称 | 语义 |
| --- | --- | --- |
| 301 | Moved Permanently | 永久重定向，方法可能被改为 GET |
| 302 | Found | 临时重定向，方法可能被改为 GET |
| 303 | See Other | 临时重定向，强制改用 GET |
| 307 | Temporary Redirect | 临时重定向，**保留原方法**（POST 还是 POST） |
| 308 | Permanent Redirect | 永久重定向，**保留原方法** |

简单理解：301/302 历史最久但语义模糊，浏览器常把 POST 变成 GET；307/308 是"严格版"，保证请求方法不变。网页跳转用 301/302 就够，API 跳转建议用 307/308。

### 3.4 注意：状态码不是随便传的

Gin 的 `Redirect` 内部会对状态码做校验，只允许 300~308（另有 201 特例），传其他值会直接 panic：

```text
panic: Cannot redirect with status code 400
```

所以状态码写错会在请求时立刻崩掉，而不是"悄悄不跳转"。

---

## 四、r.HandleContext：服务器内部转发

### 4.1 基本用法

```go
r.GET("/a", func(c *gin.Context) {
	c.Request.URL.Path = "/b" // 把路径改成目标路由
	r.HandleContext(c)        // 让路由引擎用新路径重新匹配一次
})
```

实测效果（`curl -i http://127.0.0.1:8080/a`）：

```text
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8

{"status":"b"}
```

关键观察点：

- 浏览器地址栏访问 `/a`，**URL 一直是 `/a`**，不会变成 `/b`；
- 浏览器只发了一次请求，响应体是 `/b` handler 写出来的 JSON；
- 整个过程中没有 3xx、没有 `Location`，客户端完全不知道内部发生了转发。

### 4.2 原理：把请求重新"喂"给路由引擎

`HandleContext` 的源码（gin v1.12.0）：

```go
func (engine *Engine) HandleContext(c *Context) {
	oldIndexValue := c.index
	oldHandlers := c.handlers
	c.reset()                   // 清空当前 handler 链、参数、Keys 等
	engine.handleHTTPRequest(c) // 按 c.Request.URL.Path 重新匹配路由并执行
	c.index = oldIndexValue     // 恢复原来的执行位置
	c.handlers = oldHandlers    // 恢复原来的 handler 链
}
```

执行流程：

1. 请求进入 `/a` 的 handler，代码把 `c.Request.URL.Path` 改成 `/b`；
2. `HandleContext` 保存当前执行状态，然后 `reset()` 清空上下文里的路由参数、Keys、错误等；
3. 路由引擎拿着修改后的 Path（HTTP 方法还是 GET）重新匹配路由树，找到 `/b` 的 handler 链并执行；
4. 执行完后再恢复原来的 index 和 handlers，返回外层。

所以它本质是"**递归调用一次路由匹配**"，不是直接调用 `/b` 的函数。

### 4.3 与重定向的本质区别

| 区别点 | `c.Redirect` | `r.HandleContext` |
| --- | --- | --- |
| 响应内容 | 3xx + Location，让浏览器再请求 | 直接返回目标 handler 的结果 |
| 地址栏 | 变 | 不变 |
| 请求次数 | 2 次 | 1 次 |
| 谁能看到 | 客户端参与 | 只有服务器内部发生 |
| 数据传递 | 靠 URL/查询参数 | 同一次请求上下文（但注意 4.4） |

### 4.4 注意事项

1. **HTTP 方法不变**：`HandleContext` 只重匹配路径，不改变方法。`/a` 是 GET，转发目标 `/b` 也必须注册 GET；如果目标只注册了 POST，会匹配失败返回 404。

2. **中间件会再执行一遍**：`c.reset()` 把 handler 链清空后，`handleHTTPRequest` 会重新拿目标路由的完整链（包括 `gin.Default()` 注册的 Logger、Recovery 等全局中间件）。也就是说一次 `/a` 请求，日志中间件会跑两遍（一遍 `/a`、一遍 `/b`）。

3. **转发前不要写响应**：如果在调用 `HandleContext` 之前已经写了 `c.JSON` 之类的响应，响应头可能已经发出，再转发会出问题。正确姿势是"改了路径马上转发"。

4. **上下文会被清空**：`c.reset()` 会清掉 `Params`、`Keys`、`Errors` 等。在 `/a` 里 `c.Set("user", "tom")` 设置的键，到了 `/b` 里 `c.Get("user")` 是拿不到的。需要传数据，要么先存到局部变量再塞回去，要么用 URL 查询参数。

5. **目标路由不存在就 404**：如果 `/b` 没注册，转发后返回 404，而不是报错。

6. **一般只需改 Path**：本项目只改了 `c.Request.URL.Path`，Gin 默认按它匹配，日常学习够用；如果设置了 `engine.UseRawPath` 等选项，可能还要同步 `RawPath`，暂时不需要深究。

---

## 五、运行与测试

```text
cd D:\ALLDocument\Code\go\gin8
go run .
```

**测试 1：重定向**

```bash
curl -i http://127.0.0.1:8080/index
```

观察输出里的 `301 Moved Permanently` 和 `Location: https://www.doubao.com`。

**测试 2：内部转发**

```bash
curl -i http://127.0.0.1:8080/a
```

看到 `200 OK` 和 `{"status":"b"}`，且响应头里没有任何 Location。

**测试 3：对比 301/302**

把 `main.go` 里的 301 换成 302（取消注释那一行），重新 `go run .`，再 `curl -i` 请求 `/index`：

```text
301: HTTP/1.1 301 Moved Permanently
302: HTTP/1.1 302 Found
```

浏览器测试更直观：访问 `http://127.0.0.1:8080/index` 会跳到豆包；访问 `http://127.0.0.1:8080/a` 页面显示 `{"status":"b"}`，但地址栏始终是 `/a`。

---

## 六、注意事项汇总

1. `c.Redirect` 只允许 300~308（201 特例），写错状态码会 panic。
2. 301 会被浏览器/搜索引擎缓存，调试期优先用 302；改代码后想立刻看到效果，可以清缓存或开无痕窗口。
3. `Location` 可以是相对路径，Go 会自动补全成绝对地址。
4. `HandleContext` 是"重新匹配路由"，不是函数调用：方法不变、中间件重跑、目标不存在会 404。
5. 转发前不要写响应；需要传递的数据要先保存（`reset` 会清空 Keys/Params）。
6. 重定向和转发的核心区别就一句话：**重定向是让浏览器换地址，转发是服务器内部换 handler**。

---

## 七、相关扩展

| 话题 | 说明 |
| --- | --- |
| `c.Abort` / `c.AbortWithStatusJSON` | 中断当前 handler 链，常与鉴权跳转配合 |
| `http.StatusTemporaryRedirect` (307) | 保留 POST 等方法的临时重定向 |
| 相对路径重定向 | `c.Redirect(302, "/b")`，观察 Location 自动补全 |
| 中间件顺序 | 全局中间件在注册路由时已并入 handler 链，所以 HandleContext 会重跑 |
| `c.Redirect` 的响应体 | 重定向不需要自己写正文，Gin 会生成默认 HTML 链接 |

---

## 八、练习

1. 把 `/index` 改为 302 重定向到百度（取消注释），用 `curl -i` 观察状态码和 Location，再改回 301，比较浏览器行为差异。
2. 把 `HandleContext` 改成相对地址重定向：`c.Redirect(http.StatusFound, "/b")`，访问 `/a`，观察地址栏变成 `/b` 且响应是 `{"status":"b"}`，体会两者区别。
3. 新增一个 `/c` 路由，让 `/a` 转发到 `/c`，在 `/c` 里打印一条日志，观察一次请求日志出现了几次，理解"中间件重跑"。
4. 在 `/a` 里 `c.Set("msg", "hello")` 后转发到 `/b`，在 `/b` 里尝试读取，观察拿不到的结果，理解 `reset()` 清空上下文。
5. 给 `/b` 注册 POST 版本（`r.POST("/b", ...)`），让 `/a` 转发过去，观察是否返回 404，理解"方法不变"。

---

## 小结

本项目用三个路由演示了 Gin 最常用的两种"跳转"方式：`c.Redirect` 是标准 HTTP 重定向（301/302），服务器只负责"指路"，浏览器再发一次请求；`r.HandleContext` 是服务器内部转发，改一下 `c.Request.URL.Path` 再重新匹配一次路由，地址栏不变、请求只有一次。学习时重点对比：**状态码含义（301 缓存/302 不缓存）、转发时中间件重跑、reset 清空上下文**，把这三点吃透，这两个 API 就掌握得差不多了。
