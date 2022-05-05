# io


---

## 主要的常量、变量

````
// Seek whence values.
const (
    SeekStart   = 0 // 基于文件开始位置
    SeekCurrent = 1 // 基于当前偏移量
    SeekEnd     = 2 // 基于文件结束位置
)
````

````
// 表示文件结束的错误
// io.EOF 是在没有任何可读取的内容时触发，比如某文件Reader对象，文件本身为空.
// 或者读取若干次后，文件指针指向了末尾，调用Read都会触发EOF
var EOF = errors.New("EOF")
````

---

## 主要的接口

这里省略了一些组合的接口

接口 `Reader`

定义：
````
// 方法读取数据写入到字节数组 p 中，由于 p 是有大小的，所以一次至多读取 len(p) 个字节
// 方法返回读取的数据字节数 n(0 <= n <= len(p))，以及读取过程中遇到的 error
// 即使一次调用读取到的数据小于 len(p)，也可能会占用整个字节数组 p 作为暂存空间
// 如果数据源的数据量小于 len(p) 个字节，方法只会读取当前可用数据，不会等待更多数据的到来
type Reader interface {
    Read(p []byte) (n int, err error)
}
````

何时结束：

* 在成功读取了 n（n>0）个字节后，如果产生了 error 或者 读到文件末尾 （end-of-file），本次调用必须要返回读取的字节数 n
但对于err 的值，可以选择在本次直接返回 err（err!=nil），或者在下次调用的时候再返回 err (n=0, err!=nil)。
常见的一个例子就是，读取到n个字节后到达文件末尾（EOF），此时可以返回 err=EOF 或者 err=nil，下次调用返回 n=0,err=EOF
* 调用者需要注意，每次调用后，如果 n>0，应该先处理数据，再考虑 err 是否为 nil。因为上一点已经指出，如果读取到 n>0 个字节后遇到 error，
会同时返回 n>0 和 err!=nil，此时就需要先处理数据再考虑 err。


---

接口 `Writer`

定义：
````
// 将 len(p) 个字节从 p 中写入到基本数据流中
type Writer interface {
    Write(p []byte) (n int, err error)
}
````

何时结束：

* 如果 n<len(p)，方法必须返回 err!=nil
* 方法一定不能修改字节数组 p，即使是临时修改也不被允许

---

接口 `Closer`

定义：
````
// 用于关闭数据流
// 文件 (os.File)、归档（压缩包）、数据库连接、Socket 等需要手动关闭的资源都实现了 Closer 接口
type Closer interface {
    Close() error
}
````

接口 `Seeker`

定义：
````
// 用于指定下次读取或者写入时的偏移量
// 入参: 计算新偏移量的起始值 whence， 基于whence的偏移量offset
// 返回值: 基于 whence 和 offset 计算后新的偏移量值，以及可能产生的错误
type Seeker interface {
    Seek(offset int64, whence int) (int64, error)
}
````

---

接口 `ReaderFrom`

定义：
````
// 一次性读取r的数据
// 从 r 中读取数据，直到 EOF 或发生错误。其返回值 n 为读取的字节数。除 io.EOF 之外，在读取过程中遇到的任何错误也将被返回。
// 如果 ReaderFrom 可用，Copy 函数就会使用它。
// ReadFrom 方法不会返回 err == EOF
type ReaderFrom interface {
    ReadFrom(r Reader) (n int64, err error)
}
````

该接口可以认为是对 `io.Reader` 的扩展，实现该接口有两种思路：

* 先获取文件的大小（File 的 Stat 方法），之后定义一个该大小的 []byte，通过 Read 一次性读取
* 定义一个小的 []byte，不断的调用 Read 方法直到遇到 EOF，将所有读取到的 []byte 连接到一起

参考 `bufio.Writer 和 bytes.Buffer` 对 ReaderFrom 的实现

---

接口 `WriterTo`

定义：
````
// 一次性将数据写到某个地方去
// 将数据写入 w 中，直到没有数据可写或发生错误。其返回值 n 为写入的字节数。 在写入过程中遇到的任何错误也将被返回。
// 如果 WriterTo 可用，Copy 函数就会使用它
type WriterTo interface {
    WriteTo(w Writer) (n int64, err error)
}
````

---

接口 `ReaderAt`

定义：
````
// 从偏移量off处开始，将len(p)个字节读取到p 中，它返回读取的字节数 n（0 <= n <= len(p)）以及任何遇到的错误。
// 当 ReadAt 返回的 n < len(p) 时，它就会返回一个 非nil 的错误来解释 为什么没有返回更多的字节。在这一点上，ReadAt 比 Read 更严格
// 即使 ReadAt 返回的 n < len(p)，它也会在调用过程中使用 p 的全部作为暂存空间。若一些数据可用但不到 len(p) 字节，ReadAt 就会阻塞直到所有数据都可用或产生一个错误。 
// 在这一点上 ReadAt 不同于 Read。
// 若 n = len(p) 个字节在输入源的的结尾处由 ReadAt 返回，那么这时 err == EOF 或者 err == nil。
// 
type ReaderAt interface {
    ReadAt(p []byte, off int64) (n int, err error)
}
````

---

接口 `WriterAt`

定义：
````
// 从 p 中将 len(p) 个字节写入到偏移量 off 处的基本数据流中。它返回从 p 中被写入的字节数 n（0 <= n <= len(p)）以及任何遇到的引起写入提前停止的错误。
// 若 WriteAt 返回的 n < len(p)，它就必须返回一个 非nil 的错误。
type WriterAt interface {
    WriteAt(p []byte, off int64) (n int, err error)
}
````






