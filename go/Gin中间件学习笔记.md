# Gin 中间件学习笔记

> 来源：`D:\ALLDocument\Code\go\gin10`
> 学习重点：`gin.New` 与 `gin.Default` 的区别、中间件的三种注册方式（全局 / 路由 / 分组）、`c.Next` 与 `c.Abort` 的执行机制、洋葱模型、中间件工厂函数、`c.Set` / `c.Get` 传值、goroutine 中必须使用 `c.Copy`
> 环境：Go 1.26.3、gin v1.12.0

---

## 一、项目结构

| 文件 | 作用 |
| --- | --- |
| `main.go` | 中间件定义、注册与全部路由处理逻辑 |
| `go.mod` / `go.sum` | Go module 与依赖信息（gin v1.12.0） |

本项目只有一个 `main.go`，内容围绕"中间件怎么定义、怎么注册、请求怎么穿过中间件"展开，是学习 Gin 中间件机制的最佳入门项目。

---

## 二、原项目代码

`main.go` 原文：

```go
package main

import (
	"fmt"
	"time"

	"github.com/gin-gonic/gin"
)

func m1(c *gin.Context) {
	fmt.Println("m1 in......")
	//计时
	// go funcXX(c.Copy()) //在中间件中使用groutine时不允许使用原始的上下文
	//只能用c.Copy()，保证c的只读，防止c不可控
	start := time.Now()
	c.Next() //调用后续函数
	cost := time.Since(start)
	fmt.Printf("cost:%v\n", cost)
	fmt.Println("m1 out......")
}
func m2(c *gin.Context) {
	fmt.Println("m2 in......")
	c.Next() //调用后续函数
	// c.Abort() //阻止调用后续函数
	fmt.Println("m2 out......")
}

// 一般写法
func outMiddleware(doCheck bool) gin.HandlerFunc {
	//连接数据库
	//准备其他工作
	return func(c *gin.Context) {
		c.Set("name", "zs")
		if doCheck {
			//登录判断
			// if xxx {
			// 	c.Next()
			// } else {
			// 	c.Abort()
			// 	return
			// }
		} else {
			c.Next()
		}
	}
}
func handler(c *gin.Context) {
	fmt.Println("index in......")
	c.JSON(200, gin.H{
		"message": "index",
	})
	fmt.Println("index out......")
}
func main() {
	// r := gin.Default() //默认带有两个中间件Logger, Recovery
	r := gin.New()
	r.Use(m1, m2, outMiddleware(true)) //全局注册中间件
	// r.GET("/index", m1, handler)//单一注册中间件
	r.GET("/index", handler)
	r.GET("/shop", func(c *gin.Context) {
		fmt.Println("shop in......")
		c.JSON(200, gin.H{
			"message": "shop",
		})
		fmt.Println("shop out......")
	})
	r.GET("/user", func(c *gin.Context) {
		fmt.Println("user in......")
		c.JSON(200, gin.H{
			"message": "user",
		})
		fmt.Println("user out......")
	})
	// 路由组注册中间件
	//方法一
	// xxGroup:=r.Group("/xx")
	// xxGroup.Use(outMiddleware(true))
	xxGroup := r.Group("/xx", outMiddleware(true)) //方法二
	{
		xxGroup.GET("/index", func(c *gin.Context) {
			fmt.Println("xx/index in......")
			name, ok := c.Get("name")
			if !ok {
				name = "匿名"
			}
			c.JSON(200, gin.H{
				"message": "xx/index",
				"name":    name,
			})
			fmt.Println("xx/index out......")
		})
	}
	r.Run()
}
```

这段代码可以正常运行，`r.Run()` 默认监听 `:8080`。

---

## 三、路由与中间件一览表

启动时 Gin 会在控制台打印每个路由的 handler 数量，本项目实际注册情况：

| 路由 | handler 数量 | 经过的中间件 | 响应 |
| --- | --- | --- | --- |
| `GET /index` | 4 | m1 → m2 → outMiddleware(true) → handler | `{"message":"index"}` |
| `GET /shop` | 4 | m1 → m2 → outMiddleware(true) → 内联 handler | `{"message":"shop"}` |
| `GET /user` | 4 | m1 → m2 → outMiddleware(true) → 内联 handler | `{"message":"user"}` |
| `GET /xx/index` | 5 | m1 → m2 → outMiddleware(true) → outMiddleware(true)（分组）→ 内联 handler | `{"message":"xx/index","name":"zs"}` |
| 未匹配路径 | 走 NoRoute | m1 → m2 → ... → 默认 404 handler | `404 page not found` |

