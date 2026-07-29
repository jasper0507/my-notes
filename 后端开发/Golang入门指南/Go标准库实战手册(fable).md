# Go 标准库实战手册

> 基于 Go 1.26。标注了版本号的 API(如 1.22+)在旧版本不可用,未标注的各版本通用。

## fmt

### Common Functions

| 函数 | 用途 | 典型示例 | 返回结果 |
| --- | --- | --- | --- |
| `fmt.Printf` | 格式化输出到 stdout | `fmt.Printf("id=%d\n", id)` | 通常忽略 |
| `fmt.Sprintf` | 格式化并返回字符串 | `fmt.Sprintf("user-%d", id)` | `string` |
| `fmt.Fprintf` | 格式化写入任意 `io.Writer` | `fmt.Fprintf(os.Stderr, "warn: %v\n", err)` | `(int, error)` |
| `fmt.Errorf` | 构造(可包装的)错误 | `fmt.Errorf("load config: %w", err)` | `error` |

### 常用格式化动词

| 动词 | 含义 | 示例 | 输出 |
| --- | --- | --- | --- |
| `%v` | 默认格式,万能 | `fmt.Printf("%v", u)` | `{1 jasper}` |
| `%+v` | 结构体带字段名,调试首选 | `fmt.Printf("%+v", u)` | `{ID:1 Name:jasper}` |
| `%#v` | Go 语法形式,能看出类型 | `fmt.Printf("%#v", u)` | `main.User{ID:1, Name:"jasper"}` |
| `%q` | 带引号并转义,排查不可见字符 | `fmt.Printf("%q", "a\tb")` | `"a\tb"` |
| `%T` | 值的类型 | `fmt.Printf("%T", u)` | `main.User` |
| `%d` `%s` `%t` | 整数、字符串、布尔 | `fmt.Printf("%d", 42)` | `42` |
| `%.2f` | 浮点精度控制 | `fmt.Printf("%.2f", 3.14159)` | `3.14` |
| `%w` | 包装错误(仅限 `fmt.Errorf`) | `fmt.Errorf("query: %w", err)` | 保留错误链 |

### fmt.Errorf 与 %w

```go
if err != nil {
	return fmt.Errorf("get user %d: %w", id, err)
}
```

> `%w` 和 `%v` 打印出来一样,区别在于 `%w` 保留了原错误,后续 `errors.Is` / `errors.AsType` 才能沿链匹配。描述写"当时在做什么",不要加 `error:` 这类废话前缀。

### 自定义打印:Stringer 接口

类型实现 `String() string` 后,`%v` 会自动调用它:

```go
func (u User) String() string { return fmt.Sprintf("User(%d)", u.ID) }
```

常用于枚举打印成可读名称、日志脱敏(密码字段永远打印 `***`)。

> 动词和参数类型不匹配不会报错,只会输出 `%!d(string=abc)` 这种形式。提交前跑 `go vet`,它专门检查这类问题。

## strings

### Common Functions

| 函数 | 用途 | 典型示例 | 返回结果 |
| --- | --- | --- | --- |
| `strings.TrimSpace` | 去首尾空白,处理用户输入第一步 | `strings.TrimSpace(" a ")` | `"a"` |
| `strings.Contains` | 是否包含子串 | `strings.Contains(s, "@")` | `bool` |
| `strings.HasPrefix` / `HasSuffix` | 前缀、后缀判断 | `strings.HasPrefix(s, "https://")` | `bool` |
| `strings.Split` | 按分隔符切分 | `strings.Split("a,b,c", ",")` | `[]string` |
| `strings.Join` | 拼接切片 | `strings.Join(parts, ",")` | `string` |
| `strings.ReplaceAll` | 全部替换 | `strings.ReplaceAll(s, "\r\n", "\n")` | `string` |
| `strings.EqualFold` | 忽略大小写比较 | `strings.EqualFold("Go", "GO")` | `bool` |
| `strings.Cut` | 按第一个分隔符切成两半(1.18+) | `strings.Cut("key=value", "=")` | `(before, after, found)` |
| `strings.Builder` | 循环内高效拼接 | 见下 | `string` |

