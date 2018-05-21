---
layout: post
title: Go File Read
date: 2017-10-06 18:00:00
description: go read relate
---

最近在写一个 Go 的文件搜索程序，涉及到一些文件的读写。所以趁此机会来总结一些。根据不同的场景选择最合适的读写函数。

 <hr>

#### 创建文件

创建文件可以使用`os.Create`来创建，如果这个文件存在，就截断这个文件。如果不存在,则会通过相应的读写权限和打开的用途来创建新的文件。
其他的也可以使用下面的`OpenFile`来创建文件。


#### 打开文件

`os` 包里提供了一个`Open`函数，函数签名为：

```go
func Open(name string) (*File, error)
```

函数返回一个`*File`， 而`File` 实现了`Read`和`Write`这两个方法， 所以 Open 的返回值可以直接传入以`io.Reader`和`io.Writer`为参数的函数中。
不过于对于`os.Open`而言，返回的 File 只能用于读（`Os.O_RDONLY`）。 因此在需要写的情况下，直接`Open`这个函数就不太适合。 可以使用如下：

```go
func OpenFile(name string, flag int, perm FileMode) (*File, error)
```

函数中的`flag`和`perm` 这两个参数，可以参考 linux 下的`open(2)`, 通过这两个指定文件打开的权限和读写设置。


#### 读写文件

在读写文件的方面，可以有多种方式，可以希望读出来是`[]byte` 可以是`string`. 可以按行读，也可以一次性读完整个文件。因此我们可以使用不同的包提供的相关函数。
这些包有`io`,`ioutil`,`bufio`以及`File`本身的一些方法。


<hr>

**os**

这个包里面包含了`File`，这个 type 是实现了几个有关文件的读写方法。 比如`Read`,`ReadAt`,`Readdir`等读函数，对应的也有`Write`,`WriteAt`,`WriteString`这些写函数。所以如果使用`os.Open`打开的的文件，可以直接使用自身的方法来进行相应的读取。

```go
f, err := os.Open("test.txt")
...... //err check

info, err := os.Stat("test.txt")
...... //err check

fsize := info.Size()

data := make([]byte, fsize)

_, err := f.Read(data)
..... //err check
```

这个方法的返回值是读取的字节数。 这个读取是可控制的，可以一次性读入整个文件，也可以只读入指定长度的内容，即指定`[]byte`的大小即可。比较灵活一点。同时可以使用`ReadAt`来做一个偏移后再读取。

```go
func (f *File)ReadAt(b []byte, off int64)(n int, err error)
```

使用方法和上面的 Read 类似。

对于`Readdir`方法而言，通过`os.Open`打开一个目录文件，通过设置读取的目录数来读取当前目录下的内容。

```go
func (f *File) Readdir(n int) ([]FileInfo, error)
```

这边如果`n`是小于等于 0 的话，会读取当前目录下所有的内容。返回的`FileInfo`是一个 interface。包含了获取文件一些属性的方法。

```go
type FileInfo interface {
    Name() string       // base name of the file
    Size() int64        // length in bytes for regular files; system-dependent for others
    Mode() FileMode     // file mode bits
    ModTime() time.Time // modification time
    IsDir() bool        // abbreviation for Mode().IsDir()
    Sys() interface{}   // underlying data source (can return nil)
}
```

如果只想获取当前的目录下的文件的名称，可以直接使用，`Readdirnames`方法即可。可以不用从`FileInfo`中调用相关的方法去转换了。这个是需要结合实际需求的。

<hr>

**bufio**

这个包里面提供了一些读写相关的内容。先来看看读相关的内容。 在`bufio`中有一个`Reader`的结构体。这个结构体本身实现了一些方法。这些方法的返回值基本上是
`[]byte`或者`rune`这类的。

首先需要创建一个`Reader`的变量， 上面提到过，在`os.Open`打开的文件的返回是`*File`类型的变量实现了`io.Reader`的接口，所以我们可以直接调用。

```go
func NewReader(rd io.Reader) *Reader
```

有了`*Reader`的变量后，可以通过接口来实现不少东西。我们可以查看当前可以读取的 buffer 可以读取的 byte 数。

```go
func (b *Reader) Buffered() int
```

不过这个函数在 buffer 还没有被读取的时候会返回 0，读取后，再次调用后会用总长度减去已经读取的长度。我们可以用一个简短的程序测试一下：

```go
func main() {
    br := bufio.NewReader(f)

    s := "hello, world!"  //length 13
    bs := bufio.NewReader(strings.NewReader(s))

    ns :=  bs.Buffered()  // print 0
    fmt.Println("before read string: ", ns)

    _, err = bs.ReadString(byte(','))
    if err != nil {
        log.Fatalln(err)
    }

    ns = bs.Buffered() // print 7
    fmt.Println("after read string: ", ns)
}
```

和读相关的函数如下：

+ Read
+ ReadByte
+ ReadBytes
+ ReadLine
+ ReadRune
+ ReadSlice
+ ReadString

可以根据实际需要来选择需要的方法， 这里面有一些方法是根据`delim`来作为一个读的分段，比如：

```go
func (b *Reader) ReadBytes(delim byte) ([]byte, error)
```

这个方法在失败的时候，会返回已经读到的内容。这边有一个需要注意的，返回值`[]byte`的大小，是在创建了`Reader`后会自动分配一个大小。这个大小也许会不够
比如：在读取一个二进制文件时，将`delim`设置为`\n`. 那么有时会把整个文件都读过去。所以会报错，byte 的大小不够存储。 因此这边可以使用`NewReaderSize`来
设置一个 size。

```go
func NewReaderSize(rd io.Reader, size int) *Reader
```

需要注意的是，如果这个`io.Reader`已经是一个分配了足够大空间的话，就会返回大的那个。

还有一个值得一提的方法，`Reset`。 我们在读取 buffer 中的内容时，可以使用`Reset`来丢弃所有的已经 buffered 的内容，相当于从头再来读一遍。

同时，bufio 还提供了两个 undo 函数，`UnreadByte`和`UnreadRune`. 第二个函数需要注意的是，如果最后一次读的操作不是`ReadRune`的话，这个函数就将返回一个错误。

<hr>

**io**

`io`包里面也提供了一些有关的读函数。比如 `ReadAtLeast`. 意思就是至少读这么多个字符。函数签名如下：

```go
func ReadAtLeast(r Reader, buf []byte, min int) (n int, err error)
```

这里的 buf 的大小和 min 的大小是有关系的，如果 buf 的长度比 min 的长度要小，那么直接报错，如果一样的大，也会返回一个错误`EOF`。另一个是`ReadFull`这个函数使用来读取`len(buf)`长度的内容，如果读取的内容比`len(buf)`的长度小，那么则返回一个`unexpected EOF`。

io 包里面还提供了一些 interface，这些 iterface 类型可以被其他的包实现。比如：strings 包下面的 Reader。 实现了`io.Reader`,`io.ReaderAt`等接口，所以可以直接调用相应的方法。如下示例：

```go
reader := strings.NewReader("golang")
p := make([]byte, 6)
n, err := reader.ReadAt(p, 2)
if err != nil {
  log.Fatalln(err)
}
fmt.Printf("%s, %d\n", p, n)
```
