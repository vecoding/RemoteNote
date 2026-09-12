# Gin 文件上传学习笔记：FormFile / MultipartForm

> 项目位置：`D:\ALLDocument\Code\go\gin7`
> 技术栈：Go + Gin（v1.12.0）
> 学习目标：掌握 Gin 接收 `multipart/form-data` 文件上传的两种方式——`FormFile`（单文件）与 `MultipartForm`（多文件），以及使用中的注意事项

## 1. 项目结构

| 文件 | 作用 |
| --- | --- |
| main.go | 路由注册与文件上传处理逻辑 |
| index.html | 单文件上传表单，GET /index 时返回 |
| indexmore.html | 多文件上传表单（`multiple` 属性），GET /indexmore 时返回 |
| go.mod / go.sum | Go module 与依赖信息 |

## 2. 运行流程

1. 启动服务：`go run main.go`，Gin 默认监听 `:8080`
2. 浏览器访问 `http://127.0.0.1:8080/index`，选择单个文件并提交
3. 浏览器访问 `http://127.0.0.1:8080/indexmore`，表单里的 `<input type="file">` 带有 `multiple` 属性，可一次选择多个文件
4. 后端分别通过 `FormFile` / `MultipartForm` 读取文件，保存到服务端本地（当前保存在项目根目录）
5. 上传成功返回 JSON 提示

## 3. 前端表单：文件上传的三个前提

HTML 表单要上传文件，必须同时满足：

1. `method="post"`，用 POST 提交
2. `enctype="multipart/form-data"`，以 multipart 格式编码
3. `<input type="file">` 且有 `name` 属性，后端靠这个 name 取值

index.html：

```html
<form action="/upload" method="post" enctype="multipart/form-data">
    <input type="file" name="f1">
    <input type="submit" value="上传">
</form>
```

indexmore.html（只多了 `multiple`）：

```html
<form action="/uploadmore" method="post" enctype="multipart/form-data">
    <input type="file" name="f1" multiple>
    <input type="submit" value="上传">
</form>
```

> **注意：** 如果漏掉 `enctype="multipart/form-data"`，浏览器会按普通表单（`application/x-www-form-urlencoded`）提交，后端 `FormFile` / `MultipartForm` 都拿不到文件，会报 `request Content-Type isn't multipart/form-data` 之类的错误。

## 4. 核心代码 main.go

```go
package main

import (
	"fmt"
	"net/http"
	"path/filepath"

	"github.com/gin-gonic/gin"
)

func main() {
	r := gin.Default()
	// 处理 multipart forms 提交文件时默认内存限制 32 MiB（即缓存大小），
	// 可以通过 MaxMultipartMemory 修改
	// r.MaxMultipartMemory = 8 << 20 // 8 MiB

	r.LoadHTMLGlob("./*")

	r.GET("/index", func(c *gin.Context) {
		c.HTML(http.StatusOK, "index.html", nil)
	})

	// ---------- 单文件上传：FormFile ----------
	r.POST("/upload", func(c *gin.Context) {
		file, err := c.FormFile("f1")
		if err != nil {
			c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
			return
		}

		// 拼出保存路径。注意不要用 "./" + file.Filename 或 Sprintf，
		// 见第 9 节“路径拼接安全”
		dst := filepath.Join("./", file.Filename)
		c.SaveUploadedFile(file, dst)

		c.JSON(http.StatusOK, gin.H{"message": "上传成功"})
	})

	r.GET("/indexmore", func(c *gin.Context) {
		c.HTML(http.StatusOK, "indexmore.html", nil)
	})

	// ---------- 多文件上传：MultipartForm ----------
	r.POST("/uploadmore", func(c *gin.Context) {
		form, err := c.MultipartForm()
		if err != nil {
			c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
			return
		}

		// form.Value["xxx"]：返回字符串数组，存的是非文件类字段（文本框等）
		// form.File["f1"]：返回文件数组，键是表单字段名
		files := form.File["f1"]
		for _, file := range files {
			dst := filepath.Join("./", file.Filename)
			c.SaveUploadedFile(file, dst)
		}

		c.JSON(http.StatusOK, gin.H{
			"message": fmt.Sprintf("%d 个文件上传成功", len(files)),
		})
	})

	r.Run()
}
```

> 说明：项目原代码中 err 分支用了 `if ... else ...`，上面是去掉冗余 `else` 后的等价写法，逻辑完全一致。

