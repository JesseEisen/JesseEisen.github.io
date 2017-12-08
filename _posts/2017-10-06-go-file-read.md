---
layout: post
title: Go File Read
date: 2017-10-06 18:00:00
description: go read relate
---

最近在写一个Go的文件搜索程序，涉及到一些文件的读写。所以趁此机会来总结一些。根据不同的场景选择最合适的读写函数。

 <hr>

#### 创建文件

创建文件可以使用`os.Create`来创建，如果这个文件存在，就截断这个文件。如果不存在。则以权限`0666`和`O_RDWR`来创建新的文件。
其他的也可以使用下面的`OpenFile`来创建文件。 


#### 打开文件

`os` 包里提供了一个`Open`函数，函数签名为：

```
func Open(name string) (*File, error)
```

函数返回一个`*File`， 而`File` 实现了`Read`和`Write`这两个方法， 所以Open的返回值可以直接传入以`io.Reader`和`io.Writer`为参数的函数中。
不过于对于`os.Open`而言，返回的File只能用于读（`Os.O_RDONLY`）。 因此在需要写的情况下，直接`Open`这个函数就不太适合。 可以使用如下：

```
func OpenFile(name string, flag int, perm FileMode) (*File, error)
```

函数中的`flag`和`perm` 这两个参数，可以参考linux下的`open(2)`,通过这两个指定文件打开的权限和读写设置.


#### 读写文件

在读写文件的方面,可以有多种方式,可以希望读出来是`[]byte` 可以是`string`. 可以按行读, 也可以一次性读完整个文件. 因此我们可以使用不同的包提供的相关函数.
这些包有`io`,`ioutil`,`bufio`以及`File`本身的一些方法.


<hr>

**os**  

这个包里面包含了`File`，这个type是实现了几个有关文件的读写方法。 比如`Read`,`ReadAt`,`Readdir`等读函数，对应的也有`Write`,`WriteAt`,`WriteString`这些写函数。所以如果使用`os.Open`打开的的文件，可以直接使用自身的方法来进行相应的读取。

```
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

```
func (f *File)ReadAt(b []byte, off int64)(n int, err error)
```

使用方法和上面的Read类似。

对于`Readdir`方法而言，通过`os.Open`打开一个目录文件，通过设置读取的目录数来读取当前目录下的内容。

```
func (f *File) Readdir(n int) ([]FileInfo, error)
```

这边如果`n`是小于等于0的话，会读取当前目录下所有的内容。返回的`FileInfo`是一个interface。包含了获取文件一些属性的方法。

```
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

```
func NewReader(rd io.Reader) *Reader
```

有了`*Reader`的变量后，可以通过接口来实现不少东西。我们可以查看当前可以读取的buffer可以读取的byte数。

```
func (b *Reader) Buffered() int
```

不过这个函数在buffer还没有被读取的时候会返回0，读取后，再次调用后会用总长度减去已经读取的长度。我们可以用一个简短的程序测试一下：

```
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

