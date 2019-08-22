title: Golang优雅重启
date: 2018-11-23 17:02:28
tags: [golang]
---
本文翻译自[博客](https://grisha.org/blog/2014/06/03/graceful-restart-in-golang/)
2015年4月更新: [Florian von Bock](https://github.com/fvbock) 已将本章内容写成了一个非常好的go包[endless](https://github.com/fvbock/endless).

如果您有Golang HTTP服务，可能需要重新启动它以升级二进制文件或更改某些配置。如果您（像我一样）因为网络服务器需要小心对待而认为优雅地重新启动是理所当然的，您可能会发现这个配方非常方便，因为使用Golang你需要自己动手完成。

实际上有两个问题需要处理。第一个是UNIX方面的优雅重启，也就是，一个进程重启它自己而不用关闭监听的socket的机制。第二个问题是确保所有进行中的请求被合理的完成或者超时。

## 重启而不用关闭socket
* fork一个继承监听socket的进程。
* 子进程执行初始化并开始接收在这个socket上的连接。
* 然后马上，子进程发送一个信号(signal)到父进程，引起父进程停止接收连接并终止进程。

## Fork新进程

有超过一种使用Golang库fork进程的方法，但对我们的特定情况，[exec.Command](http://golang.org/pkg/os/exec/#Command)是正确的选择。这是因为这个函数返回的[Cmd结构](http://golang.org/pkg/os/exec/#Cmd)有一个**ExtraFiles**成员，它可以指定新进程继承的打开文件(除了stdin/err/out)。

它看起来像这样：
``` golang
file := netListener.File() // this returns a Dup()
path := "/path/to/executable"
args := []string{"-graceful"}

cmd := exec.Command(path, args...)
cmd.Stdout = os.Stdout
cmd.Stderr = os.Stderr
cmd.ExtraFiles = []*os.File{file}

err := cmd.Start()
if err != nil {
    log.Fatalf("gracefulRestart: Failed to launch, error: %v", err)
}
```
上面的代码**netListener**是一个指向监听HTTP请求的[net.Listener](http://golang.org/pkg/net/#Listener)的指针。如果您是在升级，**path**变量应该包含新的可执行文件的路径（也可能和当前正在运行的可执行文件路径相同）。

上面代码的重要观点是**netListener.File()**返回一个[dup(2)](http://pubs.opengroup.org/onlinepubs/009695399/functions/dup.html)的文件描述符。复制的文件描述符不会设置[FD_CLOEXEC标记](http://pubs.opengroup.org/onlinepubs/009695399/functions/fcntl.html),此标记会引起此文件在子进程中关闭（不是我们想要的）。

您可能会遇到通过命令行参数将继承的文件描述符编号传递给子进程的示例，但**ExtraFiles**的实现方式使其没有必要。文档指出“如果非零，则条目i变为文件描述符3+i。”这意味着在上面的代码片段中，子进程中的继承文件描述符将始终为3，因此不需要明确地传递它。

最后，**args**数组应该包含一个**-graceful**选项：您的程序将需要某种方式子进程，这是优雅重启的一部分，子进程应该重用此socket而不是试图打开一个新的socket。另外一个实现的方法是通过环境变量。

## 子进程初始化
这是程序启动的一部分：
``` golang
    server := &http.Server{Addr: "0.0.0.0:8888"}

    var gracefulChild bool
    var l net.Listever
    var err error

    flag.BoolVar(&gracefulChild, "graceful", false, "listen on fd open 3 (internal use only)")

    if gracefulChild {
        log.Print("main: Listening to existing file descriptor 3.")
        f := os.NewFile(3, "")
        l, err = net.FileListener(f)
    } else {
        log.Print("main: Listening on a new file descriptor.")
        l, err = net.Listen("tcp", server.Addr)
    }
```

## 向父进程发信号通知其停止
这个时候，我们已经准备好接收请求，但在这之前，我们需要告知父进程停止接收请求并推出，就像下面这样：
``` golang
if gracefulChild {
    parent := syscall.Getppid()
    log.Printf("main: Killing parent pid: %v", parent)
    syscall.Kill(parent, syscall.SIGTERM)
}

server.Serve(l)
```

## 进行中的请求的完成/超时
对这部分来说，我们需要用[sync.WaitGroup](http://golang.org/pkg/sync/#WaitGroup)来跟踪已打开的连接。我们需要接收每个请求时增加wait group并在连接关闭时减小它。
``` golang
var httpWg sync.WaitGroup
```
猛一看，Golang标准的http包不支持任何Accept()和Close()的hook，但是接口魔法可以解决这个问题。（非常感谢[Jeff R. Allen](http://nella.org/jra/)的[这篇文章](http://blog.nella.org/zero-downtime-upgrades-of-tcp-servers-in-go/)）

下面是一个listener在每一个Accept()函数增加wait group的示例。首先，我们创建**net.Listener**的“子类”。（您将看到为什么我们需要下面的**stop**和**stopped**)。
``` golang
type gracefulListener struct {
    net.Listener
    stop    chan error
    stopped bool
}
```

然后，我们“覆盖”Accept方法。（现在先别管**gracefulConn**，我们将很快介绍它）。
``` golang
func (gl *gracefulListener) Accept() (c net.Conn, err error) {
    c, err = gl.Listener.Accept()
    if err != nil {
        return
    }

    c = gracefulConn{Conn: c}

    httpWg.Add(1)
    return
}
```

我们也需要一个“构造函数”：
``` golang
func newGracefulListener(l net.Listener) (gl *gracefulListener) {
    gl = &gracefulListener{Listener: l, stop: make(chan error)}
    go func() {
        _ = <-gl.stop
        gl.stopped = true
        gl.stop <- gl.Listener.Close()
    }()
    return
}
```

上面的函数开始了一个goroutine的原因是它不能在上面的**Accept()**中完成，因为它会block在**gl.Listener.Accept**上。这个goroutine会通过关闭文件描述符来unblock它。

我们的**Close()**方法只需要简单的发送一个**nil**到上面goroutine的stop通道来完成剩余的工作。

``` golang
func (gl *gracefulListener) Close() error {
    if gl.stopped {
        return syscall.EINVAL
    }
    gl.stop <- nil
    return <-gl.stop
}
```
最后,下面这个方便函数从net.TCPListener解压文件描述符。
``` golang
func (gl *gracefulListener) File() *os.File {
    tl := gl.Listener.(*net.TCPListener)
    fl, _ := tl.File()
    return fl
}
```
当然，我们也需要一个变种的**net.Conn**在**Close()**中减少wait group。
``` golang
type gracefulConn struct {
    net.Conn
}

func (w gracefulConn) Close() error {
    httpWg.Done()
    return w.Conn.Close()
}
```
为了使用优雅版本的Listener，我们只需要修改**server.Serve(l)**这一行为：
``` golang
netListener = newGracefulListener(l)
server.Serve(netListener)
```

并且还有一件事。您应该应该避免挂起客户端故意不关闭的连接(或者是本周内不关闭的连接)。最好像下面这样创建连接。
``` golang
server := &http.Server{
        Addr:           "0.0.0.0:8888",
        ReadTimeout:    10 * time.Second,
        WriteTimeout:   10 * time.Second,
        MaxHeaderBytes: 1 << 16}
```