启动日志原文：

```text
[GIN-debug] GET    /index                    --> main.handler (4 handlers)
[GIN-debug] GET    /shop                     --> main.main.func1 (4 handlers)
[GIN-debug] GET    /user                     --> main.main.func2 (4 handlers)
[GIN-debug] GET    /xx/index                 --> main.main.func3 (5 handlers)
```

关键观察：`/index`、`/shop`、`/user` 都是 4 个 handler（3 个全局中间件 + 1 个业务 handler）；`/xx/index` 是 5 个（全局 3 个 + 分组 1 个 + 业务 handler）。**中间件是层层叠加的，分组中间件会追加在全局中间件后面。**

---

## 四、知识点 1：`gin.New()` 与 `gin.Default()`

```go
r := gin.New()     // 本项目
// r := gin.Default()
```

源码里 `gin.Default()` 其实就是：

```go
engine := New()
engine.Use(Logger(), Recovery())
```

所以两者的关系是：

| 构造函数 | 自带中间件 | 说明 |
| --- | --- | --- |
| `gin.New()` | 无 | 干净起步，日志、恢复全由自己挂 |
| `gin.Default()` | `Logger()` + `Recovery()` | 默认打访问日志、panic 时恢复 |

本项目特意用 `gin.New()`，说明作者想"不掺杂质"地演示自定义中间件：控制台里看不到 Gin 默认的访问日志，只能看到自己 `fmt.Println` 的输出，方便观察执行顺序。

实际项目中更常见的组合是 `gin.New()` + 自定义日志中间件 + 自定义 Recovery，而不是直接使用 `gin.Default()`。

---

## 五、知识点 2：中间件是什么？怎么注册？

### 5.1 类型

在 Gin 中，中间件和普通路由 handler 是**同一个类型**：

```go
type HandlerFunc func(*Context)
```

`m1`、`m2`、`outMiddleware(true)` 的返回值、`r.GET("/index", handler)` 里的 `handler`，本质都是 `func(*Context)`。Gin 只是把一串 `HandlerFunc` 按注册顺序拼成一个"处理链"。

### 5.2 三种注册方式

**方式一：全局注册（`r.Use`）——本项目主用**

```go
r.Use(m1, m2, outMiddleware(true))
```

注册后，**所有后续注册的路由**都会先执行这三个中间件。

**方式二：路由级注册（单一注册）**

```go
// r.GET("/index", m1, handler)
```

项目里被注释掉了。把中间件直接写在路由参数里，只对这一个路由生效。

**方式三：路由组注册（两种写法）**

```go
// 方法一：创建后挂
// xxGroup := r.Group("/xx")
// xxGroup.Use(outMiddleware(true))

// 方法二：创建时挂（项目使用）
xxGroup := r.Group("/xx", outMiddleware(true))
```

两种写法等价，都是给**组内所有路由**统一加中间件。

### 5.3 注册顺序 = 执行顺序

`r.Use(m1, m2, outMiddleware(true))` 先写谁，谁就先执行。分组中间件会排在全局中间件之后、业务 handler 之前，所以 `/xx/index` 的完整链是：

```text
m1 → m2 → outMiddleware(true) → outMiddleware(true) → 业务 handler
```

同一个 `outMiddleware` 在 `/xx/index` 上出现了两次：一次全局、一次分组。

---

## 六、知识点 3：洋葱模型与 `c.Next()`

### 6.1 执行顺序

Gin 的中间件执行像"剥洋葱"：

```text
请求进入
  ↓
m1 in......
  ↓
m2 in......
  ↓
业务 handler in......
  ↓
业务 handler out......
  ↓
m2 out......
  ↓
m1 统计耗时 cost
m1 out......
  ↓
响应返回
```

一次 `GET /index` 的真实控制台输出（实测）：

```text
m1 in......
m2 in......
index in......
index out......
m2 out......
cost:527.1µs
m1 out......
```

这就是"洋葱模型"：**请求先进先出的穿过每个中间件的上半段（in），到达业务 handler，再逆序回到每个中间件的下半段（out）。**

