# url

url 格式

一个标准的 URI 格式为：`[scheme:]scheme-specific-part`，如 `https://github.com/duanjw/note` 、
`mailto:474280116@qq.com`

URI 可以细分为为`不透明的 (opaque) `和`分层的 (hierarchical) `两类：
* `opaque` 指 `scheme-specific-part` 不以 / 开头，是一个整体。
呈 `[scheme]:opaque[?query][#fragment]` 的形式。如：mailto:474280116@qq.com, opaque 必须是绝对的。
* `hierarchical` 指 `scheme-specific-part` 以 / 开头且可以划分为好几部分。
呈 `[scheme:][//[userinfo@]host[:port]]path[?query][#fragment]` 的形式。如：
https://github.com/duanjw/note, hierarchical 可以是绝对的，也可以是相对的，如：`../../static/verify.js`

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









