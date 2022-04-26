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

// URL 
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

// Values 用于操作 URL 中 query 部分
type Values map[string][]string
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

函数 `*URL.EscapedFragment() string` 编码 Fragment

基本上和 EscapedPath() 相同

````
func (u *URL) EscapedFragment() string {
    // 使用 `validEncoded()` 判断 u.RawFragment 是否规范
    if u.RawFragment != "" && validEncoded(u.RawFragment, encodeFragment) {
        f, err := unescape(u.RawFragment, encodeFragment)
        if err == nil && f == u.Fragment {
            return u.RawFragment
        }
    }
    // 其他情况直接编码 u.Fragment
    return escape(u.Fragment, encodeFragment)
}
````

---


函数 `*URL.String() string` 编码后输出完整的url

````

func (u *URL) String() string {
    var buf strings.Builder
    // 写入 Scheme
    if u.Scheme != "" {
        buf.WriteString(u.Scheme)
        buf.WriteByte(':')
    }
    // 存在 Opaque 写入 Opaque
    if u.Opaque != "" {
        buf.WriteString(u.Opaque)
    } else {
        if u.Scheme != "" || u.Host != "" || u.User != nil {
            if u.Host != "" || u.Path != "" || u.User != nil {
                buf.WriteString("//")
            }
            if ui := u.User; ui != nil {
                buf.WriteString(ui.String())
                buf.WriteByte('@')
            }
            // 这里编码后写入 Host
            if h := u.Host; h != "" {
                buf.WriteString(escape(h, encodeHost))
            }
		}
        // 编码后写入 path
        path := u.EscapedPath()
        if path != "" && path[0] != '/' && u.Host != "" {
            buf.WriteByte('/')
        }
        if buf.Len() == 0 {
            // RFC 3986 §4.2
            // A path segment that contains a colon character (e.g., "this:that")
            // cannot be used as the first segment of a relative-path reference, as
            // it would be mistaken for a scheme name. Such a segment must be
            // preceded by a dot-segment (e.g., "./this:that") to make a relative-
            // path reference.
            if i := strings.IndexByte(path, ':'); i > -1 && strings.IndexByte(path[:i], '/') == -1 {
                buf.WriteString("./")
            }
        }
		buf.WriteString(path)
	}
	// 写入 RawQuery
	if u.ForceQuery || u.RawQuery != "" {
		buf.WriteByte('?')
		buf.WriteString(u.RawQuery)
	}
	if u.Fragment != "" {
		buf.WriteByte('#')
		buf.WriteString(u.EscapedFragment())
	}
	return buf.String()
}

````

---

函数 `*URL.Redacted() string` 编码后输出完整的url, 但是会将密码改为 xxxxx

````
func (u *URL) Redacted() string {
    if u == nil {
        return ""
    }

	ru := *u
	// 如果 password 存在设置为 xxxxx
	if _, has := ru.User.Password(); has {
		ru.User = UserPassword(ru.User.Username(), "xxxxx")
	}
	// 字符串的形式输出
	return ru.String()
}
````

---

函数 `url.ParseQuery(query string) string` 解析 query 返回 Values 结构体

query 的键值对会被解码后存入 Values

````

func ParseQuery(query string) (Values, error) {
   // 创建一个 Values 解析 query 并且返回 Values
    m := make(Values)
    err := parseQuery(m, query)
    return m, err
}
````

---

函数 `Values.Encode() string` 编码 Values 并以字符串的形式返回

````
func (v Values) Encode() string {
	if v == nil {
		return ""
	}
	var buf strings.Builder
	// 创建一个 切片，存入 v 中的所有 key
	keys := make([]string, 0, len(v))
	for k := range v {
		keys = append(keys, k)
	}
	// 排序,这里暂时不清楚为什么要排序
	sort.Strings(keys)
	for _, k := range keys {
		vs := v[k]
		// 编码 k
		keyEscaped := QueryEscape(k)
		for _, v := range vs {
		    // 写入 & 作为分隔符，第一个不写
			if buf.Len() > 0 {
				buf.WriteByte('&')
			}
			// 写入 key=value
			buf.WriteString(keyEscaped)
			buf.WriteByte('=')
			buf.WriteString(QueryEscape(v))
		}
	}
	return buf.String()
}
````

---

函数 `*URL.IsAbs() bool` 判断是 URL 否存在 Scheme

````
func (u *URL) IsAbs() bool {
    return u.Scheme != ""
}
````

---

函数 `*URL.Parse() (*URL, error)` 接收一个字符串的 url 解析后将存在值的元素覆盖 u 中的值，返回一个新的 *URL