## 5. FormFile：单文件上传

签名：

```go
func (c *Context) FormFile(name string) (*multipart.FileHeader, error)
```

Gin v1.12.0 源码实现：

```go
func (c *Context) FormFile(name string) (*multipart.FileHeader, error) {
	if c.Request.MultipartForm == nil {
		if err := c.Request.ParseMultipartForm(c.engine.MaxMultipartMemory); err != nil {
			return nil, err
		}
	}
	f, fh, err := c.Request.FormFile(name)
	if err != nil {
		return nil, err
	}
	f.Close()
	return fh, err
}
```

要点：

- **懒解析**：第一次调用时才触发 `ParseMultipartForm`，把请求体解析成 multipart form。
- **返回的是文件头 `*multipart.FileHeader`，不是打开的文件**。源码内部打开文件拿到 header 后立即 `Close()` 了。要读内容时用 `file.Open()`，要保存时用 `c.SaveUploadedFile(file, dst)`（它内部会重新 `Open`）。
- **只返回同名字段的第一个文件**。如果表单里一个字段传了多个文件，`FormFile` 也只能拿到第一个，所以多文件场景要用 `MultipartForm`。
- 字段不存在或没选文件时返回错误：`http: no such file`（即标准库的 `http.ErrMissingFile`）。

与 `c.Request.FormFile(name)` 的区别：

| 方式 | 返回值 | 文件句柄 |
| --- | --- | --- |
| `c.FormFile(name)`（Gin 封装） | `(*multipart.FileHeader, error)` | 内部已自动 Close |
| `c.Request.FormFile(name)`（标准库） | `(multipart.File, *multipart.FileHeader, error)` | 需要自己手动 `f.Close()` |

## 6. SaveUploadedFile：保存上传文件

签名（v1.12.0 起支持可选的权限参数）：

```go
func (c *Context) SaveUploadedFile(file *multipart.FileHeader, dst string, perm ...fs.FileMode) error
```

源码实现：

```go
func (c *Context) SaveUploadedFile(file *multipart.FileHeader, dst string, perm ...fs.FileMode) error {
	src, err := file.Open()
	if err != nil {
		return err
	}
	defer src.Close()

	var mode os.FileMode = 0o750
	if len(perm) > 0 {
		mode = perm[0]
	}
	dir := filepath.Dir(dst)
	if err = os.MkdirAll(dir, mode); err != nil {
		return err
	}
	if err = os.Chmod(dir, mode); err != nil {
		return err
	}

	out, err := os.Create(dst)
	if err != nil {
		return err
	}
	defer out.Close()

	_, err = io.Copy(out, src)
	return err
}
```

要点：

- 目标目录**不存在时会自动创建**，默认权限 `0750`；老版本固定 0750，v1.12.0 可以用第三个参数指定权限。
- 本质就是 `Open` 源文件 → 创建目标文件 → `io.Copy`。
- `dst` 是相对路径时，相对于**进程的工作目录**保存，不一定是项目根目录；生产环境建议用绝对路径拼接上传根目录。

## 7. MultipartForm：多文件上传

签名：

```go
func (c *Context) MultipartForm() (*multipart.Form, error)
```

Gin v1.12.0 源码实现：

```go
func (c *Context) MultipartForm() (*multipart.Form, error) {
	err := c.Request.ParseMultipartForm(c.engine.MaxMultipartMemory)
	return c.Request.MultipartForm, err
}
```

`multipart.Form` 的结构：

```go
type Form struct {
	Value map[string][]string
	File  map[string][]*multipart.FileHeader
}
```

- `form.Value["字段名"]`：存放**非文件类字段**（文本框、下拉框等），值是字符串数组。例如表单里有个 `<input name="username">`，就用 `form.Value["username"][0]` 取值。
- `form.File["字段名"]`：存放**文件字段**，值是 `*multipart.FileHeader` 数组。键是 HTML 里的 `name`，一个字段可以对应多个文件（`multiple` 属性或 curl 多次 `-F` 传同名）。

本项目就是 `form.File["f1"]` 拿到所有文件，循环调用 `SaveUploadedFile` 保存。

## 8. MaxMultipartMemory：内存缓存阈值，不是大小限制

默认值（gin.go 源码）：

```go
defaultMultipartMemory = 32 << 20 // 32 MB
```

```go
MaxMultipartMemory int64 // 传给 http.Request.ParseMultipartForm 的 maxMemory 参数
```

