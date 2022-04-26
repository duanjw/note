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

---

函数 `url.Parse(rawurl string) (*URL, error)` 用于将原始 url 解析为 URL 结构

````

func Parse(rawURL string) (*URL, error) {
   // 分离出 Fragment
   u, frag := split(rawURL, '#', true)
   // 解析构造 URL struct 
   // 这里 `viaRequest = false`，因此可以解析任意形式的 url
   url, err := parse(u, false)
   if err != nil {
      return nil, &Error{"parse", u, err}
   }
   if frag == "" {
      return url, nil
   }
   // 设置 fragment
   if err = url.setFragment(frag); err != nil {
      return nil, &Error{"parse", rawURL, err}
   }
   return url, nil
}

````

---

函数 `url.ParseRequestURI(rawurl string) (*URL, error)` 用于解析一个 http 请求的 url，且假设是不带 Fragment 的

````
func ParseRequestURI(rawURL string) (*URL, error) {
   // 这里 `viaRequest = true`，因此只能解析http形式的 url
   // 和 `url.Parse(rawurl string) (*URL, error)` 的区别在于不会分离 fragment ，
   // 和在调用 `url.parse()` 的时候 viaRequest 设置的是 true
   url, err := parse(rawURL, true)
   if err != nil {
      return nil, &Error{"parse", rawURL, err}
   }
      return url, nil
}

````

---

函数 ` *URL.EscapedPath() string ` 编码 path 

获取URL的 RawPath 应该使用此函数，而不是直接读 u.RawPath

````
func (u *URL) EscapedPath() string {
   // 使用 `validEncoded()` 判断 u.RawPath 是否规范
   if u.RawPath != "" && validEncoded(u.RawPath, encodePath) {
      // 解码，和 u.path 比较, 相等返回
      p, err := unescape(u.RawPath, encodePath)
      if err == nil && p == u.Path {
         return u.RawPath
      }
   }
      if u.Path == "*" {
         return "*" // don't escape (Issue 11202)
      }
   // 其他情况直接编码 u.path
   return escape(u.Path, encodePath)
}
````

---

函数 `url.parse()` 将不带 fragment 的 url 解析为 url 结构

````
func parse(rawURL string, viaRequest bool) (*URL, error) {
   var rest string
   var err error
   // 判断是否带有 `ASCII control character`
   // 1. ASCII control character 是指 `第0～31号及第127号(共33个)`
   // 2. 这里 使用的是 `b < ' ' || b == 0x7f`，' ' 即 32号，'0x7f' 即 127号
   if stringContainsCTLByte(rawURL) {
      return nil, errors.New("net/url: invalid control character in URL")
   }
   
   if rawURL == "" && viaRequest {
      return nil, errors.New("empty url")
   }
   url := new(URL)
   
   if rawURL == "*" {
      url.Path = "*"
      return url, nil
   }
   
   // 解析分离协议部分(通过:),转化为小写后赋值到 url.Scheme
   if url.Scheme, rest, err = getScheme(rawURL); err != nil {
      return nil, err
   }
   url.Scheme = strings.ToLower(url.Scheme)
   // 是否已 ? 结尾，并且只出现一次
   if strings.HasSuffix(rest, "?") && strings.Count(rest, "?") == 1 {
      url.ForceQuery = true
      rest = rest[:len(rest)-1]
   } else {
      //  去除 ? 后，赋值 RawQuery 返回
   r  est, url.RawQuery = split(rest, '?', true)
   }
   // 是否以 / 开头，注意这里已经分离了协议
   // 不是 / 开头就代表为 opaque 的形式
   if !strings.HasPrefix(rest, "/") {
      // 协议不为空赋值 Opaque 返回
      if url.Scheme != "" {
         // We consider rootless paths per RFC 3986 as opaque.
         url.Opaque = rest
         return url, nil
      }
      // viaRequest = true 不能是 opaque 的形式
      if viaRequest {
         return nil, errors.New("invalid URI for request")
      }
   
      // 格式不符合规范，报错
      colon := strings.Index(rest, ":")
      slash := strings.Index(rest, "/")
      if colon >= 0 && (slash < 0 || colon < slash) {
      // First path segment has colon. Not allowed in relative URL.
         return nil, errors.New("first path segment in URL cannot contain colon")
      }
   }
   // 协议不为空，并且以 // 开头，表示为 hierarchical 的形式
   if (url.Scheme != "" || !viaRequest && !strings.HasPrefix(rest, "///")) && strings.HasPrefix(rest, "//") {
      var authority string
      // 分离出 authority 并解析 User 和 host
      authority, rest = split(rest[2:], '/', false)
      url.User, url.Host, err = parseAuthority(authority)
      if err != nil {
         return nil, err
      }
   }
   // Set Path and, optionally, RawPath.
   // RawPath is a hint of the encoding of Path. We don't want to set it if
   // the default escaping of Path is equivalent, to help make sure that people
   // don't rely on it in general.
   if err := url.setPath(rest); err != nil {
      return nil, err
   }
   return url, nil
}
````

---


函数 `url.split(s string, sep byte, cutc bool)` 实现将 s 按照 sep(首次出现) 分割成两个字符串，cutc 用来判断是否保留sep

````
func split(s string, sep byte, cutc bool) (string, string) {
   // 获取 sep 的索引位置
   i := strings.IndexByte(s, sep)
   if i < 0 {
      return s, ""
   }
   // 判断是否保留 sep 返回
   if cutc {
      return s[:i], s[i+1:]
   }
   return s[:i], s[i:]
}

````







