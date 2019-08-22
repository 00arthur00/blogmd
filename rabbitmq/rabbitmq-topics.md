title: rabbitmq 话题(topic)
date: 2018-10-18 14:11:53
tags: [golang, rabbitmq]
category: [rabbitmq]
---
[源自官方文档](https://www.rabbitmq.com/tutorials/tutorial-five-go.html)
(**使用Go RabbitMQ客户端**)

在上一个的教程中，我们改进了日志系统。我们使用的是直连交换机，而不是使用只能进行虚拟广播的扇出交换机，并且有可能选择性地接收日志。

尽管使用直连交换机改进了我们的系统，但仍有一些限制--不能基于多个标准进行路由。

在我们的日志系统中，我们可能不仅想要基于严重等级进行订阅，而且想要基于发送日志的来源进行订阅。您也许知道从[syslog](http://en.wikipedia.org/wiki/Syslog)知道这个概念,它依据严重性(info/warn/crit...) 及设施(auth/cron/kern...)进行路由。


这将大大提升我们的灵活性--我们可能想监听来自'cron'的关键错误以及来自'kern'的所有日志。

为了在我们的日志系统中实现这一点，我们需要了解更复杂的话题(**topic**)交换机。


### 话题交换机(topic exchange)

发送到话题交换机的消息不能具有任意routing_key - 它必须是由点分隔的单词列表。单词可以是任何内容，但通常它们指定与消息相关的一些功能。一些有效的路由密钥示例：“stock.usd.nyse”，“nyse.vmw”，“quick.orange.rabbit”。路由键中可以包含任意数量的单词，最多可达255个字节。

绑定键也必须采用相同的形式。话题交换机背后的逻辑类似于直连交换机--使用特定路由键发送的消息将被传递到与匹配绑定键绑定的所有队列。但是，绑定键有两个重要的特殊情况：

	*（星号）可以替代一个单词。
    ＃（井号）可以替换零个或多个单词。

在一个例子中解释这个是最容易的：
<div align=center>
![producer](/images/rabbitmq/python-five.png)
</div>

在这个例子中，我们将发送所有描述动物的消息。消息将与包含三个单词（两个点）的路由键一起发送。路由键中的第一个单词将描述速度，第二个是颜色，第三个是物种："`<speed>.<color>.<species>`"。

我们创建了三个绑定：Q1绑定了绑定键"`*.orange.*`"，Q2绑定了“`*.*.rabbit`"和"`lazy.＃`"。

这些绑定可以概括为：

Q1对所有橙色动物感兴趣。

Q2希望听到关于兔子的一切，以及关于懒惰动物的一切。

路由键设置为“quick.orange.rabbit”的消息将传递到两个队列。消息“lazy.orange.elephant”也将同时发送给他们。 另一方面，“quick.orange.fox”只会进入第一个队列，而“lazy.brown.fox”只会进入第二个队列。“lazy.pink.rabbit”将仅传递到第二个队列一次，即使它匹配两个绑定。“quick.brown.fox”与任何绑定都不匹配，因此它将被丢弃。

如果我们违反约定并发送带有一个或四个单词的消息，例如“orange”或“quick.orange.male.rabbit”，会发生什么？ 好吧，这些消息将不匹配任何绑定，将丢失。

另一方面，“lazy.orange.male.rabbit”，即使它有四个单词，也会匹配最后一个绑定，并将被传递到第二个队列。

**Topic exchange**
话题交换机功能强大，可以像其他交换机一样。

当队列与“＃”绑定密钥绑定时--它将接收所有消息，而不管路由密钥--如扇出交换。

当特殊字符“*”（星号）和“＃”未在绑定中使用时，话题交换机的行为就像直连交换一样。

### 综合(Putting it all together)
我们将在日志记录系统中使用话题交换机。我们将首先假设日志的路由键有两个词："`<facility>。<severity>`"。

代码与上一个教程中的代码几乎相同。

emit_log_topic.go代码：
``` golang
package main

import (
        "fmt"
        "log"
        "os"
        "strings"

        "github.com/streadway/amqp"
)

func failOnError(err error, msg string) {
        if err != nil {
                log.Fatalf("%s: %s", msg, err)
        }
}

func main() {
        conn, err := amqp.Dial("amqp://guest:guest@localhost:5672/")
        failOnError(err, "Failed to connect to RabbitMQ")
        defer conn.Close()

        ch, err := conn.Channel()
        failOnError(err, "Failed to open a channel")
        defer ch.Close()

        err = ch.ExchangeDeclare(
                "logs_topic", // name
                "topic",      // type
                true,         // durable
                false,        // auto-deleted
                false,        // internal
                false,        // no-wait
                nil,          // arguments
        )
        failOnError(err, "Failed to declare an exchange")

        body := bodyFrom(os.Args)
        err = ch.Publish(
                "logs_topic",          // exchange
                severityFrom(os.Args), // routing key
                false, // mandatory
                false, // immediate
                amqp.Publishing{
                        ContentType: "text/plain",
                        Body:        []byte(body),
                })
        failOnError(err, "Failed to publish a message")

        log.Printf(" [x] Sent %s", body)
}

func bodyFrom(args []string) string {
        var s string
        if (len(args) < 3) || os.Args[2] == "" {
                s = "hello"
        } else {
                s = strings.Join(args[2:], " ")
        }
        return s
}

func severityFrom(args []string) string {
        var s string
        if (len(args) < 2) || os.Args[1] == "" {
                s = "anonymous.info"
        } else {
                s = os.Args[1]
        }
        return s
}
```

 receive_logs_topic.go代码：
 ``` golang
 package main

import (
        "fmt"
        "log"
        "os"

        "github.com/streadway/amqp"
)

func failOnError(err error, msg string) {
        if err != nil {
                log.Fatalf("%s: %s", msg, err)
        }
}

func main() {
        conn, err := amqp.Dial("amqp://guest:guest@localhost:5672/")
        failOnError(err, "Failed to connect to RabbitMQ")
        defer conn.Close()

        ch, err := conn.Channel()
        failOnError(err, "Failed to open a channel")
        defer ch.Close()

        err = ch.ExchangeDeclare(
                "logs_topic", // name
                "topic",      // type
                true,         // durable
                false,        // auto-deleted
                false,        // internal
                false,        // no-wait
                nil,          // arguments
        )
        failOnError(err, "Failed to declare an exchange")

        q, err := ch.QueueDeclare(
                "",    // name
                false, // durable
                false, // delete when usused
                true,  // exclusive
                false, // no-wait
                nil,   // arguments
        )
        failOnError(err, "Failed to declare a queue")

        if len(os.Args) < 2 {
                log.Printf("Usage: %s [binding_key]...", os.Args[0])
                os.Exit(0)
        }
        for _, s := range os.Args[1:] {
                log.Printf("Binding queue %s to exchange %s with routing key %s",
                        q.Name, "logs_topic", s)
                err = ch.QueueBind(
                        q.Name,       // queue name
                        s,            // routing key
                        "logs_topic", // exchange
                        false,
                        nil)
                failOnError(err, "Failed to bind a queue")
        }

        msgs, err := ch.Consume(
                q.Name, // queue
                "",     // consumer
                true,   // auto ack
                false,  // exclusive
                false,  // no local
                false,  // no wait
                nil,    // args
        )
        failOnError(err, "Failed to register a consumer")

        forever := make(chan bool)

        go func() {
                for d := range msgs {
                        log.Printf(" [x] %s", d.Body)
                }
        }()

        log.Printf(" [*] Waiting for logs. To exit press CTRL+C")
        <-forever
}
```

为了接收所有日志：
``` bash
go run receive_logs_topic.go "#"
```

为了接收所有从设施"kern"来的日志：
``` bash
go run receive_logs_topic.go "kern.*"
```

或者您仅想监听"critical"的日志:
``` bash
go run receive_logs_topic.go "*.critical"
```

您可以创建多个绑定：
``` bash
go run receive_logs_topic.go "kern.*" "*.critical"
```

为了发送一个带有路由键"kern.critical"的日志：
``` bash
go run emit_log_topic.go "kern.critical" "A critical kernel error"
```
玩这些程序玩得开心。请注意，代码不会对路由或绑定键做出任何假设，您可能希望使用两个以上的路由键参数。

(完整代码[emit_log_topic.go](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/go/emit_log_topic.go)、[receive_logs_topic.go](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/go/receive_logs_topic.go))

接下来，在教程6中了解如何将往返消息作为远程过程调用

### 生产[非]适用性免责声明
请记住，这个和其他教程都是教程。他们一次展示一个新概念，可能会故意过度简化某些事情而忽略其他事物。例如，为了简洁起见，在很大程度上省略了诸如连接管理，错误处理，连接恢复，并发和度量收集之类的主题。这种简化的代码不应被视为生产就绪。

在使用您的应用程序之前，请先查看其余[文档](https://www.rabbitmq.com/documentation.html)。我们特别推荐以下指南： [Publisher Confirms and Consumer Acknowledgements](https://www.rabbitmq.com/confirms.html),[Production Checklist](https://www.rabbitmq.com/production-checklist.html)和[Monitoring](https://www.rabbitmq.com/monitoring.html)。

