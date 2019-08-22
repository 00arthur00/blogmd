title: rabbitmq RPC
tags:
  - golang
  - rabbitmq
category: [rabbitmq]
date: 2018-10-18 17:11:03
---
**(使用Go RabbitMQ客户端)**

在第二篇教程中，我们学习了如何使用工作队列在多个工作人员之间分配耗时的任务。

但是如果我们需要在远程计算机上运行一个函数并等待结果呢？嗯，这是一个不同的故事。此模式通常称为远程过程调用或RPC。

在本教程中，我们将使用RabbitMQ构建RPC系统：客户端和可伸缩的RPC服务器。由于我们没有任何值得分发的耗时任务，我们将创建一个返回Fibonacci数字的虚拟RPC服务。

**A note on RPC**

尽管RPC在计算中是一种非常常见的模式，但它经常受到批评。当程序员不知道函数调用是本地的还是慢的RPC时，会出现问题。这样的混淆导致系统不可预测，并增加了调试的不必要的复杂性。错误使用RPC可以导致不可维护的意大利面条代码，而不是简化软件。

考虑到这一点，请考虑以下建议：
```
确定明显哪个函数调用是本地的，哪个是远程的。
系统文档化。使组件之间的依赖关系变得清晰。
处理错误情况。当RPC服务器长时间停机时，客户端应该如何反应？
```
如有疑问，请避免使用RPC。如果可以，您应该使用异步管道 - 而不是类似RPC的阻塞，将结果异步推送到下一个计算阶段。

### 回调队列(Callback queue)
一般来说，通过RabbitMQ进行RPC很容易。客户端发送请求消息，服务器回复响应消息。为了接收响应，我们需要发送带有请求的“回调”队列地址。我们可以使用默认队列。 我们来试试吧：
``` golang
q, err := ch.QueueDeclare(
  "",    // name
  false, // durable
  false, // delete when usused
  true,  // exclusive
  false, // noWait
  nil,   // arguments
)

err = ch.Publish(
  "",          // exchange
  "rpc_queue", // routing key
  false,       // mandatory
  false,       // immediate
  amqp.Publishing{
    ContentType:   "text/plain",
    CorrelationId: corrId,
    ReplyTo:       q.Name,
    Body:          []byte(strconv.Itoa(n)),
})
```
**消息属性**

AMQP 0-9-1预定义了14个消息属性。除了下面的几个，大部分的属性不怎么用：
```
persistent:将消息标记为persistent（值为true）或瞬态（false）。您可能会记住第二个教程中的这个属性。
content_type:用于描述编码的mime类型。 例如，对于经常使用的JSON编码，将此属性设置为：application / json是一种很好的做法。
reply_to:通常用于命名回调队列。
correlation_id:用于将RPC响应与请求相关联。
```

### Correlation Id

在上面介绍的方法中，我们建议为每个RPC请求创建一个回调队列。这是非常低效的，但幸运的是有更好的方法 - 让我们为每个客户端创建一个回调队列。

这引发了一个新问题，在该队列中收到响应后，不清楚响应属于哪个请求。那是在使用correlation_id属性的时候。我们将为每个请求将其设置为唯一值。稍后，当我们在回调队列中收到一条消息时，我们将查看此属性，并根据该属性，我们将能够将响应与请求进行匹配。如果我们看到未知的correlation_id值，我们可以安全地丢弃该消息 - 它不属于我们的请求。

您可能会问，为什么我们应该忽略回调队列中的未知消息，而不是失败并出现错误？这是由于服务器端可能存在竞争条件。虽然不太可能，但RPC服务器可能会在向我们发送答案之后，但在发送请求的确认消息之前死亡。如果发生这种情况，重新启动的RPC服务器将再次处理请求。这就是为什么在客户端上我们必须优雅地处理重复的响应，理想情况下RPC应该是幂等的。

### 总结(summary)
<div align=center>
![producer](/images/rabbitmq/python-six.png)
</div>

我们的RPC将这样工作：

客户端启动时，会创建一个匿名的独占回调队列。

