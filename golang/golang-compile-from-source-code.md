title: 从源码编译golang 1.12
tags:
  - golang
category: [golang]
date: 2019-04-13 00:39:00
---
1. 下载源码
``` bash
$ mkdir studygo && cd studygo
$ git clone https://github.com/golang/go.git
```

2. 切换到1.4版本(需要先生成1.4版本的go，然后用1.4版本的去编译高版本的)
``` bash
$ cd go
$ git checkout release-branch.go1.4
$ cd ..;cp -r go go1.4
$ cd go/src
$ ./make.bash
---
Installed Go for linux/amd64 in /home/arthur/gostudy/go1.4
Installed commands in /home/arthur/gostudy/go1.4/bin
```

3. 设置环境变量GOROOT_BOOTSTRAP，并编译go1.12.4
``` bash
$ export GOROOT_BOOTSTRAP=$HOME/studygo/go1.4
$ git checkout release-branch.go1.12
$ cd ..;cp -r go go1.2
$ cd go1.2/src
$ ./make.bash
Building Go cmd/dist using /home/arthur/gostudy/go1.4.
Building Go toolchain1 using /home/arthur/gostudy/go1.4.
Building Go bootstrap cmd/go (go_bootstrap) using Go toolchain1.
Building Go toolchain2 using go_bootstrap and Go toolchain1.
Building Go toolchain3 using go_bootstrap and Go toolchain2.
Building packages and commands for linux/amd64.
---
Installed Go for linux/amd64 in /home/arthur/gostudy/go1.12
Installed commands in /home/arthur/gostudy/go1.12/bin
```
4. 可通过`go`命令验证
``` bash
$ /home/arthur/gostudy/go1.12/bin/go version
go version go1.12.4 linux/amd64
```