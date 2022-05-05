
````
type Request struct {
    // 指定发送的HTTP请求的方法（GET、POST、PUT等），Go的HTTP客户端不支持使用CONNECT方法发送请求
    Method string
    
    // URL 指定被请求的URI（对于服务器请求）或要访问的URL（对于客户端请求，就是要连接的服务器地址）
    // 参考 net.url
    URL *url.URL
    
    // 请求的协议版本（主版本号.次版本号）
    Proto      string // "HTTP/1.0"
    ProtoMajor int    // 1
    ProtoMinor int    // 0
    
    // 请求头(为请求报文添加了一些附加信息)，不区分大小写，对于客户端请求，某些头（如内容长度和连接）会在需要时自动写入
    // 是一个 map[string][]string 类型，具体参考 net.http.header
    Header Header
    
    // 请求体
    // 客户端的请求，GET 请求没有请求体
    // 服务端的请求，请求体始终非nil，否则会返回 EOF
    Body io.ReadCloser
    
    // 获取请求体的副本
    GetBody func() (io.ReadCloser, error)
    
    // 请求Body的大小（字节数）
    ContentLength int64
    
    // 列出从最外层到最内层的传输编码。空列表表示“身份”编码，通常可以忽略；在发送和接收请求时，根据需要自动添加和删除分块编码。
    TransferEncoding []string
    
    // 在回复了此次请求后结束连接。对服务端来说就是回复了这个 request ，对客户端来说就是收到了 response
    // 且 对于服务端是Handlers 会自动调用Close, 对客户端，如果设置长连接(Transport.DisableKeepAlives=false)，就不会关闭。没设置就关闭
    Close bool
    
    // 主机名称
    Host string
    
    // 表单数据
    // 为 map[string][]string 具体参考 url.Values
    Form url.Values
    
    // PostForm 包含来自PATCH、POST或PUT body参数的解析表单数据, 此字段仅在调用ParseForm后可用。HTTP客户机忽略PostForm，而是使用Body。
    PostForm url.Values
    
    // MultipartForm 是经过解析的多部件 表单，包括文件上传. 此字段仅在解析表单即 调用ParseMultipartForm后可用。 HTTP客户机忽略MultipartForm，而是使用Body。
    MultipartForm *multipart.Form
    
    // Trailer 指定在 当body部分发送完成之后 发送的附加头
    Trailer Header
    
    // RemoteAddr 允许HTTP服务器和其他软件记录发送请求的网络地址，通常用于日志记录。
    // 此字段不是由ReadRequest填写的，并且没有定义的格式。此包中的HTTP服务器在调用处理程序之前将RemoteAddr设置为“IP:port”地址。 HTTP客户端将忽略此字段。
    RemoteAddr string
    
    // RequestURI 是客户端发送到服务器的请求行（RFC 7230，第3.1.1节）的未修改的请求目标。通常应该改用URL字段。在HTTP客户端请求中设置此字段是错误的。
    RequestURI string
    
    // TLS 允许HTTP服务器和其他软件记录有关接收请求的TLS连接的信息。此字段不是由ReadRequest填写的。HTTP客户端将忽略此字段
    TLS *tls.ConnectionState
    
    // Cancel 是一个可选通道，它的关闭表示客户端请求应被视为已取消
    // 不推荐：改为使用NewRequestWithContext设置请求的上下文
    Cancel <-chan struct{}
    
    // Response是导致创建此请求的重定向响应。此字段仅在客户端重定向期间填充
    Response *Response
    
    // 请求的上下文。（修改时，通过使用WithContext复制整个请求来修改它）
    // 对于传出的客户机请求，上下文控制请求及其响应的整个生存期：获取连接、发送请求以及读取响应头和主体
    ctx context.Context
}

````