````
func (u *URL) Parse(ref string) (*URL, error) {
    // 使用 url.Parse 解析
    refurl, err := Parse(ref)
    if err != nil {
        return nil, err
    }
    // 使用 refurl 的元素，替换 u 中的元素的值，返回一个新的 *URL
	return u.ResolveReference(refurl), nil
}
````

---

函数 `*URL.ResolveReference() *URL` 接收一个 *URL 将存在值的元素覆盖 u 中的值，返回一个新的 *URL

该方法是 `*URL.Parse()` 的底层实现

````
func (u *URL) ResolveReference(ref *URL) *URL {
    url := *ref
	if ref.Scheme == "" {
		url.Scheme = u.Scheme
	}
	if ref.Scheme != "" || ref.Host != "" || ref.User != nil {
		// The "absoluteURI" or "net_path" cases.
		// We can ignore the error from setPath since we know we provided a
		// validly-escaped path.
		url.setPath(resolvePath(ref.EscapedPath(), ""))
		return &url
	}
	if ref.Opaque != "" {
		url.User = nil
		url.Host = ""
		url.Path = ""
		return &url
	}
	if ref.Path == "" && ref.RawQuery == "" {
		url.RawQuery = u.RawQuery
		if ref.Fragment == "" {
			url.Fragment = u.Fragment
			url.RawFragment = u.RawFragment
		}
	}
	// The "abs_path" or "rel_path" cases.
	url.Host = u.Host
	url.User = u.User
	url.setPath(resolvePath(u.EscapedPath(), ref.EscapedPath()))
	return &url
}
````

---

函数 `*URL.Query() Values` 解析 u.RawQuery 返回 Values 结构体

底层使用的是 url.ParseQuery()

````
func (u *URL) Query() Values {
    v, _ := ParseQuery(u.RawQuery)
    return v
}
````

---

函数 `*URL.RequestURI() string` 返回 path?query 或者 opaque?query 的形式

这个不知道有什么用

````

func (u *URL) RequestURI() string {
	result := u.Opaque
	// 不存在 Opaque 使用 编码后的 path
    if result == "" {
        result = u.EscapedPath()
        if result == "" {
            result = "/"
        }
    } else {
        // 如果是以 // 开头 则改为类似 https://path 的形式
        // 不知道什么情况会出现这个，Opaque 应该是没有 // 才对
        if strings.HasPrefix(result, "//") {
            result = u.Scheme + ":" + result
        }
    }
    // 这里加入query
    if u.ForceQuery || u.RawQuery != "" {
        result += "?" + u.RawQuery
    }
    return result
}
````

---

函数 `*URL.Hostname() string、*URL.Port() string` 分别用来获取Hostname和端口的

````
func (u *URL) Hostname() string {
	host, _ := splitHostPort(u.Host)
	return host
}

func (u *URL) Port() string {
	_, port := splitHostPort(u.Host)
	return port
}
````

---

函数 `*URL.MarshalBinary() (text []byte, err error)、*URL.UnmarshalBinary(text []byte) error` 前者将 *URL 使用字节数组的形式
输出，后者接收字节数组，覆盖到 *URL 上

````

func (u *URL) MarshalBinary() (text []byte, err error) {
	return []byte(u.String()), nil
}

func (u *URL) UnmarshalBinary(text []byte) error {
	u1, err := Parse(string(text))
	if err != nil {
		return err
	}
	*u = *u1
	return nil
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

---

函数 `url.parseQuery(m Values, query string)` 解析 query 解码后保存到 m 中

````

func parseQuery(m Values, query string) (err error) {
    for query != "" {
        key := query
        // 查找 & 或者 ; 第一次出现的索引位置
        if i := strings.IndexAny(key, "&;"); i >= 0 {
            // 分离出 第一个键值对 和 剩余的query
            key, query = key[:i], key[i+1:]
        } else {
            query = ""
        }
        if key == "" {
            continue
        }
        value := ""
        // 如果有 = 则分离出 key 和 value
        if i := strings.Index(key, "="); i >= 0 {
            key, value = key[:i], key[i+1:]
        }
        // 解码 key
        key, err1 := QueryUnescape(key)
        if err1 != nil {
            if err == nil {
                err = err1
            }
            continue
        }
        // 解码 value
        value, err1 = QueryUnescape(value)
        if err1 != nil {
            if err == nil {
                err = err1
            }
            continue
        }
        // 加入 map
        m[key] = append(m[key], value)
    }
    return err
}

````







