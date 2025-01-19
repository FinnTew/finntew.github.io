---
title: 字节青训营 - [抖音电商项目] 使用 RabbitMQ 实现订单定时取消
description: 使用 RabbitMQ 死信交换机实现订单定时取消，及使用延时队列插件优化实现
slug: order-scheduled-cancel-by-rabbitmq
date: 2025-01-18T22:01:51+08:00
math: true
image:
tags:
  - 字节青训营
  - Golang
  - RabbitMQ
weight: 1
---

最近参与了字节跳动青训营寒假专场的后端方向，结营项目 "抖音电商" 订单服务中有一项定时取消订单的功能需求，在这里记录一些我的想法和 demo 实现。

## 一、前言

为了避免在用户在未确认(支付)的情况下一直占用商品库存，提高整体库存周转率，避免无效订单数据长时间驻留在缓存中，我们需要采用一些机制进行资源回收，在这里即为定时取消机制。我们为订单设置一个过期时间，当订单在过期时间内未确认支付时，自动取消订单释放库存。这一需求可以通过 RabbitMQ 消息队列实现。

## 二、实现方案

我们先梳理一下在用户创建订单后的整个流程：

![Image1](img1.png)

我们需要实现的部分即："定时取消订单" 子图部分，有如下两种实现方案：

### 1) 使用死信交换机配合死信队列实现

#### 一些介绍

1. 死信交换机(DLX)

   死信交换机即一个专门用于接收死信的交换机。当消息成为死信后，它们会被路由到指定的死信交换机。根据配置，交换机可以将这些死信消息路由到一个或多个死信队列中。
   
   **工作流程**：
   1. 创建队列时配置交换机及相关属性参数。
   2. 当消息在正常队列中因为一些原因(消息过期；达到最大重试次数；队列限制)无法被消费时，被标记为死信。
   3. 标记的死信根据配置发送到设置的死信交换机。
   4. 死信交换机根据路由规则将死信发送到对应的死信队列。
2. 死信队列(DLQ)
   
   死信队列即一个特殊的队列，用于存储那些不能被正常处理的消息(因为上文描述的原因标记为死信的消息)。

#### Demo 实现

参照上文我们发现，过期的消息会被标记为死信发送到死信交换机，在轮转到死信队列。

那么我们不难想到这里的第一个方案：创建一个死信交换机，过期的消息被发送到死信交换机，然后创建一个死信队列，接收过期的死信，最后创建一个消费者，消费死信队列中的消息，即可实现定时取消订单的功能。

然后就是 Coding 实现想法了。

这里先使用 Docker Compose 搭建一个 RabbitMQ 环境，方便后续的测试：

```yaml
version: '3'

services:
  rabbitmq:
    image: rabbitmq:3-management
    container_name: rabbitmq-test
    ports:
      - "5672:5672"
      - "15672:15672"
    environment:
      RABBITMQ_DEFAULT_USER: finn
      RABBITMQ_DEFAULT_PASS: finn
    volumes:
      - rabbitmq_data:/var/lib/rabbitmq

volumes:
  rabbitmq_data:
```

首先定义以下订单结构，方便后续的模拟：

```go
type Order struct {
	ID        string    `json:"id"`
	Status    string    `json:"status"`
	CreatedAt time.Time `json:"created_at"`
}

const (
	StatusPending = "pending"
	StatusSuccess = "success"
	StatusClosed  = "closed"
)
```

连接到 RabbitMQ：

```go
conn, err := amqp.Dial("amqp://finn:finn@localhost:5672/")
// ...
defer conn.Close()
ch, err := conn.Channel()
//...
defer ch.Close()
```

然后按照流程创建交换机、队列：

```go
func declareExchanges(ch *amqp.Channel) error {
	// 声明普通交换机
	err := ch.ExchangeDeclare(
		normalExchange,
		"direct",
		true,
		false,
		false,
		false,
		nil,
	)
	// ...

	// 声明延迟交换机
	err = ch.ExchangeDeclare(
		delayExchange,
		"direct",
		true,
		false,
		false,
		false,
		nil,
	)
	// ...

	// 声明死信交换机
	err = ch.ExchangeDeclare(
		deadLetterExchange,
		"direct",
		true,
		false,
		false,
		false,
		nil,
	)
	return err
}
```

