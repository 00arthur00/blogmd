title: rabbitmq入门介绍
tags:
  - golang
  - rabbitmq
category: [rabbitmq]
date: 2018-09-18 16:35:55
---
[源自官方文档](https://www.rabbitmq.com/tutorials/tutorial-one-go.html)
### 介绍
**前提:** 本教程假定RabbitMQ已在标准端口（5672）上的localhost上[**安装**](https://www.rabbitmq.com/download.html)并运行。 如果您使用不同的主机，端口或凭据，则需要调整连接设置。

RabbitMQ是一个消息代理(broker)：它接受和转发消息。 您可以将其视为邮局：当您将要发布的邮件放在邮箱中时，您可以确信邮递员一定会把邮件送到收件人那里。 在这个比喻中，RabbitMQ是邮箱，邮局和邮递员。

RabbitMQ和邮局之间的主要区别在于它不处理纸张，而是接受，存储和转发二进制blob数据 - *消息*。

RabbitMQ和一般的消息传递使用了一些术语。

*生产只不过是发送(Producing means nothing more than sending)。 发送消息的程序是生产者：*
<div align=center>
![producer](/images/rabbitmq/producer.png)
</div>

*队列(a queue)*是RabbitMQ中的邮箱的名称。 虽然消息流经RabbitMQ和您的应用程序，但它们只能存储在*队列*中。 *队列*仅受主机的内存和磁盘限制的约束，它本质上是一个大的消息缓冲区。 许多*生产者*可以发送到一个队列的消息，并且许多*消费者*可以尝试从一个队列接收数据。 这就是我们代表队列的方式：
<div align=center>
![producer](/images/rabbitmq/queue.png)
</div>

消费(consuming)与接受(receiving)有类似的意义。 *消费者*是一个主要等待接收消息的程序：
<div align=center>
![producer](/images/rabbitmq/consumer.png)
</div>

请注意，生产者，消费者和代理不必位于同一主机上; 实际上在大多数应用中都不会这么做。

### "Hello World"
**(使用Go RabbitMQ客户端)**
在本教程的这一部分中，我们将在Go中编写两个小程序; 发送单个消息的生产者，以及接收消息并将其打印出来的消费者。 先来个热身，我们将掩盖[Go RabbitMQ](http://godoc.org/github.com/streadway/amqp) API中的一些细节，专注于这个非常简单的事情。 只是简单的传递消息“Hello World”。

在下图中，“P”是我们的生产者，“C”是我们的消费者。 中间的框是一个队列 - RabbitMQ代表消费者保留的消息缓冲区。

<div align=center>
![producer](/images/rabbitmq/python-one.png)
</div>

**Go RabbitMQ 客户端函数库**

RabbitMQ支持多种协议。 本教程使用AMQP 0-9-1，它是一种开放的，通用的消息传递协议。 RabbitMQ有许多不同语言的客户端。 我们将在本教程中使用Go amqp客户端。

首先，使用go get安装amqp：
``` bash
go get github.com/streadway/amqp
```
现在我们安装好了amqp，可以开始写代码了。

### 发送消息(sending)
<div align=center>
![producer](/images/rabbitmq/sending.png)
</div>
我们会将我们的消息发布者（发件人）send.go和我们的消息使用者（接收者）称为receive.go。 发布者将连接到RabbitMQ，发送单个消息，然后退出。

在[send.go](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/go/send.go)中，我们需要先import已安装好的库：
``` golang
package main

import (
  "fmt"
  "log"

  "github.com/streadway/amqp"
)
```
我们还需要一个辅助函数来检查每个amqp调用的返回值：
``` golang
func failOnError(err error, msg string) {
  if err != nil {
    log.Fatalf("%s: %s", msg, err)
    panic(fmt.Sprintf("%s: %s", msg, err))
  }
}
```
然后连接RabbitMQ服务器：
``` golang
conn, err := amqp.Dial("amqp://guest:guest@localhost:5672/")
failOnError(err, "Failed to connect to RabbitMQ")
defer conn.Close()
```
连接抽象了套接字连接，并为我们负责协议版本协商和身份验证等。 接下来，我们创建一个频道，这是完成任务的大部分API所在的位置：
``` golang
ch, err := conn.Channel()
failOnError(err, "Failed to open a channel")
defer ch.Close()
```
为了发送，我们必须声明一个队列供我们发送; 然后我们可以向队列发布消息：
``` golang
q, err := ch.QueueDeclare(
  "hello", // name
  false,   // durable
  false,   // delete when unused
  false,   // exclusive
  false,   // no-wait
  nil,     // arguments
)
failOnError(err, "Failed to declare a queue")

body := "Hello World!"
err = ch.Publish(
  "",     // exchange
  q.Name, // routing key
  false,  // mandatory
  false,  // immediate
  amqp.Publishing {
    ContentType: "text/plain",
    Body:        []byte(body),
  })
failOnError(err, "Failed to publish a message")
```
声明队列是幂等的(idempotent) - 只有在它不存在的情况下才会创建它。 消息内容是一个字节数组，因此您可以编码任何您喜欢的内容。

[代码位置从这获取](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/go/send.go).

**发送不起作用!**

如果这是您第一次使用RabbitMQ并且没有看到“已发送”消息，那么您可能会感到头疼，想知道可能出现的问题。 也许代理是在没有足够的可用磁盘空间的情况下启动的（默认情况下它至少需要200 MB空闲），因此拒绝接受消息。 检查代理日志文件以确认并在必要时减少限制。 [配置文件文档](http://www.rabbitmq.com/configure.html#config-items)将向您展示如何设置disk_free_limit。

### 接收消息(receiving)

上面说的是发布者(publisher)。 我们的消费者的消息来自于RabbitMQ推送，因此与发布单个消息的发布者不同，我们将使其保持运行以侦听消息并将其打印出来。

<div align=center>
![producer](/images/rabbitmq/receiving.png)
</div>
代码（在[receive.go](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/go/receive.go)中）具有与*send*相同的导入和帮助函数：
```golang
package main

import (
  "fmt"
  "log"

  "github.com/streadway/amqp"
)

func failOnError(err error, msg string) {
  if err != nil {
    log.Fatalf("%s: %s", msg, err)
    panic(fmt.Sprintf("%s: %s", msg, err))
  }
}
```
设置与发布者相同; 我们打开一个连接和一个通道，并声明我们将要消费的队列。 请注意，这与发送到的队列匹配。
``` golang
conn, err := amqp.Dial("amqp://guest:guest@localhost:5672/")
failOnError(err, "Failed to connect to RabbitMQ")
defer conn.Close()

ch, err := conn.Channel()
failOnError(err, "Failed to open a channel")
defer ch.Close()

q, err := ch.QueueDeclare(
  "hello", // name
  false,   // durable
  false,   // delete when usused
  false,   // exclusive
  false,   // no-wait
  nil,     // arguments
)
failOnError(err, "Failed to declare a queue")
```
请注意，我们在这也声明队列。因为我们可能会在发布者之前启动使用者，所以我们希望在尝试使用消息之前确保队列存在。

我们即将告诉服务器从队列中传递消息。 因为它会异步地向我们发送消息，所以我们将在goroutine中读取来自通道（由amqp :: Consume返回）的消息。

[receive.go的完整代码在这里](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/go/receive.go).

### 综合(Putting it all together)

现在我们可以运行这两个脚本。 在终端中，运行发布者：
``` bash
go run send.go
```
然后，运行消费者:
``` bash
go run receive.go
```

消费者将通过RabbitMQ打印从发布者处获得的消息。 消费者将继续运行，等待消息（使用Ctrl-C停止消息），因此请尝试从另一个终端运行发布者。

如果要检查队列，请尝试使用*rabbitmqctl list_queues*。

下一个部分将讲述如何构建一个简单的工作队列(work queue).