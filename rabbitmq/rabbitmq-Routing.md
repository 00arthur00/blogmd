---
title: 路由(Routing)
date: 2018-09-29 16:45:39
tags: [ golang, rabbitmq]
category: [rabbitmq]
---

**(使用Go RabbitMQ客户端)**

在上一个教程中，我们构建了一个简单的日志系统。我们能够向许多接收者广播日志消息。

在本教程中，我们将为其添加一个功能--我们将只能订阅一部分消息。例如，我们只能将关键错误消息定向到日志文件（以节省磁盘空间），同时仍然能够在控制台上打印所有日志消息。

### 绑定(Bindings)
在之前的例子中，我们已经创建了绑定。 您可能会记得以下代码：
``` golang
err = ch.QueueBind(
  q.Name, // queue name
  "",     // routing key
  "logs", // exchange
  false,
  nil)
```
一个绑定是一个交换机和一个队列的对应关系。可以简单的解读为：这个队列只对来自这个交换机的消息感兴趣。

绑定可以采用额外的routing_key参数。 为了避免与Channel.Publish参数混淆，我们将其称为绑定键(**binding key**)。 这就是我们如何使用键创建绑定：
```golang
err = ch.QueueBind(
  q.Name,    // queue name
  "black",   // routing key
  "logs",    // exchange
  false,
  nil)
```

绑定键的含义取决于交换类型。我们之前使用的fanout交换机只是简单的忽略了它的值。

### 直连交换机(Direct exchange)
我们上一个教程中的日志记录系统向所有消费者广播所有消息。 我们希望扩展它以允许根据消息的严重性过滤消息。 例如，我们可能希望将日志消息写入磁盘的脚本仅接收严重错误(critical errors)，而不是在警告(warn)或信息(info)日志消息上浪费磁盘空间。

我们使用的是fanout交换机，它没有给我们太大的灵活性--它只能进行漫无目的的广播。

相反，我们将使用直连交换机(direct exchange)。直接交换机背后的路由算法很简单--消息发送到绑定键(**binding key**)与消息的路由键(**routing key**)完全匹配队列。

举例来说，考虑如下设置：
<div align=center>
![producer](/images/rabbitmq/direct-exchange.png)
</div>

在本设置中，我们可以看到直连交换机X和两个绑定到它的队列。第一个队列绑定到绑定键(binding key)**orange**上，第二个队列有两个绑定，一个有绑定键**black**，另一个有绑定键**green**。

在这种设置中，使用路由键orange发送到交换机的消息将被路由到Q1。路由键为black和green的消息将被转到Q2.所有其他的消息都将被丢弃。

### 多个绑定(Multiple bindings)
<div align=center>
![producer](/images/rabbitmq/direct-exchange-multiple.png)
</div>

使用相同的绑定键绑定多个队列是完全合法的。在我们的例子中，我们可以在X和Q1之间添加绑定键black的绑定。在这种情况下，直连交换机将表现得像fanout(扇出)一样，并将消息广播到所有匹配的队列。路由键为black的消息将传送到Q1和Q2。

### 发送日志(Emitting logs)
我们将此模型用于我们的日志系统。我们会将消息发送给直连交换机，而不是(fanout)扇出交换机。我们将使用日志严重性(severity)作为路由键。这样接收脚本将能够选择它想要接收的严重性(severity)。让我们首先关注发送日志。

一如既往，让我们先先创建一个交换机：
``` golang
err = ch.ExchangeDeclare(
  "logs_direct", // name
  "direct",      // type
  true,          // durable
  false,         // auto-deleted
  false,         // internal
  false,         // no-wait
  nil,           // arguments
)
```
并且我们准备发送一条消息：
``` golang
err = ch.ExchangeDeclare(
  "logs_direct", // name
  "direct",      // type
  true,          // durable
  false,         // auto-deleted
  false,         // internal
  false,         // no-wait
  nil,           // arguments
)
failOnError(err, "Failed to declare an exchange")

body := bodyFrom(os.Args)
err = ch.Publish(
  "logs_direct",         // exchange
  severityFrom(os.Args), // routing key
  false, // mandatory
  false, // immediate
  amqp.Publishing{
    ContentType: "text/plain",
    Body:        []byte(body),
})
```
为了让事情更简单，我们将假定严重性(severity)可以是'info', 'warning','error'中的 一个。

### 订阅(Subscribe)
除了我们将创建为每个感兴趣的严重性创建一个绑定外，接收消息将和前面的教程一样。
``` golang
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
  log.Printf("Usage: %s [info] [warning] [error]", os.Args[0])
  os.Exit(0)
}
for _, s := range os.Args[1:] {
  log.Printf("Binding queue %s to exchange %s with routing key %s",
     q.Name, "logs_direct", s)
  err = ch.QueueBind(
    q.Name,        // queue name
    s,             // routing key
    "logs_direct", // exchange
    false,
    nil)
  failOnError(err, "Failed to bind a queue")
}
```

### 综合(Putting it all together)
<div align=center>
![producer](/images/rabbitmq/python-four.png)
</div>

emit_log_direct.go脚本代码：
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
                panic(fmt.Sprintf("%s: %s", msg, err))
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
                "logs_direct", // name
                "direct",      // type
                true,          // durable
                false,         // auto-deleted
                false,         // internal
                false,         // no-wait
                nil,           // arguments
        )
        failOnError(err, "Failed to declare an exchange")

        body := bodyFrom(os.Args)
        err = ch.Publish(
                "logs_direct",         // exchange
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
                s = "info"
        } else {
                s = os.Args[1]
        }
        return s
}
```

receive_logs_direct.go源码:
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
                panic(fmt.Sprintf("%s: %s", msg, err))
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
                "logs_direct", // name
                "direct",      // type
                true,          // durable
                false,         // auto-deleted
                false,         // internal
                false,         // no-wait
                nil,           // arguments
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
                log.Printf("Usage: %s [info] [warning] [error]", os.Args[0])
                os.Exit(0)
        }
        for _, s := range os.Args[1:] {
                log.Printf("Binding queue %s to exchange %s with routing key %s",
                        q.Name, "logs_direct", s)
                err = ch.QueueBind(
                        q.Name,        // queue name
                        s,             // routing key
                        "logs_direct", // exchange
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

如果您只想保存"warning"和"error"(没有"info")的日志消息到一个文件，只需要打开控制台并输入：
``` bash
go run receive_logs_direct.go warning error > logs_from_rabbit.log
```

如果您想要在屏幕上查看所有的日志消息，打开一个新终端并输入：
``` bash
go run receive_logs_direct.go info warning error
# => [*] Waiting for logs. To exit press CTRL+C
```

为了发送"error"日志消息，只需要输入：
``` bash
go run emit_log_direct.go error "Run. Run. Or it will explode."
# => [x] Sent 'error':'Run. Run. Or it will explode.'
```

([emit_log_direct.go](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/go/emit_log_direct.go)和[receive_logs_direct.go](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/go/receive_logs_direct.go)全部代码)

请移步到教程5，我们将学到怎样基于模式来监听消息。