### strings.Cut

```go
k, v, ok := strings.Cut("timeout=30s", "=")
// k="timeout"  v="30s"  ok=true
```

解析 `key=value`、`host:port` 一类结构的首选,取代了大量 `SplitN` + 下标访问的写法。

### strings.Builder

```go
var b strings.Builder
for _, x := range items {
	b.WriteString(x)
}
s := b.String()
```

> 为什么循环里不用 `+=`?字符串不可变,`+=` 每轮都分配新内存并整体拷贝;`Builder` 内部复用缓冲区,只在最后生成一次字符串。

## strconv

### Common Functions

| 函数 | 用途 | 典型示例 | 返回结果 |
| --- | --- | --- | --- |
| `strconv.Atoi` | string → int | `strconv.Atoi("42")` | `(int, error)` |
| `strconv.Itoa` | int → string | `strconv.Itoa(42)` | `string` |
| `strconv.ParseFloat` | string → float64 | `strconv.ParseFloat("3.14", 64)` | `(float64, error)` |
| `strconv.ParseBool` | string → bool | `strconv.ParseBool("true")` | `(bool, error)` |
| `strconv.FormatInt` | 整数按进制转字符串 | `strconv.FormatInt(255, 16)` | `"ff"` |

### Atoi / Itoa

```go
n, err := strconv.Atoi(r.URL.Query().Get("page"))
if err != nil {
	http.Error(w, "page 必须是数字", http.StatusBadRequest)
	return
}
```

> 为什么不用 `fmt.Sprintf("%d", n)` 做转换?数字与字符串互转一律 `strconv`:更快、语义清晰,而且 `Parse` 系列强制处理 `error`——这是外部输入进入类型系统的关卡。

## encoding/json

### Common Functions

| 函数 | 用途 | 典型示例 | 返回结果 |
| --- | --- | --- | --- |
| `json.Marshal` | 结构体 → JSON 字节 | `json.Marshal(u)` | `([]byte, error)` |
| `json.Unmarshal` | JSON 字节 → 结构体 | `json.Unmarshal(data, &u)` | `error` |
| `json.NewDecoder(...).Decode` | 从流(如 `r.Body`)解析 | `json.NewDecoder(r.Body).Decode(&req)` | `error` |
| `json.NewEncoder(...).Encode` | 序列化写入流 | `json.NewEncoder(w).Encode(resp)` | `error` |
| `json.RawMessage` | 某字段延迟解析 | `Payload json.RawMessage` | 原样字节 |

### struct tag

```go
type User struct {
	ID    int    `json:"id"`
	Name  string `json:"name"`
	Email string `json:"email,omitempty"` // 零值时省略该字段
	Pwd   string `json:"-"`               // 永不序列化
}
```

### Marshal / Unmarshal

```go
data, err := json.Marshal(u)
err = json.Unmarshal(data, &u)   // 第二个参数必须是指针
```

### 流式:HTTP 场景

```go
var req CreateUserReq
if err := json.NewDecoder(r.Body).Decode(&req); err != nil { /* 400 */ }

w.Header().Set("Content-Type", "application/json")
json.NewEncoder(w).Encode(resp)
```

不要先 `io.ReadAll` 再 `Unmarshal`,Decoder 直接对流工作。

### 高频坑

> **序列化结果是 `{}`?** 字段首字母没大写。未导出字段会被静默忽略、不报错——这是最常见的 json bug。

> **`omitempty` 区分不了"没传"和"传了零值"。** `Age int` 传 0 和不传,解析后都是 0。需要区分时字段用指针 `Age *int`,没传则为 `nil`。

> 解析到 `any` 时,所有 JSON 数字都是 `float64`。结构未知用 `map[string]any`;想把某字段留到以后再解析,用 `json.RawMessage`。

## time

### Common Functions