含义（标准库 `ParseMultipartForm` 文档）：

> The whole request body is parsed and up to a total of maxMemory bytes of its file parts are stored in memory, with the remainder stored on disk in temporary files.

即：解析 multipart 请求体时，文件部分**累计不超过 maxMemory 字节的放在内存里，超过的部分写入磁盘临时文件**。

三个容易踩的坑：

1. **它不是“上传大小限制”**。默认 32 MiB 只是内存阈值，超过 32 MiB 的文件照样能上传成功，只是超出部分会落到临时文件。
2. 修改方式：

```go
r := gin.Default()
r.MaxMultipartMemory = 8 << 20 // 8 MiB 内存阈值
```

3. 临时文件不需要手动清理。net/http 服务端在**请求处理完、返回响应后**会调用 `MultipartForm.RemoveAll()`（server.go 的 `finishRequest`），自动删除临时文件。如果想在 handler 内提前释放磁盘临时文件，也可以手动调用 `c.Request.MultipartForm.RemoveAll()`。

如果真的要限制上传大小，用 `http.MaxBytesReader`：

```go
r.POST("/upload", func(c *gin.Context) {
	// 限制请求体最大 10 MiB，超出后后续读取会报错
	c.Request.Body = http.MaxBytesReader(c.Writer, c.Request.Body, 10<<20)

	file, err := c.FormFile("f1")
	if err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
		return
	}
	// ...
})
```

## 9. 注意事项（重点）

### 9.1 表单三件套不能少

`POST` + `enctype="multipart/form-data"` + `<input type="file" name="...">`，缺一不可；后端取的 name 必须和表单里 `name` 完全一致。

### 9.2 FormFile 只取第一个文件

同一个字段传多个文件时，`FormFile` 只返回第一个，想要全部文件必须用 `MultipartForm` 或直接遍历 `c.Request.MultipartForm.File[name]`。

### 9.3 路径拼接安全（本项目注释的重点）

常见的三种写法：

```go
dst := fmt.Sprintf("./%s", file.Filename) // 不推荐
dst := "./" + file.Filename                // 不推荐
dst := filepath.Join("./", file.Filename)  // 推荐，但仍有隐患
```

前两种写法的问题：

- `file.Filename` 可能自带路径（恶意客户端可以伪造），字符串拼接会把路径直接带进来；
- Windows 和 Linux 的路径分隔符不同，拼接容易出跨平台问题。

`filepath.Join` 会调用 `Clean` 规范化路径，能处理分隔符问题。**但它并不能完全防目录穿越**：

```go
filepath.Join("./", "../evil.txt") // 结果是 ../evil.txt，仍然逃出了当前目录
```

更稳妥的做法是**只取文件名**，并校验最终路径仍在上传目录内：

```go
func safeDst(file *multipart.FileHeader) (string, error) {
	// 先统一把反斜杠换成斜杠，再取 basename，彻底去掉路径成分
	name := filepath.Base(strings.ReplaceAll(file.Filename, "\\", "/"))
	if name == "." || name == "/" || name == "" {
		return "", errors.New("非法的文件名")
	}

	// 扩展名白名单
	ext := strings.ToLower(filepath.Ext(name))
	if ext != ".txt" && ext != ".jpg" && ext != ".png" {
		return "", errors.New("不支持的文件类型")
	}

	// 加时间戳/随机数，避免同名覆盖，也避免文件名里的特殊字符
	dst := filepath.Join("./uploads", fmt.Sprintf("%d_%s", time.Now().UnixNano(), name))

	// 最后再校验一次：清理后的目标路径必须还在上传目录内
	uploadDir := filepath.Clean("./uploads")
	if !strings.HasPrefix(filepath.Clean(dst), uploadDir+string(os.PathSeparator)) {
		return "", errors.New("路径越界")
	}
	return dst, nil
}
```

### 9.4 文件名不可信