对于RPC请求，客户端发送带有两个属性的消息：reply_to（设置为回调队列）和correlation_id（设置为每个请求的唯一值）。

请求被发送到rpc_queue队列。

RPC worker(server）在等待该队列上的请求。当出现请求时，它会执行该作业，并使用reply_to字段中的队列将带有结果的消息发送回客户端。

客户端等待回调队列上的数据。 出现消息时，它会检查correlation_id属性。 如果它与请求中的值匹配，则返回对应用程序的响应。

### 综合(Putting it all together)
斐波那契函数:
``` golang
func fib(n int) int {
        if n == 0 {
                return 0
        } else if n == 1 {
                return 1
        } else {
                return fib(n-1) + fib(n-2)
        }
}
```
我们声明了我们的斐波那契函数。它假定只有有效的正整数输入。（不要指望这个适用于大数字，它可能是最慢的递归实现）。

RPC服务器端代码[rpc_server.go](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/go/rpc_server.go):
``` golang
package main

import (
        "fmt"
        "log"
        "strconv"

        "github.com/streadway/amqp"
)

func failOnError(err error, msg string) {
        if err != nil {
                log.Fatalf("%s: %s", msg, err)
        }
}

func fib(n int) int {
        if n == 0 {
                return 0
        } else if n == 1 {
                return 1
        } else {
                return fib(n-1) + fib(n-2)
        }
}

func main() {
        conn, err := amqp.Dial("amqp://guest:guest@localhost:5672/")
        failOnError(err, "Failed to connect to RabbitMQ")
        defer conn.Close()

        ch, err := conn.Channel()
        failOnError(err, "Failed to open a channel")
        defer ch.Close()

        q, err := ch.QueueDeclare(
                "rpc_queue", // name
                false,       // durable
                false,       // delete when usused
                false,       // exclusive
                false,       // no-wait
                nil,         // arguments
        )
        failOnError(err, "Failed to declare a queue")

        err = ch.Qos(
                1,     // prefetch count
                0,     // prefetch size
                false, // global
        )
        failOnError(err, "Failed to set QoS")

        msgs, err := ch.Consume(
                q.Name, // queue
                "",     // consumer
                false,  // auto-ack
                false,  // exclusive
                false,  // no-local
                false,  // no-wait
                nil,    // args
        )
        failOnError(err, "Failed to register a consumer")

        forever := make(chan bool)

        go func() {
                for d := range msgs {
                        n, err := strconv.Atoi(string(d.Body))
                        failOnError(err, "Failed to convert body to integer")

                        log.Printf(" [.] fib(%d)", n)
                        response := fib(n)

                        err = ch.Publish(
                                "",        // exchange
                                d.ReplyTo, // routing key
                                false,     // mandatory
                                false,     // immediate
                                amqp.Publishing{
                                        ContentType:   "text/plain",
                                        CorrelationId: d.CorrelationId,
                                        Body:          []byte(strconv.Itoa(response)),
                                })
                        failOnError(err, "Failed to publish a message")

                        d.Ack(false)
                }
        }()

        log.Printf(" [*] Awaiting RPC requests")
        <-forever
}
```
服务器代码非常简单：

像往常一样，我们首先建立连接，通道和声明队列。我们可能希望运行多个服务器进程。 为了在多个服务器上平均分配负载，我们需要在通道上设置预取设置。我们使用Channel.Consume来获取我们从队列接收消息的go频道。 然后我们进入goroutine工作的地方并发回响应。

RPC客户端代码：
``` golang
package main

import (
        "fmt"
        "log"
        "math/rand"
        "os"
        "strconv"
        "strings"
        "time"

        "github.com/streadway/amqp"
)

func failOnError(err error, msg string) {
        if err != nil {
                log.Fatalf("%s: %s", msg, err)
        }
}

func randomString(l int) string {
        bytes := make([]byte, l)
        for i := 0; i < l; i++ {
                bytes[i] = byte(randInt(65, 90))
        }
        return string(bytes)
}

func randInt(min int, max int) int {
        return min + rand.Intn(max-min)
}

func fibonacciRPC(n int) (res int, err error) {
        conn, err := amqp.Dial("amqp://guest:guest@localhost:5672/")
        failOnError(err, "Failed to connect to RabbitMQ")
        defer conn.Close()

        ch, err := conn.Channel()
        failOnError(err, "Failed to open a channel")
        defer ch.Close()

        q, err := ch.QueueDeclare(
                "",    // name
                false, // durable
                false, // delete when usused
                true,  // exclusive
                false, // noWait
                nil,   // arguments
        )
        failOnError(err, "Failed to declare a queue")

        msgs, err := ch.Consume(
                q.Name, // queue
                "",     // consumer
                true,   // auto-ack
                false,  // exclusive
                false,  // no-local
                false,  // no-wait
                nil,    // args
        )
        failOnError(err, "Failed to register a consumer")

        corrId := randomString(32)

        err = ch.Publish(
                "",          // exchange
                "rpc_queue", // routing key
                false,       // mandatory
                false,       // immediate
                amqp.Publishing{
                        ContentType:   "text/plain",
                        CorrelationId: corrId,
                        ReplyTo:       q.Name,
                        Body:          []byte(strconv.Itoa(n)),
                })
        failOnError(err, "Failed to publish a message")

        for d := range msgs {
                if corrId == d.CorrelationId {
                        res, err = strconv.Atoi(string(d.Body))
                        failOnError(err, "Failed to convert body to integer")
                        break
                }
        }

        return
}

func main() {
        rand.Seed(time.Now().UTC().UnixNano())

        n := bodyFrom(os.Args)

        log.Printf(" [x] Requesting fib(%d)", n)
        res, err := fibonacciRPC(n)
        failOnError(err, "Failed to handle RPC request")

        log.Printf(" [.] Got %d", res)
}

func bodyFrom(args []string) int {
        var s string
        if (len(args) < 2) || os.Args[1] == "" {
                s = "30"
        } else {
                s = strings.Join(args[1:], " ")
        }
        n, err := strconv.Atoi(s)
        failOnError(err, "Failed to convert arg to integer")
        return n
}
```
现在是时候看看我们[rpc_client.go](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/go/rpc_client.go)和[rpc_server.go](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/go/rpc_server.go)的完整示例源代码了.

我们的RPC服务现已准备就绪。我们可以启动服务器：
``` shell
go run rpc_server.go
# => [x] Awaiting RPC requests
```
要请求斐波纳契数，请运行客户端：
``` shell
go run rpc_client.go 30
# => [x] Requesting fib(30)
```
此处介绍的设计并不是RPC服务的唯一可能实现，但它具有一些重要优势：

如果RPC服务器太慢，您可以通过运行另一个服务器来扩展。尝试在新控制台中运行第二个rpc_server.go。

在客户端，RPC只需要发送和接收一条消息。 因此，对于单个RPC请求，RPC客户端只需要一次网络往返。

我们的代码仍然相当简单，并不试图解决更复杂（但重要）的问题，例如：

如果没有运行服务器，客户应该如何反应？

客户端是否应该为RPC设置某种超时？

如果服务器出现故障并引发异常，是否应将其转发给客户端？

在处理之前防止无效的传入消息（例如检查边界，键入）。

**如果您想进行实验，您可能会发现[管理UI](https://www.rabbitmq.com/management.html)对于查看队列非常有用。**

### 生产[非]适用性免责声明
请记住，这个和其他教程都是教程。他们一次展示一个新概念，可能会故意过度简化某些事情而忽略其他事物。例如，为了简洁起见，在很大程度上省略了诸如连接管理，错误处理，连接恢复，并发和度量收集之类的主题。这种简化的代码不应被视为生产就绪。

在使用您的应用程序之前，请先查看其余[文档](https://www.rabbitmq.com/documentation.html)。我们特别推荐以下指南： [Publisher Confirms and Consumer Acknowledgements](https://www.rabbitmq.com/confirms.html),[Production Checklist](https://www.rabbitmq.com/production-checklist.html)和[Monitoring](https://www.rabbitmq.com/monitoring.html)。