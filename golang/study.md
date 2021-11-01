---
title: golang time 序列化反序列化
date: 2019-07-18 22:24:01
tags: [golang, function]
category: [golang]
---
1. 编译与反编译指令
源码main.go：
``` golang
package main

func main() {
	var a = "hello"
	var b=[]byte(a)
	print(b)
}
```
编译:
``` bash
go tool compile -S main.go|grep "main.go:5"
```

2. go tool objdump 寻找make是实现
源码main.go:
``` golang
package main

import "fmt"

func main() {
	a:=make([]int,10000)
	fmt.Println(len(a))

	b:=make(map[int]string,10)
	fmt.Println(len(b))

	c:=make(chan int,10)
	fmt.Println(len(c))
}
```
查找:
```bash
go build -gcflags="-N -l" main.go
go tool objdump main|egrep  "main.go:9|main.go:6|main.go:12"
```