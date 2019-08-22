---
title: golang基础数据类型
date: 2018-07-15 21:44:50
tags: [golang, data types]
category: [golang]
---
来自[go语言圣经](https://books.studygolang.com/gopl-zh/ch3/ch3.html)
Go语言将数据类型分为四类：基础类型、复合类型、引用类型和接口类型。
* 基础类型，包括：数字、字符串和布尔型。
* 复合数据类型——数组和结构体——是通过组合简单类型，来表达更加复杂的数据结构。
* 引用类型包括指针、切片、字典、函数、通道，虽然数据种类很多，但它们都是对程序中一个变量或状态的间接引用。这意味着对任一引用类型数据的修改都会影响所有该引用的拷贝。
* 我们将在第7章介绍接口类型。

## 基础数据类型（basic data types)
* Unicode字符rune类型是和int32等价的类型，通常用于表示一个Unicode码点。这两个名称可以互换使用。同样byte也是uint8类型的等价类型，byte类型一般用于强调数值是一个原始的数据而不是一个小的整数。
* &,|,^, &^的有趣解释。可以看作两个集合的操作
> &      位运算 AND 
> |      位运算 OR 
> ^      位运算 XOR 
> &^     位清空 (AND NOT) 
```
var x uint8 = 1<<1 | 1<<5
var y uint8 = 1<<1 | 1<<2
fmt.Printf("%08b\n", x&y)  // "00000010", the intersection {1}
fmt.Printf("%08b\n", x|y)  // "00100110", the union {1, 2, 5}
fmt.Printf("%08b\n", x^y)  // "00100100", the symmetric difference {2, 5}
fmt.Printf("%08b\n", x&^y) // "00100000", the difference {5}
```
* 常量math.MaxFloat32表示float32能表示的最大数值，大约是 3.4e38；对应的math.MaxFloat64常量大约是1.8e308。它们分别能表示的最小值近似为1.4e-45和4.9e-324。一个float32类型的浮点数可以提供大约6个十进制数的精度，而float64则可以提供约15个十进制数的精度；通常应该优先使用float64类型，因为float32类型的累计计算误差很容易扩散，并且float32能精确表示的正整数并不是很大（译注：因为float32的有效bit位只有23个，其它的bit位用于指数和符号；当整数大于23bit能表达的范围时，float32的表示将出现误差）：
* 内置的len函数可以返回一个字符串中的字节数目（不是rune字符数目）。
    * 因为Go语言源文件总是用UTF8编码，并且Go语言的文本字符串也以UTF8编码的方式处理，因此我们可以将Unicode码点也写到字符串面值中。
    * Go语言字符串面值中的Unicode转义字符让我们可以通过Unicode码点输入特殊的字符。有两种形式：\uhhhh对应16bit的码点值，\Uhhhhhhhh对应32bit的码点值，其中h是一个十六进制数字；一般很少需要使用32bit的形式。每一个对应码点的UTF8编码。例如：下面的字母串面值都表示相同的值：
    ```
    "世界"
    "\xe4\xb8\x96\xe7\x95\x8c"
    "\u4e16\u754c"
    "\U00004e16\U0000754c"
    ```
    * 为了获取真实的字符有两种方式：
        * unicode/utf8包提供了该功能：
        ```
        for i := 0; i < len(s); {
            r, size := utf8.DecodeRuneInString(s[i:])
            fmt.Printf("%d\t%c\n", i, r)
            i += size
        }
        ```
        * **rune值绝对不是utf-8编码的，编码格式为UTF-32**，'世'对应的rune值为19990(0x4e10)，码点表可查看[这里](http://www.fileformat.info/info/unicode/char/4e16/index.htm)。找到一个可以查UTF-32码表的[网站](https://www.fileformat.info/info/charset/UTF-32/list.htm?start=1024)
        * Go语言的range循环在处理字符串的时候，会自动隐式解码UTF8字符串。
        ```
        for i, r := range "Hello, 世界" {
            fmt.Printf("%d\t%q\t%d\n", i, r, r)
        }
        ```
    * string转换为rune类型的unicode。直接用rune[]强制类型转换。
    ```
    s := "プログラム"
    fmt.Printf("% x\n", s) // "e3 83 97 e3 83 ad e3 82 b0 e3 83 a9 e3 83 a0"
    r := []rune(s)
    fmt.Printf("%x\n", r)  // "[30d7 30ed 30b0 30e9 30e0]"
    ```
    * rune类型的unicode转换为string。也是强制类型转换。
     ```
     fmt.Println(string(r)) // "プログラム"
     ```
    * 将一个整数转型为字符串意思是生成以只包含对应Unicode码点字符的**UTF8字符串**：
    ```
    fmt.Println(string(65))     // "A", not "65"
    fmt.Println(string(0x4eac)) // "京"
    ```
    * 标准库中有四个包对字符串处理尤为重要：bytes、strings、strconv和unicode包。
        1. strings包提供了许多如字符串的查询、替换、比较、截断、拆分和合并等功能。
        2. bytes包也提供了很多类似功能的函数，但是针对和字符串有着相同结构的[]byte类型。因为字符串是只读的，因此逐步构建字符串会导致很多分配和复制。在这种情况下，使用bytes.Buffer类型将会更有效。
        3. strconv包提供了布尔型、整型数、浮点数和对应字符串的相互转换，还提供了双引号转义相关的转换。字符和数字的转换也可以通过fmt.Sprintf,fmt.Scanf之类的函数解决。
        4. unicode包提供了IsDigit、IsLetter、IsUpper和IsLower等类似功能，它们用于给字符分类。每个函数有一个单一的rune类型的参数，然后返回一个布尔值。而像ToUpper和ToLower之类的转换函数将用于rune字符的大小写转换。所有的这些函数都是遵循Unicode标准定义的字母、数字等分类规范。strings包也有类似的函数，它们是ToUpper和ToLower，将原始字符串的每个字符都做相应的转换，然后返回新的字符串。
* 常量表达式的值在**编译期计算**，而不是在运行期。每种常量的潜在类型都是基础类型：boolean、string或数字。iota仅用于const，且从0开始每行自动加1。如：
    ```
    type Weekday int

    const (
        Sunday Weekday = iota
        Monday
        Tuesday
        Wednesday
        Thursday
        Friday
        Saturday
    )
    ```
    * 当尝试将这些无类型的常量转为一个接口值时，这些默认类型将显得尤为重要，因为要靠它们明确接口对应的动态类型。
```
fmt.Printf("%T\n", 0)      // "int"
fmt.Printf("%T\n", 0.0)    // "float64"
fmt.Printf("%T\n", 0i)     // "complex128"
fmt.Printf("%T\n", '\000') // "int32" (rune)
```