| 函数 | 用途 | 典型示例 | 返回结果 |
| --- | --- | --- | --- |
| `time.Now` | 当前时间 | `time.Now()` | `time.Time` |
| `time.Parse` | 按 UTC 解析字符串 | `time.Parse("2006-01-02", s)` | `(time.Time, error)` |
| `time.ParseInLocation` | 按指定时区解析 | `time.ParseInLocation(layout, s, loc)` | `(time.Time, error)` |
| `t.Format` | 时间 → 字符串 | `t.Format(time.RFC3339)` | `string` |
| `time.Since` | 距 start 过了多久 | `time.Since(start)` | `time.Duration` |
| `t.Add` / `t.Sub` | 时间运算 | `t.Add(24 * time.Hour)` | `Time` / `Duration` |
| `t.Unix` / `time.Unix` | 时间戳互转 | `time.Unix(sec, 0)` | `int64` / `Time` |
| `time.NewTicker` | 周期任务 | `time.NewTicker(time.Minute)` | `*Ticker` |

### 参考时间

Go 不用 `YYYY-MM-DD` 这类占位符,而是用一个**固定的参考时间**当模板:

```go
t, err := time.Parse("2006-01-02 15:04:05", "2026-07-26 10:00:00")
```

> 为什么是 2006-01-02 15:04:05?按美式顺序它正好是 1、2、3、4、5、6(1 月 2 日下午 3 点 4 分 5 秒 2006 年)。模板里写别的日期会解析出错乱结果,这是新手第一坑。

### 时区

```go
loc, _ := time.LoadLocation("Asia/Shanghai")
t, err := time.ParseInLocation("2006-01-02", "2026-07-26", loc)
```

`time.Parse` 在字符串不含时区信息时按 UTC 处理。实战约定:**存储与传输一律 UTC + `time.RFC3339`**(`time.Time` 的 JSON 序列化默认就是它),展示层再转用户时区。

### Duration 与比较

```go
timeout := 3 * time.Second
elapsed := time.Since(start)
if t1.Before(t2) { /* ... */ }
if t1.Equal(t2) { /* ... */ }
```

> 为什么比较不用 `==`?`time.Time` 内部带单调时钟读数和时区指针,同一时刻的两个值 `==` 可能为 false。相等判断永远用 `Equal`。

### Ticker

```go
ticker := time.NewTicker(time.Minute)
defer ticker.Stop()
for range ticker.C {
	refreshCache()
}
```

## errors

### Common Functions

| 函数 | 用途 | 典型示例 | 返回结果 |
| --- | --- | --- | --- |
| `errors.New` | 创建固定错误 | `errors.New("用户名不能为空")` | `error` |
| `fmt.Errorf` | 创建带变量的错误 | `fmt.Errorf("用户 %d 不存在", id)` | `error` |
| `%w` | 包装原始错误 | `fmt.Errorf("读取配置失败: %w", err)` | 保留原错误 |
| `errors.Is` | 判断错误链中是否存在某个错误值 | `errors.Is(err, os.ErrNotExist)` | `bool` |
| `errors.AsType` | 提取指定类型的错误(1.26+) | `errors.AsType[*fs.PathError](err)` | 错误值和 `bool` |
| `errors.As` | `AsType` 的旧写法 | `errors.As(err, &pathErr)` | `bool` |
| `errors.Join` | 合并多个错误(1.20+) | `errors.Join(err1, err2)` | `error` |

### 哨兵错误 + %w 包装:标准工作流

```go
var ErrUserNotFound = errors.New("user not found")   // 哨兵:包级导出,当常量用

func getUser(id int) (*User, error) {
	u, err := queryDB(id)
	if errors.Is(err, sql.ErrNoRows) {
		return nil, fmt.Errorf("查询用户 %d: %w", id, ErrUserNotFound)
	}
	if err != nil {
		return nil, fmt.Errorf("查询用户 %d: %w", id, err)
	}
	return u, nil
}
```

调用方:

```go
u, err := getUser(1001)
if errors.Is(err, ErrUserNotFound) {
	http.Error(w, "not found", http.StatusNotFound)
	return
}
```

> 为什么不直接写 `err == ErrUserNotFound`?因为 err 往往已经被包装过:
>
> ```go
> err := fmt.Errorf("读取配置失败: %w", os.ErrNotExist)
> fmt.Println(err == os.ErrNotExist)          // false
> fmt.Println(errors.Is(err, os.ErrNotExist)) // true
> ```
>
> `errors.Is` 会沿错误链逐层查找。同理,永远不要用 `strings.Contains(err.Error(), ...)` 判断错误。

