title: rabbitmq订阅发布(publish/subscribe)
tags:
  - golang
  - rabbitmq
category: [rabbitmq]
date: 2018-09-19 16:38:38
---
[源自官方文档](https://www.rabbitmq.com/tutorials/tutorial-three-go.html)
(**使用Go RabbitMQ客户端**)
在前一个教程，我们创建了一个work queue。work queue假定每一个任务被递送给一个worker。在本部分，我们将做一些完全不同的事情--我们将递送消息到多个消费者。这种模式被称为订阅/发布(publish/subscribe)。

为了讲述这种模式，我们将创建一个简单的日志系统。系统由两个程序构成--第一个发送(emit)消息,第二个将接收消息并打印。

在我们的日志系统中，每个运行的接收程序(receiver program)都将接收到消息。通过这种方式，我们可以运行一个接收程序并将日期定向到磁盘；同时我们能够运行另外一个接收程序并将其日志输出到屏幕。

基本上，发布消息就会广播消息到所有的接收者(receiver)。

### 交换机(Exchange)
在本教程的前一部分，我们从一个队列发送和接收消息。现在是时候引入RabbitMQ的完整消息模型了。

让我们快速过一遍前几个部分包含的内容：

*一个生产者(producer)是一个发送消息的用户程序*

*一个队列(queue)是一个存储消息的缓存*

*一个消费者(consumer)是一个接收消息的用户程序*

**RabbitMQ消息模型的核心思想是生产者从不直接发送任何消息到一个队列。实际上，生产者通常甚至不知道消息是否会被传递到哪个队列。**

相反，生产者可以只发送消息到一个**交换机(exchange)**。一个交换机只做很简单的一件事。一方面，它从生产者接收消息；另一方面，它推送消息到队列。交换机必须知道如何处理它收到的消息。消息应该追加到一个指定的队列？消息应该被追加到许多队列？或者消息应该被抛弃。消息处理的规则被定义为**交换机类型(exchange type)**。
<div align=center>
![producer](/images/rabbitmq/exchanges.png)
</div>

有几种可用的交换机类型:**direct**(直连),**topic**(话题),**headers**(头部)和**fanout**(扇出)。我们先重点讨论最后一个：fanout。我们创建一个这种类型的交换机，称之为logs：
``` golang
err = ch.ExchangeDeclare(
  "logs",   // name
  "fanout", // type
  true,     // durable
  false,    // auto-deleted
  false,    // internal
  false,    // no-wait
  nil,      // arguments
)
```
fanout交换机很简单。也许你已经从它的名字猜到了，它仅仅广播它收到的消息到它所知的所有的队列。这正是我们的记录器(logger)所需要的。

**列出所有的交换机**

要列出服务器上的交换，您可以运行一个非常有用的命令rabbitmqctl：
``` bash
sudo rabbitmqctl list_exchanges
```
在此结果中将有一些amq.*交换机和默认(未命名)交换机。这些是默认创建的，但目前您不太可能需要使用它们。

**默认交换机**

在教程的前几部分，虽然我们只需要知道交换机就能发送消息到队列。因为我们使用了通过空字符串("")识别的默认的交换机。
回想一下我们之前是怎么发布消息的：
``` golang
err = ch.Publish(
  "",     // exchange
  q.Name, // routing key
  false,  // mandatory
  false,  // immediate
  amqp.Publishing{
    ContentType: "text/plain",
    Body:        []byte(body),
})
```
**exchange**参数指定了交换机的名字。空字符串表示默认或者未命令(nameless)的交换机：消息通过**routing_key**(如果存在)传递到消息队列。

现在，我们可以发布到我们的命名交换机：
``` golang
err = ch.ExchangeDeclare(
  "logs",   // name
  "fanout", // type
  true,     // durable
  false,    // auto-deleted
  false,    // internal
  false,    // no-wait
  nil,      // arguments
)
failOnError(err, "Failed to declare an exchange")

body := bodyFrom(os.Args)
err = ch.Publish(
  "logs", // exchange
  "",     // routing key
  false,  // mandatory
  false,  // immediate
  amqp.Publishing{
          ContentType: "text/plain",
          Body:        []byte(body),
  })
```

### 临时队列(Temporary queues)

您可能还记得以前我们使用的是具有指定名称的队列(还记得hello和task_queue吗？).能够命名一个队列对我们很重要--我们需要worker也指定同样的队列。当您需要在消费者和生产者共享队列时，给一个队列命名很重要。

但是对我们的logger并不适用。我们想要收到所有的log消息，而不是所有消息的一个子集。我们也仅仅对当前的消息流感兴趣，对老消息流不感兴趣。为了解决这个问题，我们需要两个东西。

首先，每当连接到Rabbit时，我们需要一个新的空队列。我们可以用一个随机的名字创建一个啊队列。或者，最好让服务器给我们选个一个随即队列名。我们可以通过不提供**queue**参数给**queue_declare**来实现。

第二，一旦消费者连接关闭，队列也应该被删除。

在[amqp客户端](http://godoc.org/github.com/streadway/amqp)中，当我们将队列名称作为空字符串提供时，我们使用生成的名称创建一个非持久队列：
``` golang
q, err := ch.QueueDeclare(
  "",    // name
  false, // durable
  false, // delete when usused
  true,  // exclusive
  false, // no-wait
  nil,   // arguments
)
```
当方法返回时，队列实例返回RabbitMQ产生的一个随机队列名。例如，它可能看起来像amq.gen-JzTY20BRgKO-HjmUJj0wLg。

由于队列被声明为**exclusive**，当连接声明其已经关闭时，队列将被删除。

您可以学习关于****标志和其它队列属性的内容在[消息队列指南](https://www.rabbitmq.com/queues.html)。

### 绑定(Bindings)
<div align=center>
![producer](/images/rabbitmq/bindings.png)
</div>

我们已经创建了一个fanout类型的交换机和一个队列。现在我们需要告诉交换机发送消息到我们的对列。交换机和一个队列的对应关系我们称之为绑定(**binding**)。
``` golang
err = ch.QueueBind(
  q.Name, // queue name
  "",     // routing key
  "logs", // exchange
  false,
  nil
)
```
从现在开始，交换机logs将会追加消息到我们的队列。

**列出所有的绑定**
您可以用如下命令列出所有绑定，也许您已经猜到了。
``` bash
sudo rabbitmqctl list_bindings
```

### 综合(Putting it all together)
<div align=center>
![producer](/images/rabbitmq/python-three-overall.png)
</div>

发送消息的生产者程序和原来的教程看起来区别不大。最重要的改变是我们发布消息到交换机logs而不是那个未命名的交换机。当发送时，我们需要提供一个**routingKey**，但是其值会被**fanout**类型的交换机忽略。下面是**emit_log.go**的脚本：
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
                "logs",   // name
                "fanout", // type
                true,     // durable
                false,    // auto-deleted
                false,    // internal
                false,    // no-wait
                nil,      // arguments
        )
        failOnError(err, "Failed to declare an exchange")

        body := bodyFrom(os.Args)
        err = ch.Publish(
                "logs", // exchange
                "",     // routing key
                false,  // mandatory
                false,  // immediate
                amqp.Publishing{
                        ContentType: "text/plain",
                        Body:        []byte(body),
                })
        failOnError(err, "Failed to publish a message")

        log.Printf(" [x] Sent %s", body)
}

func bodyFrom(args []string) string {
        var s string
        if (len(args) < 2) || os.Args[1] == "" {
                s = "hello"
        } else {
                s = strings.Join(args[1:], " ")
        }
        return s
}
```
[emit_log.go源码](http://github.com/rabbitmq/rabbitmq-tutorials/blob/master/go/emit_log.go)

正如你看到的，在建立连接后，我们声明了交换机。这一步很必要，因为RabbitMQ禁止向不存在的交换机发消息。

如果没有队列绑定到交换机，消息将会丢失，但对我们来说还好；如果没有消费者在监听，我们可以安全的抛弃这条消息。

receive_logs.go源码:
``` golang
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

func main() {
        conn, err := amqp.Dial("amqp://guest:guest@localhost:5672/")
        failOnError(err, "Failed to connect to RabbitMQ")
        defer conn.Close()

        ch, err := conn.Channel()
        failOnError(err, "Failed to open a channel")
        defer ch.Close()

        err = ch.ExchangeDeclare(
                "logs",   // name
                "fanout", // type
                true,     // durable
                false,    // auto-deleted
                false,    // internal
                false,    // no-wait
                nil,      // arguments
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

        err = ch.QueueBind(
                q.Name, // queue name
                "",     // routing key
                "logs", // exchange
                false,
                nil)
        failOnError(err, "Failed to bind a queue")

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
[receive_logs.go源码](http://github.com/rabbitmq/rabbitmq-tutorials/blob/master/go/receive_logs.go)

如果您想要保存日志到一个文件，只需要打开控制台输入：
``` bash
$ go run receive_logs.go > logs_from_rabbit.log
```
如果您想在屏幕上看到日志，请生成一个新的终端并运行：
``` bash
go run receive_logs.go
```

发送日志，输入：
``` bash
go run emit_log.go
```

使用rabbitmqctl list_bindings，您可以验证代码是否实际创建了我们想要的绑定和队列。 运行两个receive_logs.go程序时，您应该看到类似的内容：
``` bash
sudo rabbitmqctl list_bindings
# => Listing bindings ...
# => logs    exchange        amq.gen-JzTY20BRgKO-HjmUJj0wLg  queue           []
# => logs    exchange        amq.gen-vso0PVvyiRIL2WoV3i48Yg  queue           []
# => ...done.
```
结果的解释很简单：来自交换机logs的数据转到两个具有服务器分配的名称的队列。而这正是我们的想要的。

要了解如何监听消息的子集，让我们继续[教程4](https://www.rabbitmq.com/tutorials/tutorial-four-go.html).