### 6.2 `c.Next()` 的作用

```go
func m1(c *gin.Context) {
	start := time.Now()
	c.Next() // 调用后续函数
	cost := time.Since(start)
	fmt.Printf("cost:%v\n", cost)
}
```

`c.Next()` 表示"继续执行后面剩下的 handler"，它会在**后面全部执行完**之后才返回，所以 `m1` 里 `Next()` 后面的代码可以用来做"善后"：

- `m1`：`Next()` 前记时间，`Next()` 后算耗时——这就是一个最简性能监控中间件；
- `m2`：`Next()` 前后各打印一次，演示进/出顺序。

---

## 七、知识点 4（本项目的隐藏坑）：不调用 `c.Next()` 不一定会中断！

看 `outMiddleware`：

```go
func outMiddleware(doCheck bool) gin.HandlerFunc {
	return func(c *gin.Context) {
		c.Set("name", "zs")
		if doCheck {
			//登录判断
			// if xxx {
			// 	c.Next()
			// } else {
			// 	c.Abort()
			// 	return
			// }
		} else {
			c.Next()
		}
	}
}
```

注册时传的是 `outMiddleware(true)`，而 `if doCheck` 里**全是注释**——既没有调用 `c.Next()`，也没有调用 `c.Abort()`。按"调用 Next 才继续"的直觉，业务 handler 应该不会执行。

**但实测 `GET /index` 依然正常返回 `{"message":"index"}`，`index in/out` 也打印了！**

原因在于 `c.Next()` 的真实实现（gin v1.12.0 源码）：

```go
func (c *Context) Next() {
	c.index++
	for c.index < safeInt8(len(c.handlers)) {
		if c.handlers[c.index] != nil {
			c.handlers[c.index](c)
		}
		c.index++
	}
}
```

`c.Next()` 不是简单的"执行下一个，然后等它回来"，而是一个 **for 循环**：不断取 `handlers[c.index]` 执行、`index++`，直到链尾。整个链条的实际运行是这样的：

1. 引擎调用 `c.Next()`，从 index=0 开始执行 m1；
2. m1 里 `c.Next()` 继续执行 m2；
3. m2 里 `c.Next()` 继续执行 outMiddleware；
4. outMiddleware 返回（没有调用 Next），此时**控制权回到 m2 的 Next 循环里**，循环继续 `index++`，把业务 handler 也执行了；
5. 全部执行完，一层层返回，m2、m1 的 out 部分依次打印。

所以结论是：

- **不调用 `c.Next()` 不等于中断链**——外层调用者的 `Next()` 循环会继续往后执行；
- **真正中断链的唯一方式是 `c.Abort()`**（把 index 调到链尾之外）；
- 正确的登录校验写法就是注释里那套：**通过就 `c.Next()`，不通过就 `c.Abort()` 并 `return`**。

---

## 八、知识点 5：`c.Abort()` 才是真正的中断

### 8.1 原理

```go
const abortIndex int8 = math.MaxInt8 >> 1 // 63

func (c *Context) Abort() {
	c.index = abortIndex
}
```

`c.Abort()` 直接把 index 跳到 63，任何 `Next()` 的 for 循环都会立刻退出，后面的业务 handler 不会再执行。

### 8.2 对照实验

把 `m2` 改成"只 Abort、不 Next"：

```go
func m2(c *gin.Context) {
	fmt.Println("m2 in......")
	// c.Next()
	c.Abort()
	fmt.Println("m2 out......")
}
```

实测 `GET /index`：

| 项目 | 结果 |
| --- | --- |
| HTTP 状态码 | 200 |
| 响应体 | 空（没有任何 JSON） |
| 控制台日志 | `m1 in......` → `m2 in......` → `m2 out......` → `cost:0s` → `m1 out......` |

可以看到：

- 业务 handler 完全没执行；
- **但 m1 的 out 部分（cost、m1 out）依然执行了**——因为 m1 在 m2 之前，m2 返回后 m1 的 Next 循环发现 index 已是 63，直接退出，然后继续执行 m1 自己的收尾代码。

常用组合：

```go
c.AbortWithStatus(401)                       // 中断并写状态码
c.AbortWithStatusJSON(401, gin.H{"msg": "未登录"}) // 中断并写 JSON
```