### errors.AsType

`errors.Is` 匹配错误**值**,`errors.AsType` 匹配错误**类型**并取出,用于读取错误携带的字段:

```go
_, err := os.Open("config.json")
pathErr, ok := errors.AsType[*fs.PathError](err)
if ok {
	fmt.Println("操作:", pathErr.Op)
	fmt.Println("路径:", pathErr.Path)
}
```

1.26 之前的等价写法(存量代码里大量存在,要认识):

```go
var pathErr *fs.PathError
if errors.As(err, &pathErr) {
	fmt.Println(pathErr.Path)
}
```

需要携带数据时,自定义错误类型:

```go
type ValidationError struct{ Field string }

func (e *ValidationError) Error() string { return "invalid field: " + e.Field }
```

### errors.Join

批量操作收集所有错误:

```go
var errs []error
for _, item := range batch {
	if err := process(item); err != nil {
		errs = append(errs, err)
	}
}
return errors.Join(errs...)   // 空切片返回 nil;errors.Is 对每个成员都生效
```

> 日志纪律:每层包装只加"当时在做什么"的上下文;同一个错误只在最终处理处打**一次**日志。层层都打的话,一个错误刷五行,排查反而困难。

## context

### Common Functions

| 函数 | 用途 | 典型示例 | 返回结果 |
| --- | --- | --- | --- |
| `context.Background` | 链路起点(main、测试) | `context.Background()` | `Context` |
| `context.WithTimeout` | 带超时的子 ctx | `context.WithTimeout(ctx, 2*time.Second)` | `(Context, CancelFunc)` |
| `context.WithCancel` | 手动取消的子 ctx | `context.WithCancel(ctx)` | `(Context, CancelFunc)` |
| `ctx.Done` | 取消信号 channel | `<-ctx.Done()` | `<-chan struct{}` |
| `ctx.Err` | 取消原因 | `ctx.Err()` | `Canceled` / `DeadlineExceeded` |
| `context.WithValue` | 挂请求级元数据 | `context.WithValue(ctx, key, tid)` | `Context` |

### 标准形状

```go
func fetchUser(ctx context.Context, id int) (*User, error) {
	ctx, cancel := context.WithTimeout(ctx, 2*time.Second)
	defer cancel()   // 必须 defer,否则计时器和子协程泄漏

	req, _ := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
	// ...
}
```

规则:`ctx` 永远是**第一个参数**;只沿调用链传递,不存进 struct 字段。

### 与 HTTP 服务端集成

handler 里用 `r.Context()`:客户端断开连接时它自动取消,数据库查询、下游调用全部跟着终止,不做无用功——这是 Go 后端最优雅的机制之一。

```go
func getUser(w http.ResponseWriter, r *http.Request) {
	u, err := store.Get(r.Context(), r.PathValue("id"))
	// ...
}
```

### select 等待取消

```go
select {
case <-ctx.Done():
	return ctx.Err()   // context.Canceled 或 context.DeadlineExceeded
case res := <-ch:
	return res, nil
}
```

### WithValue 的边界

只放请求级元数据(trace id、当前登录用户),业务参数老老实实走函数签名。key 用自定义未导出类型:

```go
type traceKey struct{}

ctx = context.WithValue(ctx, traceKey{}, traceID)
tid, _ := ctx.Value(traceKey{}).(string)
```

> 为什么 key 不用 string?不同包用相同字符串 key 会互相覆盖;未导出类型保证只有本包能读写这个值。

## io / os

### Common Functions

