---
title: golang搭建静态文件服务器
date: 2018-09-13 22:34:32
tags: [golang, fileserver]
category: [golang]
---
公司有一些公开的文件需要有地方存放用于下载，想到搭建一个简单的文件服务器，文件通过scp上传到目录，点击可以查看内容或下载。
PS:想通过浏览器的http访问，不用ftp
发现golang有现成的fileserver代码，可以拿来直接用，正是我要的效果，关键代码就2-3行。
代码如下:
``` go
/*
https://gist.github.com/paulmach/7271283
Serve is a very simple static file server in go
Usage:
	-p="8100": port to serve on
	-d=".":    the directory of static files to host
Navigating to http://localhost:8100 will display the index.html or directory
listing file.
*/
package main

import (
	"flag"
	"log"
	"net/http"
)

func main() {
	port := flag.String("p", "8100", "port to serve on")
	directory := flag.String("d", ".", "the directory of static file to host")
	flag.Parse()
	fileServer := http.FileServer(http.Dir(*directory))
	http.Handle("/", fileServer)

	log.Printf("Serving %s on HTTP port: %s\n", *directory, *port)
	log.Fatal(http.ListenAndServe(":"+*port, nil))
}
```
重点函数就是**http.FileServer(http.Dir(*directory))**，函数解释见https://golang.org/pkg/net/http/#FileServer。
``` go
func FileServer(root FileSystem) Handler
```

> FileServer returns a handler that serves HTTP requests with the contents of the file system rooted at root.
> To use the operating system's file system implementation, use http.Dir:

``` go
http.Handle("/", http.FileServer(http.Dir("/tmp")))
```

> As a special case, the returned file server redirects any request ending in "/index.html" to the same path, without the final "index.html".

如果访问的链接以/index.html结尾，则会被重定向到不包含index.html的那个路径。根据测试，如果后缀是index.html且文件存在，则显示其内容，否则显示目录结构。
Eg1:
``` go
package main

import (
	"log"
	"net/http"
)

func main() {
	// Simple static webserver:
	log.Fatal(http.ListenAndServe(":8080", http.FileServer(http.Dir("/usr/share/doc"))))
}
```

Eg2: StripPrefix
``` go
package main

import (
	"net/http"
)

func main() {
	// To serve a directory on disk (/tmp) under an alternate URL
	// path (/tmpfiles/), use StripPrefix to modify the request
	// URL's path before the FileServer sees it:
	http.Handle("/tmpfiles/", http.StripPrefix("/tmpfiles/", http.FileServer(http.Dir("/tmp"))))
}
```