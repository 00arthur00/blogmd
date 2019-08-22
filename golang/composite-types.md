---
title: golang 复合类型(composite types)
date: 2018-07-17 22:25:11
tags: [golang, composite types]
category: [golang]
---
复合类型(composite types)，就是用基础类型组合出来的类型，包括：数组、slice、map和结构体
## 数组
* 个数固定，编译时确定。
* 可以让编译器自己推断，如“q := [...]int{1, 2, 3}”
* 初始化：
    1. 直接初始化：
    ```
    q := [3]int{1, 2, 3}
    ```
    2. 带索引的初始化：
    ```
    type Currency int
    const (
        USD Currency = iota // 美元
        EUR                 // 欧元
        GBP                 // 英镑
        RMB                 // 人民币
    )
    symbol := [...]string{USD: "$", EUR: "€", GBP: "￡", RMB: "￥"}
    fmt.Println(RMB, symbol[RMB]) // "3 ￥"
    ```
    3. 未指定初始值的元素默将用零值初始化。如“r := [...]int{99: -1}”，定义了一个含有100个元素的数组r，最后一个元素被初始化为-1，其它元素都是用0初始化。
* 比较。可以直接比较是否相等，**但类型必须相同，否则编译报错。**
## Slice
* Slice（切片）代表变长的序列，序列中每个元素都有相同的类型。slice的语法和数组很像，只是没有固定长度而已。
* 一个slice由三个部分构成：指针、长度和容量。**在某种程度上可以理解为c++的vector**
![slice内部结构](/images/ch4-01.png "slice内部结构")
* 初始化:使用make或者直接隐式赋值。
```
make([]T, len)
make([]T, len, cap) // same as make([]T, cap)[:len]
s := []int{0, 1, 2, 3, 4, 5}
```
* 初始值
```
var s []int    // len(s) == 0, s == nil
s = nil        // len(s) == 0, s == nil
s = []int(nil) // len(s) == 0, s == nil
s = []int{}    // len(s) == 0, s != nil
```
* 访问。通过下标访问元素。**如果切片操作超出cap(s)的上限将导致一个panic异常**
* 比较： **slice之间不能比较**，唯一合法的比较是 slice和nil的比较，如 slice==nil 
* append，追加元素，一个，多个甚至是同类型的slice。运行机制类似c++中的vector。
```
var x []int
x = append(x, 1)
x = append(x, 2, 3)
x = append(x, 4, 5, 6)
x = append(x, x...) // append the slice x
fmt.Println(x)      // "[1 2 3 4 5 6 1 2 3 4 5 6]
```
* 复制slice。可以使用函数copy(dst,src),不用担心覆盖，只进行min(len(dst),len(src))次
* 删除元素。没有现成的函数，不过可以通过下面的remove函数实现。
```
func remove(slice []int, i int) []int {
    copy(slice[i:], slice[i+1:])
    return slice[:len(slice)-1]
}

func main() {
    s := []int{5, 6, 7, 8, 9}
    fmt.Println(remove(s, 2)) // "[5 6 8 9]"
}
```
## Map
在golang中Map是hash表的实现，和C++中的map不同。map[K]V中K的类型是**必须是进行==比较运算符的数据类型，不要用浮点型作为key**；V的类型没有限制。
### 创建方式
1. make 函数
```
ages := make(map[string]int) // mapping from strings to ints
```
2. 字面值
```
ages := map[string]int{
    "alice":   31,
    "charlie": 34,
}
//相当于下面
ages := make(map[string]int)
ages["alice"] = 31
ages["charlie"] = 34
```
3. 创建空的map的表达式是map[string]int{}

### 删除元素
使用内置的delete函数可以删除元素.
```
delete(ages, "alice") // remove element ages["alice"]
```
元素不存在也是安全的。

