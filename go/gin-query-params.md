# Gin 框架查询参数学习笔记

> 对应代码：`main.go` 中的 `GET /web` 接口
>
> 访问示例：`http://127.0.0.1:8080/web?query=f&age=18`

---

## 一、三个方法速览

| 方法 | 返回值 | 参数不存在时 | 适用场景 |
| --- | --- | --- | --- |
| `c.Query(key)` | `string` | 返回空字符串 `""` | 简单取值，不关心参数是否存在 |
| `c.DefaultQuery(key, defaultValue)` | `string` | 返回指定的默认值 | 参数可选，需要默认值兜底 |
| `c.GetQuery(key)` | `(string, bool)` | 返回 `("", false)` | 需要判断参数"是否存在" |

三个方法都用于获取 URL 查询参数（即 `?` 后面的部分），例如请求：

```
GET /web?query=f&age=18
```

此时：

- `c.Query("query")` 返回 `"f"`
- `c.Query("age")` 返回 `"18"`
- `c.Query("name")` 返回 `""`（参数不存在）

---

## 二、逐个拆解

### 1. `c.Query(key)` —— 最简单的取值

```go
name := c.Query("query")
age := c.Query("age")
```

特点：

- 只有一个返回值：`string`
- 参数存在就返回对应值，**不存在则返回空字符串**，不会报错
- 适合"参数可有可无"的场景

⚠️ 注意：空字符串有两种可能——参数不存在，或参数本身是空的（如 `?query=`）。`c.Query` 无法区分这两种情况。

### 2. `c.DefaultQuery(key, defaultValue)` —— 带默认值

```go
name := c.DefaultQuery("query", "没脑子")
```

特点：

- 参数存在且**不为空**时，返回参数值
- 参数不存在（或为空字符串）时，返回第二个参数指定的默认值
- 适合"参数可省略，省略时用默认值"的场景

等价写法：

```go
name := c.Query("query")
if name == "" {
    name = "没脑子"
}
```

### 3. `c.GetQuery(key)` —— 安全地判断是否存在

```go
name, ok := c.GetQuery("query")
if !ok {
    name = "没脑子"
}
```

特点：

- 返回两个值：`(value string, exists bool)`
- `ok` 为 `true` 表示参数**存在**，`false` 表示参数不存在
- 能真正区分"参数不存在"和"参数值为空字符串"

对比下面两种情况：

```go
// 请求：/web?query=
// c.Query("query")  -> ""
// c.GetQuery("query") -> ("", true)，注意 ok 是 true

// 请求：/web
// c.Query("query")  -> ""
// c.GetQuery("query") -> ("", false)
```

这就是你代码中选择 `c.GetQuery` 的原因：只有当 `query` 参数**真的没传**时，才给默认值"没脑子"。

---

## 三、对照你写的代码

```go
r.GET("/web", func(c *gin.Context) {
    age := c.Query("age")

    name, ok := c.GetQuery("query")
    if !ok {
        name = "没脑子"
    }

    c.JSON(200, gin.H{
        "name": name,
        "age":  age,
    })
})
```

三种请求的执行结果：

| 请求 | `name` | `age` |
| --- | --- | --- |
| `/web?query=f&age=18` | `f` | `18` |
| `/web?age=18` | `没脑子`（因为 `query` 没传） | `18` |
| `/web?query=&age=18` | `""`（传了但为空，`ok=true`，不会走默认值） | `18` |

> 💡 值得思考：如果这里用 `c.Query("query")` 代替，第三种请求返回的 `name` 也是 `""`；但如果你想在"参数为空"时也显示"没脑子"，就要改用 `c.DefaultQuery("query", "没脑子")`。

---

## 四、如何选择

1. **不关心参数是否存在**，取不到就用空字符串 → `c.Query(key)`
2. **参数可选，缺省时希望有默认值** → `c.DefaultQuery(key, defaultValue)`
3. **必须区分"没传"和"传了空值"**，或者要在取值后做进一步判断 → `c.GetQuery(key)`

简单记忆：

- `Query` = 查
- `DefaultQuery` = 查不到就用默认
- `GetQuery` = 查并告诉你有没有查到

---

## 五、注意事项

1. **返回值永远是字符串**

   查询参数没有类型，`age=18` 取出来是 `"18"` 而不是数字 `18`。需要数字时用 `strconv.Atoi` 转换：

   ```go
   ageStr := c.Query("age")
   age, err := strconv.Atoi(ageStr)
   if err != nil {
       age = 0
   }
   ```

2. **URL 编码自动解码**

   中文、空格等特殊字符在 URL 中会被编码（如 `query=%E4%BD%A0%E5%A5%BD`），`c.Query` 返回时已经解码为原始字符。

3. **同名参数多个值**

   如果 `?tag=a&tag=b`，`c.Query("tag")` 只返回第一个值 `"a"`。取全部值用 `c.QueryArray("tag")`。

4. **与表单参数（POST）的区别**

   `c.Query` 只取 URL 上的查询参数；POST 请求里表单提交的数据要用 `c.PostForm` / `c.GetPostForm` 获取。

---

## 六、相关扩展方法

| 方法 | 作用 |
| --- | --- |
| `c.QueryArray(key)` | 返回同名参数的所有值，类型 `[]string` |
| `c.GetQueryArray(key)` | 返回 `([]string, bool)`，可判断是否存在 |
| `c.QueryMap(key)` | 返回形如 `map[key]key[subkey]` 的键值对 |
| `c.GetQueryMap(key)` | 返回 `(map[string]string, bool)` |
| `c.PostForm(key)` | 获取 POST 表单字段 |
| `c.DefaultPostForm(key, default)` | 获取 POST 表单字段，带默认值 |
| `c.GetPostForm(key)` | 获取 POST 表单字段，返回 `(string, bool)` |

---

## 七、练习

改造你的 `/web` 接口，实现：

1. `age` 没传时默认 `0`
2. `query` 没传或为空时都显示"没脑子"
3. 把 `age` 转换为数字后返回

参考实现：

```go
r.GET("/web", func(c *gin.Context) {
    ageStr := c.DefaultQuery("age", "0")
    age, err := strconv.Atoi(ageStr)
    if err != nil {
        age = 0
    }

    name := c.DefaultQuery("query", "没脑子")

    c.JSON(200, gin.H{
        "name": name,
        "age":  age,
    })
})
```

---

## 小结

`c.Query`、`c.DefaultQuery`、`c.GetQuery` 是 Gin 获取 URL 查询参数最常用的三个方法，难度不大，关键是理解它们对"参数不存在"的处理差异：空字符串、默认值、`false`。根据是否需要区分"没传"和"传空"，选择合适的方法即可。