---

## 九、知识点 6：中间件工厂函数（参数化中间件）

```go
func outMiddleware(doCheck bool) gin.HandlerFunc {
	//连接数据库
	//准备其他工作
	return func(c *gin.Context) {
		c.Set("name", "zs")
		...
	}
}
```

`outMiddleware` 返回的不是 `HandlerFunc` 本身，而是一个**返回 `gin.HandlerFunc` 的函数**，这就是"中间件工厂"：

- 闭包捕获参数 `doCheck`，同一个工厂可以按参数产出不同行为的中间件；
- "连接数据库、准备其他工作"放在**工厂函数里只执行一次**，而不是每个请求都执行——比写在返回的闭包里更高效；
- 注册时调用的是 `outMiddleware(true)` 或 `outMiddleware(false)`，传参决定行为。

典型登录校验完整版（把项目里注释的部分补全）：

```go
func auth(doCheck bool) gin.HandlerFunc {
	return func(c *gin.Context) {
		if doCheck {
			token := c.GetHeader("token")
			if token == "ok" {
				c.Next()
			} else {
				c.AbortWithStatusJSON(401, gin.H{"message": "未登录"})
				return
			}
		} else {
			c.Next()
		}
	}
}
```

---

## 十、知识点 7：`c.Set` / `c.Get` 在中间件与 handler 之间传值

```go
// 中间件里存
c.Set("name", "zs")

// handler 里取
name, ok := c.Get("name")
if !ok {
	name = "匿名"
}
```

`c.Keys` 是挂在 `*Context` 上的一个 map，**同一个请求的处理链共享**，所以：

- 中间件登录后把用户信息 `c.Set("user", userInfo)` 存进去；
- 后面的 handler 用 `c.Get("user")` 拿出来；
- 不同请求之间互不干扰。

实测 `GET /xx/index` 返回：

```json
{"message":"xx/index","name":"zs"}
```

注意：

- `c.Get` 返回 `(value, exists)`，取不到时 `exists=false`，本项目用 `ok` 判断后回退成"匿名"；
- `c.MustGet(key)` 取不到会 **panic**，只有确定存在时才用；
- 存入的是 `any`，取出后需要类型断言：`name.(string)`。

---

## 十一、知识点 8：goroutine 中必须使用 `c.Copy()`

项目注释：

```go
// go funcXX(c.Copy()) //在中间件中使用groutine时不允许使用原始的上下文
//只能用c.Copy()，保证c的只读，防止c不可控
```

为什么不能直接用原始 `c`：

- Gin 的 `*Context` 来自 `sync.Pool`，**请求处理完会被放回池子复用**；
- 如果在 goroutine 里继续持有并读写原始 `c`，主协程可能已经把 `c` 回收并用于下一个请求，造成**数据竞争、读到别人的请求**；
- goroutine 里的工作是"异步善后"，主流程已经结束，所以只需要**只读副本**。

`c.Copy()` 做了什么（源码要点）：

```go
cp := Context{
	writermem: c.writermem,
	Request:   c.Request,
	engine:    c.engine,
}
cp.writermem.ResponseWriter = nil  // 不能写响应
cp.index = abortIndex              // 不能再调用 Next
cp.handlers = nil
cp.Keys = 克隆一份                  // Keys 独立
```

正确用法：

```go
func m1(c *gin.Context) {
	copyC := c.Copy()
	go func() {
		// 异步任务：读取 copyC.Request.URL.Path、copyC.Get("xxx")
		// 只读，不要写响应
	}()
	c.Next()
}
```

---

## 十二、运行与测试

```text
cd D:\ALLDocument\Code\go\gin10
go run .
```

curl 测试命令：

```bash
curl http://127.0.0.1:8080/index
curl http://127.0.0.1:8080/shop
curl http://127.0.0.1:8080/user
curl http://127.0.0.1:8080/xx/index
curl http://127.0.0.1:8080/notexist
```

实测结果汇总：

| 请求 | 状态码 | 响应体 |
| --- | --- | --- |
| `GET /index` | 200 | `{"message":"index"}` |
| `GET /shop` | 200 | `{"message":"shop"}` |
| `GET /user` | 200 | `{"message":"user"}` |
| `GET /xx/index` | 200 | `{"message":"xx/index","name":"zs"}` |
| `GET /notexist` | 404 | 纯文本 `404 page not found` |