| 函数 | 用途 | 典型示例 | 返回结果 |
| --- | --- | --- | --- |
| `os.ReadFile` | 一次读整个小文件 | `os.ReadFile("config.yaml")` | `([]byte, error)` |
| `os.WriteFile` | 一次写整个文件 | `os.WriteFile("out.txt", data, 0644)` | `error` |
| `os.Open` | 打开文件流式处理 | `os.Open("big.log")` | `(*os.File, error)` |
| `io.Copy` | 流式拷贝 | `io.Copy(dst, src)` | `(int64, error)` |
| `io.ReadAll` | 读完整个流 | `io.ReadAll(resp.Body)` | `([]byte, error)` |
| `os.Getenv` / `LookupEnv` | 读环境变量 | `os.LookupEnv("DEBUG")` | `string` / `(string, bool)` |
| `os.MkdirAll` | 递归建目录 | `os.MkdirAll("data/logs", 0755)` | `error` |
| `filepath.Join` | 跨平台路径拼接 | `filepath.Join("data", "a.txt")` | `string` |

### 小文件与大文件

```go
data, err := os.ReadFile("config.yaml")   // 小文件一把梭

f, err := os.Open("big.log")              // 大文件流式
if err != nil {
	return err
}
defer f.Close()                           // 紧跟 err 检查之后写
```

### Reader / Writer:组合哲学

文件、网络连接、HTTP body、`bytes.Buffer` 实现的是同一对接口,所以工具函数对它们通吃:

```go
n, err := io.Copy(dstFile, resp.Body)                    // 下载到文件,内存占用恒定
body, err := io.ReadAll(io.LimitReader(r.Body, 1<<20))   // 限 1MB,防恶意大 body
```

### 环境变量与文件存在性

```go
port := os.Getenv("PORT")
v, ok := os.LookupEnv("DEBUG")   // 区分"未设置"和"设为空串"

if _, err := os.Stat(path); errors.Is(err, fs.ErrNotExist) {
	// 文件不存在
}
```

> 路径永远用 `filepath.Join`,不要硬编码 `/`。在 Windows 上开发、部署到 Linux 时,这是最容易踩的跨平台坑。

## bufio

### Common Functions

| 函数 | 用途 | 典型示例 | 返回结果 |
| --- | --- | --- | --- |
| `bufio.NewScanner` | 按行读取 | `bufio.NewScanner(f)` | `*Scanner` |
| `sc.Scan` / `sc.Text` | 迭代、取当前行 | `for sc.Scan() { sc.Text() }` | `bool` / `string` |
| `sc.Err` | 循环结束后取错误 | `sc.Err()` | `error` |
| `sc.Buffer` | 调大单行上限 | 见下 | — |
| `bufio.NewWriter` + `Flush` | 缓冲写 | `defer w.Flush()` | — |

### 按行读

```go
sc := bufio.NewScanner(f)
for sc.Scan() {
	line := sc.Text()
	// ...
}
if err := sc.Err(); err != nil {   // 必须查:Scan 返回 false 可能是出错而非读完
	return err
}
```

> Scanner 默认单行上限 64KB,超长行报 `token too long`。处理来源不明的数据先调大:`sc.Buffer(make([]byte, 0, 1024*1024), 1024*1024)`。

### 缓冲写

```go
w := bufio.NewWriter(f)
defer w.Flush()   // 忘记 Flush,缓冲区里的数据直接丢
for _, line := range lines {
	fmt.Fprintln(w, line)
}
```

## net/http(服务端)

### Common Functions

| 函数 | 用途 | 典型示例 | 返回结果 |
| --- | --- | --- | --- |
| `http.NewServeMux` | 路由器 | `mux := http.NewServeMux()` | `*ServeMux` |
| `mux.HandleFunc` | 注册路由(1.22+ 支持方法与路径参数) | `mux.HandleFunc("GET /users/{id}", h)` | — |
| `r.PathValue` | 取路径参数(1.22+) | `r.PathValue("id")` | `string` |
| `http.Server` | 可配置超时的服务器 | 见下 | — |
| `http.Error` | 写错误响应 | `http.Error(w, "not found", 404)` | — |
| `srv.Shutdown` | 优雅关闭 | `srv.Shutdown(ctx)` | `error` |

### 最小可上线骨架