```go
func declareQueues(ch *amqp.Channel) error {
	// 声明普通队列
	_, err := ch.QueueDeclare(
		normalQueue,
		true,
		false,
		false,
		false,
		nil,
	)
	//...

	// 绑定普通队列
	err = ch.QueueBind(
		normalQueue,
		normalKey,
		normalExchange,
		false,
		nil,
	)
	//...

	// 声明延迟队列
	args := amqp.Table{
		"x-dead-letter-exchange":    deadLetterExchange,
		"x-dead-letter-routing-key": deadLetterKey,
		"x-message-ttl":             int32(orderTTL.Milliseconds()),
	}

	_, err = ch.QueueDeclare(
		delayQueue,
		true,
		false,
		false,
		false,
		args,
	)
	//...

	// 绑定延迟队列
	err = ch.QueueBind(
		delayQueue,
		delayKey,
		delayExchange,
		false,
		nil,
	)
	//...

	// 声明死信队列
	_, err = ch.QueueDeclare(
		deadLetterQueue,
		true,
		false,
		false,
		false,
		nil,
	)
	//...

	// 绑定死信队列
	err = ch.QueueBind(
		deadLetterQueue,
		deadLetterKey,
		deadLetterExchange,
		false,
		nil,
	)
	return err
}
```

最后即消费者的实现：

```go
func consumeNormalQueue(ch *amqp.Channel) {
	msgs, err := ch.Consume(
		normalQueue,
		"",
		false,
		false,
		false,
		false,
		nil,
	)
	//...

	for msg := range msgs {
		var order Order
		err := json.Unmarshal(msg.Body, &order)
		if err != nil {
			//...
			msg.Ack(false)
			continue
		}

		// 改变订单状态，这里随机模拟
		if rand.Float32() < 0.3 {
			order.Status = StatusSuccess
		}
		
		//...

		// 轮转发送到延迟队列
		ctx := context.Background()
		publishOrder(ctx, ch, delayExchange, delayKey, order)

		msg.Ack(false)
	}
}

func consumeDeadLetterQueue(ch *amqp.Channel) {
	msgs, err := ch.Consume(
		deadLetterQueue,
		"",
		false,
		false,
		false,
		false,
		nil,
	)
	//...

	for msg := range msgs {
		var order Order
		err := json.Unmarshal(msg.Body, &order)
		if err != nil {
			//...
			msg.Ack(false)
			continue
		}

		// 检查订单状态
		if order.Status == StatusPending {
			order.Status = StatusClosed
			log.Printf("Order %s timeout, status changed to %s", order.ID, order.Status)
		} else {
			log.Printf("Order %s status is %s, no change needed", order.ID, order.Status)
		}

		msg.Ack(false)
	}
}
```

最后在主函数模拟用户创建订单即可完成整个流程：

```go
// 模拟创建订单
ctx := context.Background()
for i := 1; i <= 5; i++ {
    order := Order{
        ID:        fmt.Sprintf("ORDER_%d", i),
        Status:    StatusPending,
        CreatedAt: time.Now(),
    }

    // 发送到普通队列
    publishOrder(ctx, ch, normalExchange, normalKey, order)
    time.Sleep(2 * time.Second)
}

// 启动消费者
forever := make(chan struct{})

go consumeNormalQueue(ch)
go consumeDeadLetterQueue(ch)

<-forever
```

其中省略了一些不关键的错误处理和日志输出，最终效果如下：

```plaintext
2025/01/19 16:51:06 Order ORDER_1 timeout, status changed to closed
2025/01/19 16:51:06 Order ORDER_2 status is success, no change needed
2025/01/19 16:51:06 Order ORDER_3 status is success, no change needed
2025/01/19 16:51:06 Order ORDER_4 status is success, no change needed
2025/01/19 16:51:06 Order ORDER_5 status is success, no change needed
```

可以看到，订单在过期时间内未确认支付时，自动取消订单确实已经完成。

但是我们继续考虑这种实现可能会导致的问题：

1. 当订单满足超时条件需要取消时，我们需要轮转消息到死信队列，之后在死信队列中重新处理，这可能会造成延迟，导致实际取消时间超出预设的时间。
2. 大量订单消息轮转到死信队列中，造成消息积压，可能导致负载增加，系统性能下降。
3. 业务逻辑肉眼可见的变得复杂，不易于维护。