`file.Filename` 是客户端提供的，可以包含空字符串、超长字符串、`..`、`/`、`\`、特殊字符等。生产环境建议：

- 用 `filepath.Base` 只保留文件名，或干脆**改名**为随机字符串 + 白名单扩展名；
- 不要直接拿文件名做数据库主键或静态资源 URL；
- 不要把它拼进 HTML 或日志直接输出，防止注入。

### 9.5 只校验扩展名不够

扩展名可以被伪造，更可靠的是校验文件内容（magic bytes）或用 `http.DetectContentType` 判断真实类型：

```go
src, _ := file.Open()
defer src.Close()
buf := make([]byte, 512)
n, _ := src.Read(buf)
ct := http.DetectContentType(buf[:n]) // 根据文件头判断
```

另外，上传目录不要和可执行的静态资源目录混在一起，避免上传 `.html`/`.svg` 后触发存储型 XSS。

### 9.6 大文件场景

`SaveUploadedFile` 是整体拷贝，适合常规文件。超大文件或想边收边处理时，可以用 `c.Request.MultipartReader()` 手动逐 part 流式读取（注意：一旦用它手动读取，就不能再调 `MultipartForm` / `FormFile`，会返回 `http: multipart handled by MultipartReader` 错误）。

### 9.7 重复调用 MultipartForm 是安全的

标准库 `ParseMultipartForm` 幂等：`After one call to ParseMultipartForm, subsequent calls have no effect.`，所以 handler 里多次调用 `c.MultipartForm()` 或同时使用 `FormFile` + `MultipartForm` 都没有问题。

### 9.8 保存目录

项目里保存到 `./`（运行目录），学习没问题。生产环境建议：

- 单独建 `uploads` 目录并自动创建（`SaveUploadedFile` 会自动 `MkdirAll`）；
- 用绝对路径，不依赖工作目录；
- 按日期分目录，避免单目录文件过多；
- 定期清理无用文件。

## 10. FormFile vs MultipartForm 对比

| 对比项 | `c.FormFile(name)` | `c.MultipartForm()` |
| --- | --- | --- |
| 适用场景 | 单文件上传 | 多文件、文件 + 普通字段混合 |
| 返回值 | `(*multipart.FileHeader, error)` | `(*multipart.Form, error)` |
| 同名字段多文件 | 只拿第一个 | 全部拿到 |
| 普通文本字段 | 拿不到，需配合 `PostForm` | `form.Value[字段名]` |
| 文件字段 | 直接返回 header | `form.File[字段名]` 返回数组 |
| 底层 | 内部同样走 `ParseMultipartForm` | 直接调 `ParseMultipartForm` |
| 典型错误 | 没文件时返回 `http: no such file` | 解析失败返回 error |

## 11. 用 curl 测试

Windows PowerShell 里 `curl` 是 `Invoke-WebRequest` 的别名，请用 `curl.exe`：

单文件：

```powershell
curl.exe -F "f1=@D:\test.txt" http://127.0.0.1:8080/upload
```

多文件（同名 `-F` 传多次）：

```powershell
curl.exe -F "f1=@D:\a.txt" -F "f1=@D:\b.txt" http://127.0.0.1:8080/uploadmore
```

文件和普通字段一起传：

```powershell
curl.exe -F "f1=@D:\a.txt" -F "username=zhangsan" http://127.0.0.1:8080/uploadmore
```

后端用 `form.Value["username"]` 就能拿到 `["zhangsan"]`。

## 12. multipart 请求体长什么样

理解底层结构有助于理解 `FormFile` 和 `MultipartForm` 在解析什么：

```http
POST /upload HTTP/1.1
Content-Type: multipart/form-data; boundary=----WebKitFormBoundary7MA4YWxkTrZu0gW

------WebKitFormBoundary7MA4YWxkTrZu0gW
Content-Disposition: form-data; name="f1"; filename="hello.txt"
Content-Type: text/plain

hello, gin
------WebKitFormBoundary7MA4YWxkTrZu0gW
Content-Disposition: form-data; name="username"

zhangsan
------WebKitFormBoundary7MA4YWxkTrZu0gW--
```

每个 part 用 `boundary` 分隔，`Content-Disposition` 里的 `name` 就是 `FormFile` / `form.File` 的 key，`filename` 就是 `FileHeader.Filename`。解析器按 `name` 分组：有 `filename` 的进 `Form.File`，没有的进 `Form.Value`。

## 13. 小结

- 单文件：`c.FormFile("f1")` → `c.SaveUploadedFile(file, dst)`，三步完成上传。
- 多文件：`c.MultipartForm()` → `form.File["f1"]` 循环保存。
- `MaxMultipartMemory` 是**内存缓存阈值**（默认 32 MiB），不是大小限制；限制大小用 `http.MaxBytesReader`。
- `filepath.Join` 优于字符串拼接，但防目录穿越要 `filepath.Base` + 路径前缀校验。
- 永远不要信任客户端传来的文件名、扩展名和内容类型。