```go
func main() {
	mux := http.NewServeMux()
	mux.HandleFunc("GET /health", func(w http.ResponseWriter, r *http.Request) {
		w.WriteHeader(http.StatusOK)
	})
	mux.HandleFunc("GET /users/{id}", getUser)
	mux.HandleFunc("POST /users", createUser)

	srv := &http.Server{
		Addr:         ":8080",
		Handler:      logging(mux),
		ReadTimeout:  5 * time.Second,
		WriteTimeout: 10 * time.Second,
		IdleTimeout:  60 * time.Second,
	}
	log.Fatal(srv.ListenAndServe())
}
```

> 为什么不用一行 `http.ListenAndServe(":8080", nil)`?它没有任何超时配置,一个只连接不发数据的慢客户端就能一直占住连接,攒多了服务就被拖死。生产必须自己构造 `http.Server`。

### 路由规则(1.22+)

`"GET /users/{id}"` 匹配方法 + 精确路径;不写方法则匹配所有方法;以 `/` 结尾是前缀匹配(如 `"/static/"`);handler 里 `r.PathValue("id")` 取参数。

### 典型 handler:JSON 进出 + 错误分支

```go
func getUser(w http.ResponseWriter, r *http.Request) {
	u, err := store.Get(r.Context(), r.PathValue("id"))
	if errors.Is(err, ErrUserNotFound) {
		http.Error(w, "not found", http.StatusNotFound)
		return   // 每个错误分支处理完必须 return
	}
	if err != nil {
		http.Error(w, "internal error", http.StatusInternalServerError)
		return
	}
	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(u)
}

func createUser(w http.ResponseWriter, r *http.Request) {
	var req CreateUserReq
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		http.Error(w, "bad json", http.StatusBadRequest)
		return
	}
	// ...
}
```

> Header 必须在写状态码或 body **之前**设置,写完再 `w.Header().Set` 无效。

### 中间件

签名 `func(http.Handler) http.Handler`,整个 Go 生态通用的形状:

```go
func logging(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		start := time.Now()
		next.ServeHTTP(w, r)
		slog.Info("req", "method", r.Method, "path", r.URL.Path, "dur", time.Since(start))
	})
}
```

多个中间件像洋葱一样套:`recover(logging(auth(mux)))`。

### 优雅关闭

收到 SIGTERM 后停止接新请求、等存量请求处理完再退出(K8s 滚动发布必备):

```go
ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
defer stop()

go func() { log.Println(srv.ListenAndServe()) }()

<-ctx.Done()
shutCtx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()
srv.Shutdown(shutCtx)
```

## net/url

### Common Functions

| 函数 | 用途 | 典型示例 | 返回结果 |
| --- | --- | --- | --- |
| `url.Parse` | 解析 URL | `url.Parse(raw)` | `(*url.URL, error)` |
| `u.Query` | 取 query 参数集 | `u.Query().Get("q")` | `url.Values` |
| `url.Values` + `Encode` | 构造 query 串 | 见下 | `string` |
| `url.QueryEscape` | 转义 query 参数 | `url.QueryEscape("a b")` | `"a+b"` |
| `url.PathEscape` | 转义路径段 | `url.PathEscape("a b")` | `"a%20b"` |

### 构造与解析

```go
q := url.Values{}
q.Set("q", "go web")
q.Set("page", "2")
u.RawQuery = q.Encode()   // q=go+web&page=2,自动转义

u2, _ := url.Parse("https://api.example.com/search?q=go&page=2")
page := u2.Query().Get("page")
```

> 永远不要手拼 query 字符串——参数里出现空格、`&`、中文就出 bug。注意 `QueryEscape` 和 `PathEscape` 规则不同(空格分别转成 `+` 和 `%20`),用错位置服务端解析结果会不一致。

## sync

### Common Functions

| 函数 | 用途 | 典型示例 | 返回结果 |
| --- | --- | --- | --- |
| `sync.WaitGroup` | 等一批 goroutine 结束 | `wg.Add(1)` / `wg.Done()` / `wg.Wait()` | — |
| `wg.Go` | Add/Done 的封装(1.25+) | `wg.Go(func() { ... })` | — |
| `sync.Mutex` | 互斥锁 | `mu.Lock()` + `defer mu.Unlock()` | — |
| `sync.RWMutex` | 读写锁,读多写少 | `mu.RLock()` | — |
| `sync.OnceValue` | 惰性单次初始化(1.21+) | `sync.OnceValue(loadConfig)` | `func() T` |