一次请求的控制台日志（`GET /index`，实测）：

```text
m1 in......
m2 in......
index in......
index out......
m2 out......
cost:527.1µs
m1 out......
```

未匹配路径也会执行全局中间件（`GET /notexist`，实测）：

```text
m1 in......
m2 in......
m2 out......
cost:503.9µs
m1 out......
```

因为 Gin 的 404 处理链 = 全局中间件 + 默认 NoRoute handler（源码 `allNoRoute = combineHandlers(noRoute)`），所以**全局中间件对未匹配路由同样生效**。

---

## 十三、注意事项

1. `gin.New()` 没有任何自带中间件；`gin.Default()` = `gin.New()` + `Logger()` + `Recovery()`，二者不要混用。
2. 中间件注册顺序就是执行顺序；全局 `r.Use` 要在注册路由**之前**调用，只对之后注册的路由生效。
3. `c.Next()` 的实现是 for 循环执行后续 handler，不是简单的"让出控制权"；它返回时，后续链已经全部执行完。
4. **不调用 `c.Next()` 不等于中断**——除非配合 `c.Abort()`，否则外层 Next 循环会继续把后面的 handler 执行完。这是本项目最大的隐藏坑。
5. `c.Abort()` 只跳过后面的 handler，**之前已执行的中间件的收尾代码仍会执行**（比如 m1 的 cost、m1 out）。
6. 分组中间件会与全局中间件叠加：`/xx/index` 有 5 个 handler，同一个 `outMiddleware` 执行了两次。
7. `c.Set` / `c.Get` 是"同一请求内"的上下文传递，取出后需要类型断言；`MustGet` 取不到会 panic。
8. 中间件里开 goroutine 只能用 `c.Copy()`，且副本是只读的，不能写响应、不能调 `Next`。
9. 全局中间件对 404 等未匹配路由也生效（实测 `GET /notexist` 时 m1、m2 都会执行）。
10. 单条链的 handler 数量有上限：Gin 源码断言 `finalSize < abortIndex(63)`，注册过多会启动 panic（too many handlers），实战中几乎不会碰到。
11. 登录校验等场景的正确姿势是：**通过 `c.Next()`，不通过 `c.Abort()` + `return`**，千万别只写 `return` 不写 Abort。

---

## 十四、练习

1. 把 `r := gin.New()` 改成 `r := gin.Default()`，重启后请求 `/index`，观察控制台多出的 Logger 访问日志，以及中间件执行顺序是否变化。
2. 把 `m2` 里 `c.Next()` 换成 `c.Abort()`，请求 `/index`，验证响应为空、handler 不执行，同时观察 m1 的 out 部分仍然打印。
3. 把 `outMiddleware` 里注释的登录判断补全：token 通过就 `c.Next()`，否则 `c.AbortWithStatusJSON(401, ...)`，再用带/不带 header 的请求验证：
   ```bash
   curl http://127.0.0.1:8080/xx/index
   curl -H "token: ok" http://127.0.0.1:8080/xx/index
   ```
4. 给 `/xx` 分组用方法一 `xxGroup.Use(...)` 再挂一个打印日志的中间件，观察 `/xx/index` 变成 6 个 handler 及执行顺序。
5. 把注释掉的 `r.GET("/index", m1, handler)` 恢复（先注销全局 `r.Use`），验证路由级中间件只对 `/index` 生效。
6. 在 `m1` 里用 `c.Copy()` 开一个 goroutine 打印 `copyC.Request.URL.Path`，观察异步输出时机。

---

## 小结

本项目用极简代码把 Gin 中间件的核心机制全部串了起来：**`gin.New()` vs `gin.Default()`** 说明中间件是"选装"的；**全局 / 路由 / 分组三种注册方式**说明中间件的挂载范围；**`c.Next()` 的洋葱模型**解释了请求如何穿过链条；而最容易踩坑的是——`c.Next()` 本质是一个循环，**忘记调用 Next 并不会中断，只有 `c.Abort()` 才能真正拦住后续 handler**。配合 `c.Set` / `c.Get` 传递登录信息、工厂函数按参数定制中间件、以及 goroutine 里 `c.Copy()` 的只读约束，就基本掌握了 Gin 中间件的全部核心用法。
