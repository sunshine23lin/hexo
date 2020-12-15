---
title: RocketMQ
date: 2020-12-01 08:30:33
categories: 消息队列
tags: MQ
---

#  MQ有哪些作用?

##  异步化提高系统性能

##  系统解耦

##  高并发削峰

#  MQ技术选型

![image-20201203142559984](https://jameslin23.gitee.io/2020/12/01/RocketMQ/image-20201203142559984.png)

#  RocketMQ核心原理

##  NameServer

>  是整个消息队列中的状态服务器，集群的各个组件通过它来了解全局的信息。每个broker都会注册到所有的NameServer上，Broker和NameServer采用心跳机制,Broker会每隔30s给所有NameServer发送心跳,告诉每个nameServer自己还活着，然后NameServer会每隔10s运行一个任务，去检测一下各个Broker的最近一次心跳时间，如果某个Broker超过120s都没发心跳了，就认为这个Broker已经挂掉了。
>
> 发送消息。通过nameServer去获取路由信息，就知道集群有哪些Broker等消息
> 消费消息，也会到NameServer获取路由信息，去找到对应的Broker获取消息

##  Broker

真正存储数据的地方。

###  CommitLog

> 消息内容原文的存储文件，同Kafka一样，消息是变长的，顺序写入
>
> 生成规则:
>
> 每个文件的默认1G =1024 * 1024 * 1024，commitlog的文件名fileName，名字长度为20位，左边补零，剩余为起始偏移量；比如00000000000000000000代表了第一个文件，起始偏移量为0，文件大小为1G=1 073 741 824Byte；当这个文件满了，第二个文件名字为00000000001073741824，起始偏移量为1073741824, 消息存储的时候会顺序写入文件，当文件满了则写入下一个文件

###  ConsumeQueue

> ConsumeQueue中并不需要存储消息的内容，而存储的是消息在CommitLog中的offset。也就是说，ConsumeQueue其实是CommitLog的一个索引文件。
>
> $HOME/store/consumequeue/{topic}/{queueId}/{fileName}

ConsumeQueue是定长的结构，每1条记录固定的20个字节。很显然，Consumer消费消息的时候，要读2次：先读ConsumeQueue得到offset，再读CommitLog得到消息内容

![image-20201202202251399](https://jameslin23.gitee.io/2020/12/01/RocketMQ/image-20201202202251399.png)

  **ConsumeQueue的作用**

1. 通过broker保存的offset可以在ConsumeQueue中获取消息，从而快速的定位到commitLog的消息位置
2. 过滤tag是也是通过遍历ConsumeQueue来实现的（先比较hash(tag)符合条件的再到consumer比较tag原文）
3. 并且ConsumeQueue还能保存于操作系统的PageCache进行缓存提升检索性能

>  当你的Broker收到一条消息写入了CommitLog之后，其实他同时会将这条消息在CommitLog中的物理位置，也就是一个文
> 件偏移量，就是一个offset，写入到这条消息所属的MessageQueue对应的ConsumeQueue文件中去。

###  异步刷盘和同步刷盘

**Broker是基于OS操作系统的PageCache和顺序写两个机制，来提升写入CommitLog文件的性能的**

![img](https://jameslin23.gitee.io/2020/12/01/RocketMQ/15762560-d88bf85df93dce41.png)

#### 异步刷盘

> 首先Broker是以顺序的方式将消息写入CommitLog磁盘文件的，也就是每次写入就是在文件末尾追加一条数据就可以了，对文件进行顺序写的性能要比对文件随机写的性能提升很多
>
> 数据写入CommitLog文件的时候，其实不是直接写入底层的物理磁盘文件的，而是先进入OS的PageCache内存缓存中，然后后
> 续由OS的后台线程选一个时间，异步化的将OS PageCache内存缓冲中的数据刷入底层的磁盘文件
>
> 所以在这样的优化之下，采用**磁盘文件顺序写+OS PageCache写入+OS异步刷盘**的策略，基本上可以让消息写入CommitLog的性能
> 跟你直接写入内存里是差不多的，所以正是如此，才可以让Broker高吞吐的处理每秒大量的消息写入。

**存在问题**

> 在上述的异步刷盘模式下，生产者把消息发送给Broker，Broker将消息写入OS PageCache中，就直接返回ACK给生产者了。
>
> 生产者认为消息写入成功了，但是实际上那条消息此时是在Broker机器上的os cache中的，如果此时Broker直
> 接宕机，可能会有数据丢失的风险。

####  同步刷盘

>  生产者发送一条消息出去，broker收到了消息，必须直接强制把这个消息刷入底层的物理磁盘文件中，然后才会返回ack给producer，此时你才知道消息写入成功了

**优缺点**

> 吞吐量低，但是消息不容易丢失。



##  Producer

rocketmq支持三种发送消息的方式，分别是同步发送（sync），异步发送（async）和直接发送（oneway）

###  同步发送（Sync）

> 这种只有在消息完全发送完成之后才返回结果，此方法需要同步等待发送结果的时间为代价
>
> 这种方式具有内部重试机制,即在主动声明本次发送失败之前，内部实现将重试一定的次数，默认为2次（DefaultMQProducer＃setRetryTimesWhenSendFailed（））发送结果存在同一个消息可能被多次发送给broker,这里需要应用开发者在自己消费端处理。

~~~java

public class SyncProducer {
	public static void main(String[] args) throws Exception {
    	// 实例化消息生产者Producer
        DefaultMQProducer producer = new DefaultMQProducer("please_rename_unique_group_name");
    	// 设置NameServer的地址
    	producer.setNamesrvAddr("localhost:9876");
    	// 启动Producer实例
        producer.start();
    	for (int i = 0; i < 100; i++) {
    	    // 创建消息，并指定Topic，Tag和消息体
    	    Message msg = new Message("TopicTest" /* Topic */,
        	"TagA" /* Tag */,
        	("Hello RocketMQ " + i).getBytes(RemotingHelper.DEFAULT_CHARSET) /* Message body */
        	);
        	// 发送消息到一个Broker
            SendResult sendResult = producer.send(msg);
            // 通过sendResult返回消息是否成功送达
            System.out.printf("%s%n", sendResult);
    	}
    	// 如果不再发送消息，关闭Producer实例。
    	producer.shutdown();
    }
}
~~~

###  异步发送（async）

> 异步消息通常在对响应时间敏感的业务场景，即发送端不能容忍长时间等待Broker的响应,异步模式也在内部实现了重试机制，默认次数为2次(setRetryTimesWhenSendAsyncFailed()

~~~java
public class AsyncProducer {
	public static void main(String[] args) throws Exception {
    	// 实例化消息生产者Producer
        DefaultMQProducer producer = new DefaultMQProducer("please_rename_unique_group_name");
    	// 设置NameServer的地址
        producer.setNamesrvAddr("localhost:9876");
    	// 启动Producer实例
        producer.start();
        producer.setRetryTimesWhenSendAsyncFailed(0);
    	for (int i = 0; i < 100; i++) {
                final int index = i;
            	// 创建消息，并指定Topic，Tag和消息体
                Message msg = new Message("TopicTest",
                    "TagA",
                    "OrderID188",
                    "Hello world".getBytes(RemotingHelper.DEFAULT_CHARSET));
                // SendCallback接收异步返回结果的回调
                producer.send(msg, new SendCallback() {
                    @Override
                    public void onSuccess(SendResult sendResult) {
                        System.out.printf("%-10d OK %s %n", index,
                            sendResult.getMsgId());
                    }
                    @Override
                    public void onException(Throwable e) {
      	              System.out.printf("%-10d Exception %s %n", index, e);
      	              e.printStackTrace();
                    }
            	});
    	}
    	// 如果不再发送消息，关闭Producer实例。
    	producer.shutdown();
    }
}
~~~

###  直接发送(sendOneway)

>  这种方式主要用在不特别关心发送结果的场景，例如日志发送。

~~~java

public class OnewayProducer {
	public static void main(String[] args) throws Exception{
    	// 实例化消息生产者Producer
        DefaultMQProducer producer = new DefaultMQProducer("please_rename_unique_group_name");
    	// 设置NameServer的地址
        producer.setNamesrvAddr("localhost:9876");
    	// 启动Producer实例
        producer.start();
    	for (int i = 0; i < 100; i++) {
        	// 创建消息，并指定Topic，Tag和消息体
        	Message msg = new Message("TopicTest" /* Topic */,
                "TagA" /* Tag */,
                ("Hello RocketMQ " + i).getBytes(RemotingHelper.DEFAULT_CHARSET) /* Message body */
        	);
        	// 发送单向消息，没有任何返回结果
        	producer.sendOneway(msg);
 
    	}
    	// 如果不再发送消息，关闭Producer实例。
    	producer.shutdown();
    }
}
~~~

###  顺序消息发送

> 消息有序指的是可以按照消息的发送顺序来消费(FIFO)。RocketMQ可以严格的保证消息有序，可以分为分区有序或者全局有序。 顺序消费的原理解析，在默认的情况下消息发送会采取Round Robin轮询方式把消息发送到不同的queue(分区队列)；而消费消息的时候从多个queue上拉取消息，这种情况发送和消费是不能保证顺序。
>
> 要保证发送与消费有序，那同一类型的消息-发送到同一个队列当中，同一个队列只能一个线程去处理，即同一个消费者。
>
> 面用订单进行分区有序的示例。一个订单的顺序流程是：创建、付款、推送、完成。订单号相同的消息会被先后发送到同一个队列中，消费时，同一个OrderId获取到的肯定是同一个队列。

~~~Java
import org.apache.rocketmq.client.producer.DefaultMQProducer;
import org.apache.rocketmq.client.producer.MessageQueueSelector;
import org.apache.rocketmq.client.producer.SendResult;
import org.apache.rocketmq.common.message.Message;
import org.apache.rocketmq.common.message.MessageQueue;
 
import java.text.SimpleDateFormat;
import java.util.ArrayList;
import java.util.Date;
import java.util.List;
 
/**
* Producer，发送顺序消息
*/
public class Producer {
 
   public static void main(String[] args) throws Exception {
       DefaultMQProducer producer = new DefaultMQProducer("please_rename_unique_group_name");
 
       producer.setNamesrvAddr("127.0.0.1:9876");
 
       producer.start();
 
       String[] tags = new String[]{"TagA", "TagC", "TagD"};
 
       // 订单列表
       List<OrderStep> orderList = new Producer().buildOrders();
 
       Date date = new Date();
       SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
       String dateStr = sdf.format(date);
       for (int i = 0; i < 10; i++) {
           // 加个时间前缀
           String body = dateStr + " Hello RocketMQ " + orderList.get(i);
           Message msg = new Message("TopicTest", tags[i % tags.length], "KEY" + i, body.getBytes());
 
           SendResult sendResult = producer.send(msg, new MessageQueueSelector() {
               @Override
               public MessageQueue select(List<MessageQueue> mqs, Message msg, Object arg) {
                   Long id = (Long) arg;  //根据订单id选择发送queue
                   long index = id % mqs.size();
                   return mqs.get((int) index);
               }
           }, orderList.get(i).getOrderId());//订单id
 
           System.out.println(String.format("SendResult status:%s, queueId:%d, body:%s",
               sendResult.getSendStatus(),
               sendResult.getMessageQueue().getQueueId(),
               body));
       }
 
       producer.shutdown();
   }
~~~

~~~Java
 /**
    * 订单的步骤
    */
   private static class OrderStep {
       private long orderId;
       private String desc;
 
       public long getOrderId() {
           return orderId;
       }
 
       public void setOrderId(long orderId) {
           this.orderId = orderId;
       }
 
       public String getDesc() {
           return desc;
       }
 
       public void setDesc(String desc) {
           this.desc = desc;
       }
 
       @Override
       public String toString() {
           return "OrderStep{" +
               "orderId=" + orderId +
               ", desc='" + desc + '\'' +
               '}';
       }
   }
 
   /**
    * 生成模拟订单数据
    */
   private List<OrderStep> buildOrders() {
       List<OrderStep> orderList = new ArrayList<OrderStep>();
 
       OrderStep orderDemo = new OrderStep();
       orderDemo.setOrderId(15103111039L);
       orderDemo.setDesc("创建");
       orderList.add(orderDemo);
 
       orderDemo = new OrderStep();
       orderDemo.setOrderId(15103111065L);
       orderDemo.setDesc("创建");
       orderList.add(orderDemo);
 
       orderDemo = new OrderStep();
       orderDemo.setOrderId(15103111039L);
       orderDemo.setDesc("付款");
       orderList.add(orderDemo);
 
       orderDemo = new OrderStep();
       orderDemo.setOrderId(15103117235L);
       orderDemo.setDesc("创建");
       orderList.add(orderDemo);
 
       orderDemo = new OrderStep();
       orderDemo.setOrderId(15103111065L);
       orderDemo.setDesc("付款");
       orderList.add(orderDemo);
 
       orderDemo = new OrderStep();
       orderDemo.setOrderId(15103117235L);
       orderDemo.setDesc("付款");
       orderList.add(orderDemo);
 
       orderDemo = new OrderStep();
       orderDemo.setOrderId(15103111065L);
       orderDemo.setDesc("完成");
       orderList.add(orderDemo);
 
       orderDemo = new OrderStep();
       orderDemo.setOrderId(15103111039L);
       orderDemo.setDesc("推送");
       orderList.add(orderDemo);
 
       orderDemo = new OrderStep();
       orderDemo.setOrderId(15103117235L);
       orderDemo.setDesc("完成");
       orderList.add(orderDemo);
 
       orderDemo = new OrderStep();
       orderDemo.setOrderId(15103111039L);
       orderDemo.setDesc("完成");
       orderList.add(orderDemo);
 
       return orderList;
   }
}
~~~

###  发送延迟消息

> 现在RocketMq并不支持任意时间的延时，需要设置几个固定的延时等级，从1s到2h分别对应着等级1到18
>
> private String messageDelayLevel = "1s 5s 10s 30s 1m 2m 3m 4m 5m 6m 7m 8m 9m 10m 20m 30m 1h 2h";

~~~java
import org.apache.rocketmq.client.producer.DefaultMQProducer;
import org.apache.rocketmq.common.message.Message;
 
public class ScheduledMessageProducer {
   public static void main(String[] args) throws Exception {
      // 实例化一个生产者来产生延时消息
      DefaultMQProducer producer = new DefaultMQProducer("ExampleProducerGroup");
      // 启动生产者
      producer.start();
      int totalMessagesToSend = 100;
      for (int i = 0; i < totalMessagesToSend; i++) {
          Message message = new Message("TestTopic", ("Hello scheduled message " + i).getBytes());
          // 设置延时等级3,这个消息将在10s之后发送(现在只支持固定的几个时间,详看delayTimeLevel)
          message.setDelayTimeLevel(3);
          // 发送消息
          producer.send(message);
      }
       // 关闭生产者
      producer.shutdown();
  }
}
~~~

###  发送批量消息

> 批量发送消息能显著提高传递小消息的性能。限制是这些批量消息应该有相同的topic，相同的waitStoreMsgOK，而且不能是延时消息。此外，这一批消息的总大小不应超过4MB。

~~~java
String topic = "BatchTest";
List<Message> messages = new ArrayList<>();
messages.add(new Message(topic, "TagA", "OrderID001", "Hello world 0".getBytes()));
messages.add(new Message(topic, "TagA", "OrderID002", "Hello world 1".getBytes()));
messages.add(new Message(topic, "TagA", "OrderID003", "Hello world 2".getBytes()));
try {
   producer.send(messages);
} catch (Exception e) {
   e.printStackTrace();
   //处理error
}
~~~

###  生产者常用设置参数

~~~Java
// 初始化 设置相关参数
            defaultMQProducer = new DefaultMQProducer(producerGroupName); 
             // 设置nameSever地址
            defaultMQProducer.setNamesrvAddr(namesrvAddr);
             // 实例名
            defaultMQProducer.setInstanceName(instanceName);
            // 异步发送失败重发次数，默认2次
            defaultMQProducer.setRetryTimesWhenSendAsyncFailed(10);
            // 同步发送失败重发次数，默认2次
            defaultMQProducer.setRetryTimesWhenSendFailed(10);
            // 消息没有存储成功是否发送到另外一个broker
            defaultMQProducer.setRetryAnotherBrokerWhenNotStoreOK(true);
            // 发送超时时间
            defaultMQProducer.setSendMsgTimeout(5000);
            defaultMQProducer.start();
~~~

##  Consumer

###  Push模式

> 由于消息中间件主动地将消息推送给消费者，可以尽可能实时地将消息发送给消费者进行消费。但是消费者的处理消息的能力较弱时，比如(消费者端业务系统处理一条流程比较复杂，导致消费比较耗时)，而MQ不断地向消费者Push消息，消费者端缓冲区可能会溢出，导致异常。

> **Push模式实际上在内部还是使用的Pull方式实现的，通过Pull不断地轮询Broker获取消息，当不存在新消息时，Broker端会挂起Pull请求，直到有新消息产生才取消挂起，返回新消息。**
>
> 需要返回消费状态，MQ内部维护Offset

###  Pull模式

> 由消费者客户端主动向消息中间件（MQ消息服务器代理）拉取消息；采用Pull方式，如何设置Pull消息的频率需要重点去考虑，举个例子来说，可能1分钟内连续来了1000条消息，然后2小时内没有新消息产生（概括起来说就是**“消息延迟与忙等待”**）。如果每次Pull的时间间隔比较久，会增加消息的延迟，即消息到达消费者的时间加长，MQ中消息的堆积量变大；若每次Pull的时间间隔较短，但是在一段时间内MQ中并没有任何消息可以消费，那么会产生很多无效的Pull请求的RPC开销，影响MQ整体的网络性能；

> **消息消费的偏移量需要Consumer端自己去维护**

###  集群模式&广播模式

> 集群模式：一条消息，在同一个消费者组中，只有一个消费实例消费。
>
> 广播模式:   一条消息，在同一个消费者组中,全部消费实例都会消费。

###  消费者与队列关系

消费者==队列数

> 一个消费负责一个队列

消费者>队列数

> 一个消费者负责一个队列，剩余消费者空闲

消费者<队列数

> 队列数均摊所有消费者

###  消费重复问题

**Redis**

> 给消息分配一个全局id,只要消费过该消息，将<id,message>以K-V形式写入redis。消费者开始消费之前，先去redis中查询有没有消费记录即可

###  常用的命令

1. 查看所有topic

   > sh mqadmin topicList -n localhost:9876

2. 查看clustername

   > sh mqadmin clusterList -n localhost:9876

3. 查看所有消费组

   > sh mqadmin consumerProgress -n localhost:9876

4. 查看消费组的发送、接收、阻塞情况

   > sh mqadmin consumerProgress -g groupname -n localhost:9876

5. 删除topic

   > sh mqadmin deleteTopic -n localhost:9876 -c cluster-bus -t topicname

6. 删除group

   > sh mqadmin deleteSubGroup -c topicname -g groupname -n localhost:9876

7. 新增增或修改topic

   > sh mqadmin updateTopic -c cluster-bus -n localhost:9876 -o true -p 6 -w 50 -r 50 -t topicName

8. 新增或修改消费组

   > sh mqadmin updateSubGroup -n localhost:9876 -c cluster-bus -g groupname

9. 查看topic配置属性

   > sh mqadmin topicRoute -n localhost:9876 -t topicname

10. 根据消息Key查询消息

    > sh mqadmin queryMsgByKey -n localhost:9876 -t topicname -k key

11. 重置消费位点

    > sh mqadmin resetOffsetByTime -n localhost:9876 -g groupname -t topicname -s now
