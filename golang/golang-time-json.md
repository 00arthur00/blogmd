---
title: golang time 序列化反序列化
date: 2019-07-18 22:24:01
tags: [golang, function]
category: [golang]
---

golang中反序列化字符串推荐用time.RFC3339
``` golang
RFC3339     = "2006-01-02T15:04:05Z07:00"
RFC3339Nano = "2006-01-02T15:04:05.999999999Z07:00"
```

golang根据传递的字符串自动反序列化成time.Time类型并包含时区信息,序列化时根据时区进行序列化显示.
``` golang
package main

import (
	"encoding/json"
	"fmt"
	"time"
)
//代码在东八区的机器上运行
func main() {
    tnow:=time.Now()
    ret, err := json.Marshal(tnow)
    fmt.Println(string(ret), err)
    //输出当前东八区时间
    // "2019-07-18T22:53:00.987584125+08:00" <nil>
   
    t, _ := time.Parse(time.RFC3339, "2019-07-18T13:59:33.342880414Z")
	ret, err = json.Marshal(t)
    fmt.Println(string(ret), err)
    // 输出内容如下,和解析内容一致
    // "2019-07-18T13:59:33.342880414Z" <nil>
    fmt.Println("---------------------")
    
    t, _ = time.Parse(time.RFC3339, "2019-07-18T21:38:23.712986149+08:00")
	ret, err = json.Marshal(t)
    fmt.Println(string(ret), err)
    // 输出内容如下,和解析内容一致
    // "2019-07-18T21:38:23.712986149+08:00" <nil>
}
```

在docker上运行的机器默认是UTC时间,而我们在东八区北京时间.若有一个运行docker的进程用time.Now()(返回的UTC时间),则格式实际为time.RFC3339的UTC时间,而在docker之外运行的东八区的进行反序列化之后,进行运算,最后序列化输出的还是time.RFC3339格式的UTC时间.若接收方为其它语言如python编写的则需要解析RFC3339格式,否则否则容易出现时区误差.

## MarshalJSON/UnmarshalJSON
序列化时默认使用RFC3339Nano格式,但不是UTC的会按时区显示.
``` golang
// MarshalJSON implements the json.Marshaler interface.
// The time is a quoted string in RFC 3339 format, with sub-second precision added if present.
func (t Time) MarshalJSON() ([]byte, error) {
	if y := t.Year(); y < 0 || y >= 10000 {
		// RFC 3339 is clear that years are 4 digits exactly.
		// See golang.org/issue/4556#c15 for more discussion.
		return nil, errors.New("Time.MarshalJSON: year outside of range [0,9999]")
	}

	b := make([]byte, 0, len(RFC3339Nano)+2)
	b = append(b, '"')
	b = t.AppendFormat(b, RFC3339Nano)
	b = append(b, '"')
	return b, nil
}
```

UnmarshalJSON默认使用RFC3339格式解析,但非UTC的按时区设置时区
``` golang
// UnmarshalJSON implements the json.Unmarshaler interface.
// The time is expected to be a quoted string in RFC 3339 format.
func (t *Time) UnmarshalJSON(data []byte) error {
	// Ignore null, like in the main JSON package.
	if string(data) == "null" {
		return nil
	}
	// Fractional seconds are handled implicitly by Parse.
	var err error
	*t, err = Parse(`"`+RFC3339+`"`, string(data))
	return err
}
```

## python rfc3339库
安装tonyg-rfc3339:
``` sh
pip install tonyg-rfc3339
```
测试代码如下,时间差相同:
``` python
import rfc3339,datetime
dt = rfc3339.parse_datetime("2019-07-18T21:38:23.712986149+08:00")
now = datetime.datetime.now(dt.tzinfo)
print(now -dt)
#输出:2:09:36.500622

dt = rfc3339.parse_datetime("2019-07-18T13:38:23.712986149Z")
now = datetime.datetime.now(dt.tzinfo)
print(now -dt)
# 输出: 2:09:36.500667 
```