### 访问Map
* 使用下标访问单个元素：
```
ages["alice"] = 32
fmt.Println(ages["alice"]) // "32"
```
**元素不存在也是安全的，如果一个查找失败将返回value类型对应的零值**。
```
ages["bob"] = ages["bob"] + 1 // happy birthday!
ages["bob"] += 1
ages["bob"]++
```
* map中的元素并不是一个变量，因此我们不能对map的元素进行取址操作。禁止对map元素取址的原因是map可能随着元素数量的增长而重新分配更大的内存空间，从而可能导致之前的地址无效。
* 使用range进行遍历：
```
for name, age := range ages {
    fmt.Printf("%s\t%d\n", name, age)
}
```
**每次都使用随机的遍历顺序可以强制要求程序不会依赖具体的哈希函数实现，每次返回的key/value对都是随机的(by design)。**
* **排序**需要对把key取出来放在slice里，然后再排序。
```
import "sort"

var names []string
//names := make([]string, 0, len(ages)) //better solution
for name := range ages {
    names = append(names, name)
}
sort.Strings(names)
for _, name := range names {
    fmt.Printf("%s\t%d\n", name, ages[name])
}
```
* 零值
map上的大部分操作，包括查找、删除、len和range循环都可以安全工作在nil值的map上，它们的行为和一个空的map类似。但是向一个nil值的map存入元素将导致一个panic异常：
```
var ages map[string]int
fmt.Println(ages == nil)    // "true"
fmt.Println(len(ages) == 0) // "true"
ages["carol"] = 21 // panic: assignment to entry in nil map
```
**在向map存数据前必须先创建map。**
* 通过key作为索引下标来访问map将产生一个value。如果key在map中是存在的，那么将得到与key对应的value；如果key不存在，那么将得到value对应类型的零值，正如我们前面看到的ages["bob"]那样。这个规则很实用，但是有时候可能需要知道对应的元素是否真的是在map之中。
**所以判断一个key是否存在应该是:if age, ok := ages["bob"]; !ok { /* ... */ }.**
* 和slice一样，map之间也不能进行相等比较；唯一的例外是和nil进行比较。下面是一个比较两个map是否相等的函数。
```
func equal(x, y map[string]int) bool {
    if len(x) != len(y) {
        return false
    }
    for k, xv := range x {
        if yv, ok := y[k]; !ok || yv != xv {
            return false
        }
    }
    return true
}
```
* 有时候我们需要一个map或set的key是slice类型，但是map的key必须是可比较的类型，但是slice并不满足这个条件。不过，我们可以通过两个步骤绕过这个限制。第一步，定义一个辅助函数k，将slice转为map对应的string类型的key，确保只有x和y相等时k(x) == k(y)才成立。然后创建一个key为string类型的map，在每次对map操作时先用k辅助函数将slice转化为string类型。
```
var m = make(map[string]int)

func k(list []string) string { return fmt.Sprintf("%q", list) }

func Add(list []string)       { m[k(list)]++ }
func Count(list []string) int { return m[k(list)] }
```
**使用同样的技术可以处理任何不可比较的key类型，而不仅仅是slice类型。同时，辅助函数k(x)也不一定是字符串类型，它可以返回任何可比较的类型，例如整数、数组或结构体等。**
* map的惰性初始化
```
var graph = make(map[string]map[string]bool)

func addEdge(from, to string) {
    edges := graph[from]
    if edges == nil {
        edges = make(map[string]bool)
        graph[from] = edges
    }
    edges[to] = true
}

func hasEdge(from, to string) bool {
    return graph[from][to]
}
```
## 结构体
* 结构体是一种聚合的数据类型，是由零个或多个任意类型的值聚合成的实体。可以进行点操作，取地址操作但是没有->操作。
* 如果结构体成员名字是以大写字母开头的，那么该成员就是导出的；这是Go语言导出规则决定的。一个结构体可能同时包含导出和未导出的成员。

### 结构体字面值
两种方式如下：
```
type Point struct{ X, Y int }
p := Point{1, 2}
p2 := Point{Y:2}
```
第一种要求以结构体成员定义的**顺序**为每个结构体成员指定一个字面值。
第二种以成员名字和相应的值来初始化，可以包含部分或全部的成员。如果成员被忽略的话将默认用零值，如p2.X==0。
两种不同形式的写法不能混合使用。
### 结构体比较
如果结构体的全部成员都是可以比较的，那么结构体也是可以比较的，那样的话两个结构体将可以使用==或!=运算符进行比较。**所以含有map,slice的结构体不能比较。**
### 结构体嵌入和匿名成员
Go语言有一个特性让我们只声明一个成员对应的数据类型而不指名成员的名字；这类成员就叫匿名成员。
```
type Point struct {
    X, Y int
}
type Circle struct {
    Point
    Radius int
}

type Wheel struct {
    Circle
    Spokes int
}
var w Wheel
w.X = 8            // equivalent to w.Circle.Point.X = 8
w.Y = 8            // equivalent to w.Circle.Point.Y = 8
w.Radius = 5       // equivalent to w.Circle.Radius = 5
w.Spokes = 20
```
**在右边的注释中给出的显式形式访问这些叶子成员的语法依然有效，因此匿名成员并不是真的无法访问了。其中匿名成员Circle和Point都有自己的名字——就是命名的类型名字——但是这些名字在点操作符中是可选的。**

不幸的是，结构体字面值并没有简短表示匿名成员的语法， 因此下面的语句都不能编译通过：
```
w = Wheel{8, 8, 5, 20}                       // compile error: unknown fields
w = Wheel{X: 8, Y: 8, Radius: 5, Spokes: 20} // compile error: unknown fields
```
**结构体字面值必须遵循形状类型声明时的结构，所以我们只能用下面的两种语法，它们彼此是等价的：**
```
w = Wheel{Circle{Point{8, 8}, 5}, 20}

w = Wheel{
    Circle: Circle{
        Point:  Point{X: 8, Y: 8},
        Radius: 5,
    },
    Spokes: 20, // NOTE: trailing comma necessary here (and at Radius)
}
```
**外层的结构体不仅仅是获得了匿名成员类型的所有成员，而且也获得了该类型导出的全部的方法。**

