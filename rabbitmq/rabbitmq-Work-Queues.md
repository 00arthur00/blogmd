title: rabbitmq工作队列(Work Queues)
tags:
  - golang
  - rabbitmq
category: [rabbitmq]
date: 2018-09-18 19:40:36
---
[源自官方文档](https://www.rabbitmq.com/tutorials/tutorial-two-go.html)
**使用Go RabbitMQ客户端**
<div align=center>
![producer](/images/rabbitmq/python-two.png)
</div>

在第一篇教程中，我们编写了程序来发送和接收来自**命名队列**的消息。在这个中，我们将创建一个**工作队列(Work Queue)**，用于在多个worker之间分配耗时的任务。

工作队列（又称：**任务队列**）背后的主要思想是避免立即执行资源密集型任务并必须等待它完成。相反，我们安排任务稍后完成。 我们将**任务**封装为消息并将其发送到一个队列。在后台运行的工作进程将弹出任务并最终执行作业。 当您运行许多worker时，它们之间将共享任务。

这个概念在Web应用程序中特别有用，因为无法在短链接的HTTP请求窗口中处理复杂的任务。

### 准备(Preparation)
在本教程的前一部分中，我们发送了一条包含“Hello World！”的消息。现在我们将发送代表复杂任务的字符串。 我们没有现实世界的任务，比如要调整大小的图像或要渲染的pdf文件，所以让我们假装我们很忙-通过使用*time.Sleep*函数来假装。 我们将字符串中的点数来表示其复杂性;每个点都会占据一秒钟的“工作”。例如，*Hello...*表示任务将花费三秒钟。


我们将稍微修改前一个示例中的send.go代码，以允许从命令行发送任意消息。 该程序将任务调度到我们的工作队列，所以我们将其命名为*new_task.go*：
``` golang
body := bodyFrom(os.Args)
err = ch.Publish(
  "",           // exchange
  q.Name,       // routing key
  false,        // mandatory
  false,
  amqp.Publishing {
    DeliveryMode: amqp.Persistent,
    ContentType:  "text/plain",
    Body:         []byte(body),
  })
failOnError(err, "Failed to publish a message")
log.Printf(" [x] Sent %s", body)
```
我们旧的receive.go脚本还需要进行一些更改：它需要为消息体中的每个点伪造一秒钟的工作。 它将从队列中弹出消息并执行任务，所以我们称之为worker.go：
``` golang
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
    log.Printf("Received a message: %s", d.Body)
    dot_count := bytes.Count(d.Body, []byte("."))
    t := time.Duration(dot_count)
    time.Sleep(t * time.Second)
    log.Printf("Done")
  }
}()

log.Printf(" [*] Waiting for messages. To exit press CTRL+C")
<-forever
```
请注意，我们的假任务模拟了执行时间。

像在教程一中那样运行它们：
``` bash
# shell 1
go run worker.go
```
``` bash
# shell 2
go run new_task.go
```

### 循环分发(Round-robin dispatching)
使用**任务队列(Task Queue)**的一个优点是能够轻松地并行工作。 如果我们正在创建积压工作(blacklog of work)，我们可以添加更多worker，这样就可以轻松扩展。

首先，让我们尝试同时运行两个worker.go脚本。他们都会从队列中获取消息，但究竟如何呢？让我们来看看。

你需要打开三个console。两个将运行worker.go脚本，这两个console将成为我们的两个消费者-C1和C2。
``` bash
# shell 1
go run worker.go
# => [*] Waiting for messages. To exit press CTRL+C
```
``` bash
# shell 2
go run worker.go
# => [*] Waiting for messages. To exit press CTRL+C
```
在第三个console中，我们将发布新任务。 启动消费者后，您可以发布一些消息：
``` bash
# shell 3
go run new_task.go First message.
go run new_task.go Second message..
go run new_task.go Third message...
go run new_task.go Fourth message....
go run new_task.go Fifth message.....
```
让我们看看传给(deliver)我们woker的是什么：
``` bash
# shell 1
go run worker.go
# => [*] Waiting for messages. To exit press CTRL+C
# => [x] Received 'First message.'
# => [x] Received 'Third message...'
# => [x] Received 'Fifth message.....'
```
``` bash
# shell 2
go run worker.go
# => [*] Waiting for messages. To exit press CTRL+C
# => [x] Received 'Second message..'
# => [x] Received 'Fourth message....'
```
默认情况下，RabbitMQ将按顺序将每条消息发送给下一个消费者。 平均而言，每个消费者将获得相同数量的消息。 这种分发消息的方式称为循环法(round-robin)。 可以用三个或更多worker一起试试。

### 消息确认(Message acknowledgment)
执行任务可能需要几秒钟。您可能想知道如果其中一个消费者开始执行长时间任务并且仅在部分完成时挂掉会发生什么。使用我们当前的代码，一旦RabbitMQ向客户发送消息，它立即将其标记为删除。在这种情况下，如果你杀死一个worker，我们将丢失它刚刚处理的消息。我们还将丢失分发给这个特定worker但尚未处理的所有消息。

但我们不想失去任何任务。如果worker挂掉了，我们希望将任务交付给另一个worker。

为了确保消息永不丢失，RabbitMQ支持[**消息确认**](https://www.rabbitmq.com/confirms.html)。消费者发回ack(确认)告诉RabbitMQ已经接收并处理了特定消息，RabbitMQ可以随意删除它。

如果消费者(consumer和worker是同一个意思)挂掉（其信道关闭，连接关闭或TCP连接丢失）而不发送确认，RabbitMQ会理解为消息未完全处理并将其重新排队。如果其他消费者同时在线，则会迅速将其重新发送(redeliver)给其他消费者。这样你就可以确保没有消息丢失，即使woker偶尔会挂掉。

**RabbitMQ没有任何消息超时**;当消费者挂掉时，RabbitMQ将重新发送消息。即使处理消息需要非常长的时间，也没关系。

在本教程中，我们将使用手动消息确认，通过为“auto-ack”参数传递false，然后一旦完成了任务,worker使用d.Ack(false)发送适当的确认（此确认单向传递）。
``` golang
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
    log.Printf("Received a message: %s", d.Body)
    dot_count := bytes.Count(d.Body, []byte("."))
    t := time.Duration(dot_count)
    time.Sleep(t * time.Second)
    log.Printf("Done")
    d.Ack(false)
  }
}()

log.Printf(" [*] Waiting for messages. To exit press CTRL+C")
<-forever
```
使用此代码，我们可以确定即使您在处理消息时使用CTRL+C杀死一个worker，也不会丢失任何内容。worker挂掉后不久，所有未经确认的消息将被重新传递。

确认必须在收到递送的消息的同一信道上发送。 尝试使用不同的信道进行确认将导致信道级协议异常。有关[**确认的文档指南**](https://www.rabbitmq.com/confirms.html)，请参阅了解更多信息。

**忘记确认**
忘记ack是一个常见的错误。 这是一个简单的错误，但后果是严重的。当您的客户端退出时，消息将被重新传递（这可能看起来像随机重新传递），但RabbitMQ将会占用越来越多的内存，因为它无法释放任何未经确认的消息。

为了调试这种错误，您可以使用*rabbitmqctl*来打印*messages_unacknowledged*字段：
``` bash
sudo rabbitmqctl list_queues name messages_ready messages_unacknowledged
```
在Windows上，删除sudo：
``` bash
rabbitmqctl.bat list_queues name messages_ready messages_unacknowledged
```

### 消息持久性(Message durability)
我们已经学会了如何确保即使消费者死亡，任务也不会丢失。但是如果RabbitMQ服务器停止，我们的任务仍然会丢失。

当RabbitMQ退出或崩溃时，它将忘记队列和消息，除非你告诉它不要。确保消息不会丢失需要做两件事：我们需要将队列和消息都标记为持久。

首先，我们需要确保RabbitMQ永远不会丢失我们的队列。 为此，我们需要声明它是**持久的(durable)**：
``` golang
q, err := ch.QueueDeclare(
  "hello",      // name
  true,         // durable
  false,        // delete when unused
  false,        // exclusive
  false,        // no-wait
  nil,          // arguments
)
failOnError(err, "Failed to declare a queue")
```
虽然此命令本身是正确的，但它在我们当前的设置中不起作用。那是因为我们已经定义了一个名为*hello*的队列，这个队列不是持久的。RabbitMQ不允许您使用不同的参数重新定义现有队列，并将向尝试执行此操作的任何程序返回错误。但是有一个快速的解决方法-让我们声明一个具有不同名称的队列，例如*task_queue*：
``` golang
q, err := ch.QueueDeclare(
  "task_queue", // name
  true,         // durable
  false,        // delete when unused
  false,        // exclusive
  false,        // no-wait
  nil,          // arguments
)
failOnError(err, "Failed to declare a queue")
```
生产者和消费者代码都需要更改这个*durable*参数。

此时,我们确信即使RabbitMQ重新启动，*task_queue*队列也不会丢失。现在我们需要将消息标记为持久的(persistent)-使用*amqp.Publishing*的*amqp.Persistent*参数。

``` golang
err = ch.Publish(
  "",           // exchange
  q.Name,       // routing key
  false,        // mandatory
  false,
  amqp.Publishing {
    DeliveryMode: amqp.Persistent,
    ContentType:  "text/plain",
    Body:         []byte(body),
})
```

**消息持久化需要注意**
将消息标记为持久的(persistent)并不能100%保证此消息不会丢失。尽管RabbitMQ被告知需要将此消息存入磁盘，仍然有一个短的窗口期，在此窗口期内RabbitMQ接收了消息但为存入磁盘。并且RabbitMQ并不是对每个消息都做*fsync*操作-消息可能只是在缓存内并没有真正的写入磁盘。这种持久化的保障性并不强，但是对我们简单的人物队列(task queue)已经足够了。 如果您想要更强的保障，可以使用[**发布者确认（publisher confirms)**](https://www.rabbitmq.com/confirms.html)

### 公平分发(Fair dispatch)

您也许注意到了分发并没有如我们想要的那样工作。例如，在两个woker的情况下，当奇数消息的工作量比较大，而偶数消息的工作量比较小。一个worker会一直比较忙，而另一个几乎不干什么活。然而，RabbitMQ对此一无所知，仍然会均匀地发送消息。

发生这种情况是因为RabbitMQ只是在消息进入队列时调度消息。它不会查看消费者未确认消息的数量。它只是盲目地向第n个消费者发送每个第n个消息。


<div align=center>
![producer](/images/rabbitmq/prefetch-count.png)
</div>

为了避免(defeat)这种情况，我们可以设置值为1的预取计数。这告诉RabbitMQ一次不要向一个worker发送超过一条消息。 或者，换句话说，在处理并确认前一个消息之前，不要向worker发送新消息。 相反，它会将消息发送给下一个仍然不忙的worker。

**注意队列的大小**
如果所有的worker都在忙，您的队列将会被填满。您需要注意这种情况，也许添加更多的worker或者才用其他策略能解决。

### 综合(Putting it all together)
*new_task.go*的最终代码:
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

        q, err := ch.QueueDeclare(
                "task_queue", // name
                true,         // durable
                false,        // delete when unused
                false,        // exclusive
                false,        // no-wait
                nil,          // arguments
        )
        failOnError(err, "Failed to declare a queue")

        body := bodyFrom(os.Args)
        err = ch.Publish(
                "",           // exchange
                q.Name,       // routing key
                false,        // mandatory
                false,
                amqp.Publishing{
                        DeliveryMode: amqp.Persistent,
                        ContentType:  "text/plain",
                        Body:         []byte(body),
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
[**new_task.go源码位置在这里**](http://github.com/rabbitmq/rabbitmq-tutorials/blob/master/go/new_task.go)

worker.go源码:
``` golang
package main

import (
        "bytes"
        "fmt"
        "github.com/streadway/amqp"
        "log"
        "time"
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

        q, err := ch.QueueDeclare(
                "task_queue", // name
                true,         // durable
                false,        // delete when unused
                false,        // exclusive
                false,        // no-wait
                nil,          // arguments
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
                        log.Printf("Received a message: %s", d.Body)
                        dot_count := bytes.Count(d.Body, []byte("."))
                        t := time.Duration(dot_count)
                        time.Sleep(t * time.Second)
                        log.Printf("Done")
                        d.Ack(false)
                }
        }()

        log.Printf(" [*] Waiting for messages. To exit press CTRL+C")
        <-forever
}
```
[**worker.go源码在这里**](http://github.com/rabbitmq/rabbitmq-tutorials/blob/master/go/worker.go)

使用消息确认和预取计数，您可以设置工作队列。 即使RabbitMQ重新启动，持久性选项也可以使任务生效。

有关amqp.Channel方法和消息属性的更多信息，您可以浏览[**amqp API文档**](http://godoc.org/github.com/streadway/amqp)。

现在我们去教程3学习如何递送一个消息到多个消费者。