显然需要进行一些优化，这里引出我们的第二种实现方案：

### 2) 使用延时队列插件实现

RabbitMQ 提供了一个延时队列插件 `rabbitmq_delayed_message_exchange`。我们可以依赖这个插件实现定时取消。

#### 一些介绍

**延迟消息交换机**是一种特殊的交换机，允许用户将消息在指定的时间延迟后再发送到目标队列。这种机制适用于需要在未来某个时刻处理的消息场景，例如定时任务、延迟通知等。 

**工作流程**：
1. 创建延迟消息交换机时，指定其类型为 x-delayed-message，并设置所需的参数。 
2. 发布消息时，可以在消息的属性中设置延迟时间（以毫秒为单位）。 
3. 消息在指定的延迟时间后，交换机会将其路由到相应的目标队列。 
4. 消费者从目标队列中获取到延迟后的消息进行处理。

#### Demo 实现

我们首先在之前开启的 RabbitMQ 服务中引入这个插件：

1. 下载插件：
   ```bash
   mkdir -p ~/rabbitmq/plugins && wget -P ~/rabbitmq/plugins https://github.com/rabbitmq/rabbitmq-delayed-message-exchange/releases/download/v3.13.0/rabbitmq_delayed_message_exchange-3.13.0.ez
   ```
   
2. Copy 到容器：
   ```bash
   sudo docker cp ~/rabbitmq/plugins/rabbitmq_delayed_message_exchange-3.13.0.ez rabbitmq-test:/plugins
   ```
   
3. 启动插件：
   ```bash
   sudo docker exec -it rabbitmq-test rabbitmq-plugins enable rabbitmq_delayed_message_exchange
   ```
   
4. 重启服务：
   ```bash
   sudo docker restart rabbitmq-test
   ```
   
然后开始调整一下代码实现：

只需要修改一下创建队列和交换机的部分即可，其他部分不变：

```go
// 声明普通交换机
err := ch.ExchangeDeclare(
	normalExchange,
	"direct",
	true,
	false,
	false,
	false,
	nil, 
	)
// 声明延时交换机（使用 delayed-message-exchange 插件）
args := amqp.Table{
	"x-delayed-type": "direct",
}
err = ch.ExchangeDeclare(
	delayedExchange,
	"x-delayed-message",
	true,
	false,
	false,
	false,
	args, 
	)
// 声明普通队列
_, err = ch.QueueDeclare(
	normalQueue,
	true,
	false,
	false,
	false,
	nil, 
	)
// 绑定普通队列
err = ch.QueueBind(
	normalQueue,
	normalKey,
	normalExchange,
	false,
	nil,
	)
// 声明延时队列
_, err = ch.QueueDeclare(
	delayedQueue,
	true,
	false,
	false,
	false,
	nil,
	)
// 绑定延时队列
err = ch.QueueBind(
	delayedQueue,
	delayedKey,
	delayedExchange,
	false,
	nil,
	)
```

## 三、总结

| 比较项         | `rabbitmq_delayed_message_exchange`                                        | 死信队列和死信交换机                                  |
|-------------|------------------------------------------------------------------------|--------------------------------------------------|
| **超时取消延迟**  | 无需轮转消息，直接在预设时间后处理，避免延迟，精准控制取消时间                  | 需要轮转到死信队列，处理延迟可能导致实际取消时间超出预设时间     |
| **消息积压**    | 消息在延迟交换机中处理，避免积压，系统性能更稳定                              | 大量订单消息轮转到死信队列，可能导致消息积压，增加系统负载，影响性能 |
| **业务逻辑复杂性** | 逻辑简单，易于理解和维护，直观的延迟处理机制                              | 业务逻辑复杂，需处理死信队列重试和转发，增加维护成本和复杂度     |
| **系统性能**    | 高效处理，减少资源消耗，优化整体性能                                      | 可能因消息积压导致系统性能下降，影响其他业务流程                     |
| **配置和管理**   | 配置简单，无需额外设置死信队列，减少管理负担                                | 需要配置和管理死信队列及交换机，增加运维复杂度                      |
| **灵活性**     | 可以灵活设置不同的延迟时间，适应多种业务场景                                | 灵活性有限，重试逻辑和延迟处理需要额外设计                           |