## JSON
JavaScript对象表示法(JSON)是一种用于发送和接收结构化信息的标准协议。
基本的JSON类型有数字（十进制或科学记数法）、布尔值（true或false）、字符串，其中字符串是以双引号包含的Unicode字符序列，支持和Go语言类似的反斜杠转义特性，不过JSON使用的是\Uhhhh转义数字来表示一个UTF-16编码（译注：UTF-16和UTF-8一样是一种变长的编码，有些Unicode码点较大的字符需要用4个字节表示；而且UTF-16还有大端和小端的问题），而不是Go语言的rune类型。
这些基础类型可以通过JSON的数组和对象类型进行递归组合。一个JSON数组是一个有序的值序列，写在一个方括号中并以逗号分隔；一个JSON数组可以用于编码Go语言的数组和slice。一个JSON对象是一个字符串到值的映射，写成以系列的name:value对形式，用花括号包含并以逗号分隔；JSON的对象类型可以用于编码Go语言的map类型（key类型是字符串）和结构体。例如：
```
boolean         true
number          -273.15
string          "She said \"Hello, BF\""
array           ["gold", "silver", "bronze"]
object          {"year": 1980,
                 "event": "archery",
                 "medals": ["gold", "silver", "bronze"]}
```
在编码时，默认使用Go语言结构体的成员名字作为JSON的对象。只有导出的结构体成员才会被编码，这也就是我们为什么选择用大写字母开头的成员名称。
```
type Movie struct {
    Title  string
    Year   int  `json:"released"`
    Color  bool `json:"color,omitempty"`
    Actors []string
}

var movies = []Movie{
    {Title: "Casablanca", Year: 1942, Color: false,
        Actors: []string{"Humphrey Bogart", "Ingrid Bergman"}},
    {Title: "Cool Hand Luke", Year: 1967, Color: true,
        Actors: []string{"Paul Newman"}},
    {Title: "Bullitt", Year: 1968, Color: true,
        Actors: []string{"Steve McQueen", "Jacqueline Bisset"}},
    // ...
}
```
结构体到JSON，json.Marshal()或json.MarshalIndent;JSON到结构体，json.Unmarshal()
```
data, err := json.Marshal(movies)
if err != nil {
    log.Fatalf("JSON marshaling failed: %s", err)
}
fmt.Printf("%s\n", data)

var titles []struct{ Title string }
if err := json.Unmarshal(data, &titles); err != nil {
    log.Fatalf("JSON unmarshaling failed: %s", err)
}
fmt.Println(titles) // "[{Casablanca} {Cool Hand Luke} {Bullitt}]"
```
结构体的Tag用于控制编码的类型(json)，变量和参数。

## 文本和HTML模板
这些功能是由text/template和html/template等模板包提供的，它们提供了一个将变量值填充到一个文本或HTML格式的模板的机制。
一个模板是一个字符串或一个文件，里面包含了一个或多个由双花括号包含的**{{action}}**对象。大部分的字符串只是按字面值打印，但是对于actions部分将触发其它的行为。每个actions都包含了一个用模板语言书写的表达式，一个action虽然简短但是可以输出复杂的打印值，**模板语言包含通过选择结构体的成员、调用函数或方法、表达式控制流if-else语句和range循环语句，还有其它实例化模板等诸多特性**。下面是一个简单的模板字符串：
```
const templ = `{{.TotalCount}} issues:
{{range .Items}}----------------------------------------
Number: {{.Number}}
User:   {{.User.Login}}
Title:  {{.Title | printf "%.64s"}}
Age:    {{.CreatedAt | daysAgo}} days
{{end}}`

report, err := template.New("report").
    Funcs(template.FuncMap{"daysAgo": daysAgo}).
    Parse(templ)
if err != nil {
    log.Fatal(err)
}
```
**对于每一个action，都有一个当前值的概念，对应点操作符，写作“.”，当前值“.”最初被初始化为调用模板时的参数。在一个action中，|操作符表示将前一个表达式的结果作为后一个函数的输入，类似于UNIX中管道的概念。**
生成模板的输出需要两个处理步骤。第一步是要分析模板并转为内部表示，然后基于指定的输入执行模板。分析模板部分一般只需要执行一次。
因为模板通常在编译时就测试好了，如果模板解析失败将是一个致命的错误。**template.Must辅助函数可以简化这个致命错误的处理**：它接受一个模板和一个error类型的参数，检测error是否为nil（如果不是nil则发出panic异常），然后返回传入的模板。
**html/template模板包。它使用和text/template包相同的API和模板语言，但是增加了一个将字符串自动转义特性，这可以避免输入字符串和HTML、JavaScript、CSS或URL语法产生冲突的问题。**这个特性还可以避免一些长期存在的安全问题，比如通过生成HTML注入攻击，通过构造一个含有恶意代码的问题标题，这些都可能让模板输出错误的输出，从而让他们控制页面。