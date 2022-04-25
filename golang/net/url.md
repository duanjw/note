# url

一个标准的 URI 格式为：`[scheme:]scheme-specific-part`，如 `https://github.com/duanjw/note` 、
`mailto:474280116@qq.com`

URI 可以细分为为`不透明的 (opaque) `和`分层的 (hierarchical) `两类：
* `opaque` 指 `scheme-specific-part` 不以 / 开头，是一个整体。
呈 `[scheme]:opaque[?query][#fragment]` 的形式。如：mailto:474280116@qq.com, opaque 必须是绝对的。
* `hierarchical` 指 `scheme-specific-part` 以 / 开头且可以划分为好几部分。
呈 `[scheme:][//[userinfo@]host[:port]]path[?query][#fragment]` 的形式。如：
https://github.com/duanjw/note, hierarchical 可以是绝对的，也可以是相对的，如：`../../static/verify.js`

---

结构定义
````
type URL struct {
    Scheme      string    // http或https协议
    Opaque      string    // 编码不透明数据
    User        *Userinfo // 用户名密码信息--例如FTP需要带着用户名密码访问
    Host        string    // 主机或主机：端口
    Path        string    // 路径（相对路径可以省略前导斜线）
    RawPath     string    // 编码后的路径(如空格会被编码为%20)
    ForceQuery  bool      // 即使没有请求参数也会强制添加 ? 号
    RawQuery    string    // 编码不带 ? 号的请求参数
    Fragment    string    // 不带 # 号的引用片段
    RawFragment string    // 编码引用片段
}
````

---

## 主要函数

注意部分是未导出的函数，这里把导出函数排前面。
url.func 的方式都是指 url包下面的func函数
*URL.func 的方式都是指 URL结构体的引用

---

函数 `url.Parse(rawurl string) (*URL, error)` 用于将原始 url 解析为 URL 结构

1. `Parse` 会首先通过 `url.split()` 分离出 url(不带fragment) 和 fragment
2. 使用 `url.parse()` 解析构造 URL struct
   1. 这里 `viaRequest = false`，因此可以解析任意形式的 url
3. 如果存在 fragment 设置 fragment 后返回

---

函数 `url.ParseRequestURI(rawurl string) (*URL, error)` 用于解析一个 http 请求的 url，且假设是不带 Fragment 的

1. 使用 `url.parse()` 解析构造 URL struct
   1. 这里 `viaRequest = true`，因此只能解析http形式的 url
   
和 `url.Parse(rawurl string) (*URL, error)` 的区别在于不会分离 fragment ，和在调用 `url.parse()` 的时候 viaRequest 设置的是 true

---

函数 ` *URL.EscapedPath() string ` 编码 path 

获取URL的 RawPath 应该使用此函数，而不是直接读 u.RawPath

1. 使用 `validEncoded()` 判断 u.RawPath 是否规范
2. 规范则使用 `unescape()` 解码，然后和 u.Path 比较，相同则返回
3. u.RawPath 不规范或解码后和 u.Path 不同，直接使用 `escape` 编码 u.Path 后返回


---

函数 `url.parse()` 将不带 fragment 的 url 解析为 url 结构

主要实现解析：

1. url.parse 第二个参数用于判断 url 的类型，为 false 支持所有类型，true 只支持 `hierarchical` 的类型
2. 使用 `url.stringContainsCTLByte()` 判断是否带有 `ASCII control character`
   1. ASCII control character 是指 `第0～31号及第127号(共33个)`
   2. 这里 使用的是 `b < ' ' || b == 0x7f`，' ' 即 32号，'0x7f' 即 127号
3. 使用 `url.getScheme()` 解析分离协议部分(通过:),转化为小写后赋值到 url.Scheme
4. 解析 query 部分
   1. 如果是以 `?` 结尾，并且只出现了一次，`url.ForceQuery = true`
   2. 否则使用 url.split 分离 `?`
5. 通过判断是否已`/`开头(注意这里已经分离了协议了)来确定是否为 Opaque
6. 分离出 authority 信息，并且使用 `url.authority()` 解析出 url.User 和 url.Host
7. 使用 `url.setPath()` 设置 url.Path 和 url.RawPath
8. 返回 url 结构体

---


函数 `url.split(s string, sep byte, cutc bool)` 实现将 s 按照 sep 分割成两个字符串，cutc 用来判断是否保留sep

主要实现解析：

1. 使用 `strings.IndexByte()` 获取 sep 的索引位置
2. 通过 cutc 判断是否保留 sep 返回