### WaitGroup

```go
var wg sync.WaitGroup
for _, url := range urls {
	wg.Add(1)   // Add 在 go 之前调
	go func() {
		defer wg.Done()
		fetch(url)   // 1.22+ 循环变量每轮独立,不用再写 url := url
	}()
}
wg.Wait()
```

1.25+ 的简写:`wg.Go(func() { fetch(url) })`,省掉 Add/Done 样板。

### Mutex 保护共享状态

```go
type Counter struct {
	mu sync.Mutex
	m  map[string]int
}

func (c *Counter) Inc(k string) {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.m[k]++
}
```

> map 并发写不是返回错误,是直接 `fatal error: concurrent map writes`——整个进程崩溃,recover 都救不了。共享 map 必须加锁。开发期常跑 `go run -race` / `go test -race`,数据竞争检测器是保命工具。

### sync.OnceValue

```go
var getConfig = sync.OnceValue(loadConfig)   // 首次调用真正执行,之后返回缓存结果

cfg := getConfig()
```

### 实战补充:errgroup

"并发跑任务 + 收集第一个错误 + 限制并发数"这个组合,标准库要手搓,准标准库 `golang.org/x/sync/errgroup` 一步到位,后端项目几乎人手一个:

```go
g, ctx := errgroup.WithContext(ctx)
g.SetLimit(10)   // 最多 10 个并发
for _, u := range urls {
	g.Go(func() error { return fetch(ctx, u) })
}
if err := g.Wait(); err != nil { /* 第一个非 nil 错误 */ }
```

## channel 与 select

### Common Operations

| 操作 | 用途 | 典型示例 | 说明 |
| --- | --- | --- | --- |
| `make(chan T)` | 无缓冲 channel | `make(chan int)` | 收发双方到齐才通过 |
| `make(chan T, n)` | 带缓冲 | `make(chan Result, 10)` | 缓冲满才阻塞发送 |
| `close(ch)` | 关闭(仅发送方调用) | `close(jobs)` | 通知"不再有数据" |
| `for v := range ch` | 持续接收 | 见下 | channel 关闭后自动退出 |
| `v, ok := <-ch` | 接收并感知关闭 | `ok == false` 即已关闭 | — |
| `select` | 多路等待 | 见下 | 配超时与取消 |

### worker pool

```go
jobs := make(chan int)
results := make(chan Result, 10)

for i := 0; i < 3; i++ {   // 3 个 worker
	go func() {
		for j := range jobs {   // jobs 关闭后循环自动退出
			results <- process(j)
		}
	}()
}

for _, j := range work {
	jobs <- j
}
close(jobs)   // 发送方关闭
```

### select:超时与取消

```go
select {
case r := <-results:
	handle(r)
case <-ctx.Done():
	return ctx.Err()
case <-time.After(2 * time.Second):
	return ErrTimeout
}
```

> 铁律:只有发送方 close;向已关闭 channel 发送会 panic,重复 close 也 panic;nil channel 收发永远阻塞。最常见的泄漏是 goroutine 永远卡在没人接收的发送上——发送侧配 `select` + `ctx.Done()` 兜底。经验法则:channel 传递数据所有权,Mutex 保护共享状态,别拿 channel 硬造锁。

## log/slog

### Common Functions

| 函数 | 用途 | 典型示例 | 返回结果 |
| --- | --- | --- | --- |
| `slog.Info` / `Error` 等 | 分级结构化日志 | `slog.Info("login", "user_id", 42)` | — |
| `slog.New` + Handler | 构造 logger | `slog.New(slog.NewJSONHandler(...))` | `*Logger` |
| `slog.SetDefault` | 设为全局默认 | `slog.SetDefault(logger)` | — |
| `slog.With` | 派生带公共字段的 logger | `slog.With("trace_id", tid)` | `*Logger` |
| `slog.InfoContext` | 带 ctx 记录 | `slog.InfoContext(ctx, "msg")` | — |
| `slog.Int` 等 | 强类型属性 | `slog.Int("user_id", 42)` | `Attr` |

### 初始化与使用

```go
logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
	Level: slog.LevelInfo,
}))
slog.SetDefault(logger)

slog.Info("user login", "user_id", 42, "ip", ip)
// {"time":"...","level":"INFO","msg":"user login","user_id":42,"ip":"..."}
slog.Error("db query failed", "err", err, "sql", q)

reqLog := slog.With("trace_id", tid)
reqLog.Info("request start")
```

> key 和 value 必须成对出现,写漏一个会输出 `!BADKEY`。想要更强保障用类型化写法:`slog.Info("login", slog.Int("user_id", 42))`。

约定:`JSONHandler` 给生产环境(日志系统机器采集),`TextHandler` 给本地开发;服务代码一律 slog,老的 `log` 包只留给一次性脚本。

## testing

### Common Functions

| 函数 | 用途 | 典型示例 | 返回结果 |
| --- | --- | --- | --- |
| `t.Run` | 子测试 | `t.Run(tt.name, func(t *testing.T) {...})` | — |
| `t.Errorf` | 记录失败,继续执行 | `t.Errorf("got %d", got)` | — |
| `t.Fatalf` | 记录失败,终止当前用例 | `t.Fatalf("setup: %v", err)` | — |
| `t.Helper` | 标记测试辅助函数 | 函数开头调 `t.Helper()` | 报错定位到调用处 |
| `httptest.NewRequest` / `NewRecorder` | 不起服务器测 handler | 见下 | — |

### 表驱动测试

社区统一风格,加一个用例就是加一行:

```go
func TestParseAge(t *testing.T) {
	tests := []struct {
		name    string
		in      string
		want    int
		wantErr bool
	}{
		{"normal", "18", 18, false},
		{"empty", "", 0, true},
		{"negative", "-1", 0, true},
	}
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			got, err := ParseAge(tt.in)
			if (err != nil) != tt.wantErr {
				t.Fatalf("err = %v, wantErr %v", err, tt.wantErr)
			}
			if got != tt.want {
				t.Errorf("got %d, want %d", got, tt.want)
			}
		})
	}
}
```

### 测 HTTP handler

```go
req := httptest.NewRequest(http.MethodGet, "/users/1", nil)
rec := httptest.NewRecorder()
mux.ServeHTTP(rec, req)

if rec.Code != http.StatusOK {
	t.Fatalf("code = %d", rec.Code)
}
```

### 常用命令

```text
go test ./...                 # 跑全部包
go test -run TestParseAge -v  # 只跑指定测试,显示细节
go test -cover                # 覆盖率
go test -race                 # 并发代码必跑
go test -bench=.              # 基准测试:func BenchmarkX(b *testing.B)
```

> `Fatalf` 与 `Errorf` 怎么选?前置条件失败(连不上依赖、构造失败)用 `Fatalf`,继续跑没有意义;断言失败用 `Errorf`,让同一用例的其他断言也跑完,一次看到全部差异。

## slices / maps(1.21+)

### Common Functions

| 函数 | 用途 | 典型示例 | 返回结果 |
| --- | --- | --- | --- |
| `slices.Contains` | 是否包含元素 | `slices.Contains(ids, 42)` | `bool` |
| `slices.Sort` | 排序(有序类型) | `slices.Sort(names)` | 原地排序 |
| `slices.SortFunc` | 自定义排序 | 见下 | 原地排序 |
| `slices.Max` / `Min` | 最值 | `slices.Max(nums)` | 元素 |
| `maps.Keys` | 键迭代器(1.23+) | `slices.Collect(maps.Keys(m))` | `[]K` |

```go
slices.SortFunc(users, func(a, b User) int { return cmp.Compare(a.Age, b.Age) })
keys := slices.Collect(maps.Keys(m))
```

以前要手写循环的高频操作现在都有官方实现,新代码优先用。

---

**综合练习建议**:只用标准库写一个用户 CRUD 小服务,把 encoding/json、errors、context、net/http、slog、testing 串起来——JSON 收发、错误链、超时取消、路由中间件、结构化日志、handler 测试,一个项目全覆盖。
