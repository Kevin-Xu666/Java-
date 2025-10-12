# RocketMQ

https://www.bilibili.com/video/BV1L4411y7mn/

## 核心功能

### MQ介绍

1. 为什么要用MQ

   消息队列是一种“先进先出”的数据结构

   <img src="img\queue1.png" style="zoom:50%;" />

   其应用场景主要包含以下3个方面

   * 应用解耦

   系统的耦合性越高，容错性就越低。以电商应用为例，用户创建订单后，如果耦合调用库存系统、物流系统、支付系统，任何一个子系统出了故障或者因为升级等原因暂时不可用，都会造成下单操作异常，影响用户使用体验。

   <img src="D:\Desktop\Java学习个人笔记整理\13 RocketMQ.assets\解耦1.png" style="zoom:80%;" />

   使用消息队列解耦合，系统的耦合性就会提高了。比如物流系统发生故障，需要几分钟才能来修复，在这段时间内，物流系统要处理的数据被缓存到消息队列中，用户的下单操作正常完成。当物流系统回复后，补充处理存在消息队列中的订单消息即可，终端系统感知不到物流系统发生过几分钟故障。

   <img src="D:\Desktop\Java学习个人笔记整理\13 RocketMQ.assets\解耦2.png" style="zoom:80%;" />

   * 流量削峰

   <img src="D:\Desktop\Java学习个人笔记整理\13 RocketMQ.assets\mq-5.png" style="zoom:80%;" />

   应用系统如果遇到系统请求流量的瞬间猛增，有可能会将系统压垮。有了消息队列可以将大量请求缓存起来，分散到很长一段时间处理，这样可以大大提到系统的稳定性和用户体验。

   <img src="D:\Desktop\Java学习个人笔记整理\13 RocketMQ.assets\mq-6.png" style="zoom:80%;" />

   一般情况，为了保证系统的稳定性，如果系统负载超过阈值，就会阻止用户请求，这会影响用户体验，而如果使用消息队列将请求缓存起来，等待系统处理完毕后通知用户下单完毕，这样总不能下单体验要好。

   <u>处于经济考量目的：</u>

   业务系统正常时段的QPS如果是1000，流量最高峰是10000，为了应对流量高峰配置高性能的服务器显然不划算，这时可以使用消息队列对峰值流量削峰

   * 数据分发

   <img src="D:\Desktop\Java学习个人笔记整理\13 RocketMQ.assets\mq-1.png" style="zoom:80%;" />

   通过消息队列可以让数据在多个系统更加之间进行流通。数据的产生方不需要关心谁来使用数据，只需要将数据发送到消息队列，数据使用方直接在消息队列中直接获取数据即可

   <img src="D:\Desktop\Java学习个人笔记整理\13 RocketMQ.assets\image-20251004184012867.png" alt="image-20251004184012867" style="zoom:80%;" />

2. MQ的优点和缺点

   优点：解耦、削峰、数据分发

   缺点包含以下几点：

   * 系统可用性降低

     系统引入的外部依赖越多，系统稳定性越差。一旦MQ宕机，就会对业务造成影响。

     如何保证MQ的高可用？

   * 系统复杂度提高

     MQ的加入大大增加了系统的复杂度，以前系统间是同步的远程调用，现在是通过MQ进行异步调用。

     如何保证消息没有被重复消费？怎么处理消息丢失情况？那么保证消息传递的顺序性？

   * 一致性问题

     A系统处理完业务，通过MQ给B、C、D三个系统发消息数据，如果B系统、C系统处理成功，D系统处理失败。

     如何保证消息数据处理的一致性？

3. 各种MQ产品的比较

   常见的MQ产品包括Kafka、ActiveMQ、RabbitMQ、RocketMQ。 

   <img src="D:\Desktop\Java学习个人笔记整理\13 RocketMQ.assets\MQ比较.png" style="zoom:40%;" />

### RocketMQ角色介绍

* Producer：消息的发送者；举例：发信者
* Consumer：消息接收者；举例：收信者
* Broker：暂存和传输消息；举例：邮局
* NameServer：管理Broker；举例：各个邮局的管理机构
* Topic：区分消息的种类；一个发送者可以发送消息给一个或者多个Topic；一个消息的接收者可以订阅一个或者多个Topic消息
* Message Queue：相当于是Topic的分区；用于并行发送和接收消息

Broker是实际存放消息的地方，Name Server相当于一个注册中心，生产者需要先从Name Server处知道发送到哪个Broker，消费者也需要从Name Server处获知从哪个Broker处消费。

<img src="D:\Desktop\Java学习个人笔记整理\13 RocketMQ.assets\RocketMQ角色.jpg" style="zoom:50%;" />



### 高可用集群搭建

####  集群搭建方式

1. 集群特点

   - NameServer是一个几乎无状态节点，可集群部署，节点之间无任何信息同步。由于Broker会向Name Server中每一个节点上报信息，因此Name Server集群中每个节点地位等同。

   - Broker部署相对复杂，Broker分为Master与Slave，一个Master可以对应多个Slave，但是一个Slave只能对应一个Master，Master与Slave的对应关系通过指定相同的BrokerName，不同的BrokerId来定义，BrokerId为0表示Master，非0表示Slave。Master也可以部署多个。每个Broker与NameServer集群中的所有节点建立长连接，定时注册Topic信息到所有NameServer。
   - Producer与NameServer集群中的其中一个节点（随机选择）建立长连接，定期从NameServer取Topic路由信息，并向提供Topic服务的Master建立长连接，且定时向Master发送心跳。Producer完全无状态，可集群部署。
   - Consumer与NameServer集群中的其中一个节点（随机选择）建立长连接，定期从NameServer取Topic路由信息，并向提供Topic服务的Master、Slave建立长连接，且定时向Master、Slave发送心跳。Consumer既可以从Master订阅消息，也可以从Slave订阅消息，订阅规则由Broker配置决定。

2. 集群模式

   （1）单Master模式

   这种方式风险较大，一旦Broker重启或者宕机时，会导致整个服务不可用。不建议线上环境使用,可以用于本地测试。

   （2）多Master模式

   一个集群无Slave，全是Master，例如2个Master或者3个Master，这种模式的优缺点如下：

   - 优点：配置简单，单个Master宕机或重启维护对应用无影响，在磁盘配置为RAID10时，即使机器宕机不可恢复情况下，由于RAID10磁盘非常可靠，消息也不会丢（异步刷盘丢失少量消息，同步刷盘一条不丢），性能最高；
   - 缺点：单台机器宕机期间，这台机器上未被消费的消息在机器恢复之前不可订阅，消息实时性会受到影响。

   （3）多Master多Slave模式（异步）

   每个Master配置一个Slave，有多对Master-Slave，HA采用异步复制方式，主备有短暂消息延迟（毫秒级）。**异步指的是当producer发送消息到Broker，Broker落库后会先响应producer，再同步到slave**。这种模式的优缺点如下：

   - 优点：即使磁盘损坏，消息丢失的非常少，且消息实时性不会受影响，同时Master宕机后，消费者仍然可以从Slave消费，而且此过程对应用透明，不需要人工干预，性能同多Master模式几乎一样；
   - 缺点：Master宕机，磁盘损坏情况下会丢失少量消息。

   （4）多Master多Slave模式（同步）

   每个Master配置一个Slave，有多对Master-Slave，HA采用同步双写方式，即**只有Broker的主备都写成功，才向producer返回响应**，这种模式的优缺点如下：

   - 优点：数据与服务都无单点故障，Master宕机情况下，消息无延迟，服务可用性与数据可用性都非常高；
   - 缺点：性能比异步复制模式略低（大约低10%左右），发送单个消息的RT会略高，且目前版本在主节点宕机后，备机不能自动切换为主机。

#### 双主双从介绍

1. 总体架构

   消息高可用采用2m-2s（同步双写）方式

   <img src="D:\Desktop\Java学习个人笔记整理\13 RocketMQ.assets\RocketMQ集群.png" style="zoom:70%;" />

2. 集群工作流程

   - 启动NameServer，NameServer起来后监听端口，等待Broker、Producer、Consumer连上来，相当于一个路由控制中心。

   - Broker启动，跟所有的NameServer保持长连接，定时发送心跳包。心跳包中包含当前Broker信息(IP+端口等)以及存储所有Topic信息。注册成功后，NameServer集群中就有Topic跟Broker的映射关系。

   - 收发消息前，先创建Topic，创建Topic时需要指定该Topic要存储在哪些Broker上，也可以在发送消息时自动创建Topic。

   - Producer发送消息，启动时先跟NameServer集群中的其中一台建立长连接，并从NameServer中获取当前发送的Topic存在哪些Broker上，轮询从队列列表中选择一个队列，然后与队列所在的Broker建立长连接从而向Broker发消息。

   - Consumer跟Producer类似，跟其中一台NameServer建立长连接，获取当前订阅Topic存在哪些Broker上，然后直接跟Broker建立连接通道，开始消费消息。

### 各种消息发送

#### 基本样例

1. 消息发送

   （1）发送同步消息

   同步发送指生产者发送线程需要阻塞等待Broker返回确认。这种可靠性同步地发送方式使用的比较广泛，比如：重要的消息通知，短信通知。

   ```java
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
   ```

   （2）发送异步消息

   异步发送指生产者将消息交给Broker后立即返回，不会阻塞，当Broker返回结果，通过回调函数处理响应。异步消息通常用在对响应时间敏感的业务场景，即发送端不能容忍长时间地等待Broker的响应。

   ```java
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
   ```

   （3）单向发送消息

   这种方式主要用在不特别关心发送结果的场景，例如日志发送。

   ```java
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
   ```

2. 消息消费

   （1）负载均衡模式（默认）

   消费者采用负载均衡方式消费消息，多个消费者共同消费队列消息，每个消费者处理的消息不同。比如队列中有10条消息，三个消费者，那么三个消费者共同消费这10条消息。

   ```java
   public static void main(String[] args) throws Exception {
       // 实例化消息生产者,指定组名
       DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("group1");
       // 指定Namesrv地址信息.
       consumer.setNamesrvAddr("localhost:9876");
       // 订阅Topic
       consumer.subscribe("Test", "*");
       //负载均衡模式消费
       consumer.setMessageModel(MessageModel.CLUSTERING);
       // 注册回调函数，处理消息
       consumer.registerMessageListener(new MessageListenerConcurrently() {
           @Override
           public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs,
                                                           ConsumeConcurrentlyContext context) {
               System.out.printf("%s Receive New Messages: %s %n", 
                                 Thread.currentThread().getName(), msgs);
               return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
           }
       });
       //启动消息者
       consumer.start();
       System.out.printf("Consumer Started.%n");
   }
   ```

   （2）广播模式

   消费者采用广播的方式消费消息，每个消费者消费的消息都是相同的。比如队列中有10条消息，三个消费者，那么每个消费者都会消费这10条消息。

   ```java
   public static void main(String[] args) throws Exception {
       // 实例化消息生产者,指定组名
       DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("group1");
       // 指定Namesrv地址信息.
       consumer.setNamesrvAddr("localhost:9876");
       // 订阅Topic
       consumer.subscribe("Test", "*");
       //广播模式消费
       consumer.setMessageModel(MessageModel.BROADCASTING);
       // 注册回调函数，处理消息
       consumer.registerMessageListener(new MessageListenerConcurrently() {
           @Override
           public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs,
                                                           ConsumeConcurrentlyContext context) {
               System.out.printf("%s Receive New Messages: %s %n", 
                                 Thread.currentThread().getName(), msgs);
               return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
           }
       });
       //启动消息者
       consumer.start();
       System.out.printf("Consumer Started.%n");
   }
   ```


#### 顺序消息

1. 什么是顺序消息

   顺序消息支持消费者按照发送消息的先后顺序获取消息，从而实现业务场景中的顺序处理。

   消息的顺序性分为两种：

   - 全局有序：所有消息严格按照发送顺序被消费，只能让整个topic仅有一个message queue，性能极低；
   - 局部有序：相同业务标识的消息保持顺序，不同业务标识的消息不保证顺序。比如以orderId确定消息投放至哪个message queue，那么相同orderId的消息能保证有序性。

   <img src="D:\Desktop\Java学习个人笔记整理\13 RocketMQ.assets\image-20251005153018598.png" alt="image-20251005153018598" style="zoom:33%;" />

2. 顺序消息实现原理

   默认的情况下，对于生产者投放至某一topic的消息，会采取轮询方式将消息发放至不同的message queue，消费者会使用多线程去多个queue上拉取消息，因此不能保证顺序。

   rocketmq实现顺序消息的方法是：

   - 队列选择器：生产者通过某种策略（如哈希），将同一业务ID消息发送到确定的message queue；
   - 顺序消费：由于queue的特性，同一queue内确保先进先出；
   - 单线程消费：对某一确定的message queue，消费者只会使用一个线程去消费该message queue。

3. 顺序消息生产

   以订单为例，订单的流程为：创建、付款、推送、完成。可以看成每一个环节完成都会发送消息，那我们肯定希望对于同一笔订单，这几个环节发送的消息都是有序的。

   ```java
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
   		   // send第二个参数表示队列选择器
              SendResult sendResult = producer.send(msg, new MessageQueueSelector() {
                  @Override
                  public MessageQueue select(List<MessageQueue> mqs, Message msg, Object arg) {
                      Long id = (Long) arg;  //根据订单id选择发送queue
                      long index = id % mqs.size();
                      return mqs.get((int) index);
                  }
              }, orderList.get(i).getOrderId());//订单id，也是上面方法的第三个入参arg
   
              System.out.println(String.format("SendResult status:%s, queueId:%d, body:%s",
                  sendResult.getSendStatus(),
                  sendResult.getMessageQueue().getQueueId(),
                  body));
          }
   
          producer.shutdown();
      }
   
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
   ```

4. 顺序消息消费

   ```java
   /**
   * 顺序消息消费，带事务方式（应用可控制Offset什么时候提交）
   */
   public class ConsumerInOrder {
   
      public static void main(String[] args) throws Exception {
          DefaultMQPushConsumer consumer = new 
              DefaultMQPushConsumer("please_rename_unique_group_name_3");
          consumer.setNamesrvAddr("127.0.0.1:9876");
          /**
           * 设置Consumer第一次启动是从队列头部开始消费还是队列尾部开始消费<br>
           * 如果非第一次启动，那么按照上次消费的位置继续消费
           */
          consumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_FIRST_OFFSET);
   
          consumer.subscribe("TopicTest", "TagA || TagC || TagD");
   
          consumer.registerMessageListener(new MessageListenerOrderly() { // 选择orderly的listener
   
              Random random = new Random();
   
              @Override
              public ConsumeOrderlyStatus consumeMessage(List<MessageExt> msgs, ConsumeOrderlyContext context) {
                  context.setAutoCommit(true);
                  for (MessageExt msg : msgs) {
                      // 可以看到每个queue有唯一的consume线程来消费, 订单对每个queue(分区)有序
                      System.out.println("consumeThread=" + Thread.currentThread().getName() + "queueId=" + msg.getQueueId() + ", content:" + new String(msg.getBody()));
                  }
   
                  try {
                      //模拟业务逻辑处理中...
                      TimeUnit.SECONDS.sleep(random.nextInt(10));
                  } catch (Exception e) {
                      e.printStackTrace();
                  }
                  return ConsumeOrderlyStatus.SUCCESS;
              }
          });
   
          consumer.start();
   
          System.out.println("Consumer Started.");
      }
   }
   ```

#### 延时消息

1. 应用

   比如电商里，提交了一个订单就可以发送一个延时消息，1h后去检查这个订单的状态，如果还是未付款就取消订单释放库存。

2. 延时消息生产

   延时消息在生产者侧定义：

   ```java
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
   ```

3. 延时使用限制

   ```java
   // org/apache/rocketmq/store/config/MessageStoreConfig.java
   private String messageDelayLevel = "1s 5s 10s 30s 1m 2m 3m 4m 5m 6m 7m 8m 9m 10m 20m 30m 1h 2h";
   ```

   现在RocketMq并不支持任意时间的延时，需要设置几个固定的延时等级，从1s到2h分别对应着等级1到18。

#### 批量消息

1. 限制

   批量发送消息能显著提高传递小消息的性能。限制是这些批量消息应该有相同的topic，相同的waitStoreMsgOK，而且不能是延时消息。此外，这一批消息的总大小不应超过4MB。

2. 批量消息发送

   发送过程实际上就是在send时传入一个List。

   ```java
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
   ```

   如果消息的总长度可能大于4MB时，这时候最好把消息进行分割。

#### 过滤消息

1. tag过滤

   生产者再发送消息时可以指定topic下的tag，消费者选择订阅哪些tag的消息进行消费。

   在大多数情况下，tag是一个简单而有用的设计，其可以来选择想要的消息。例如：

   ```java
   DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("CID_EXAMPLE");
   consumer.subscribe("TOPIC", "TAGA || TAGB || TAGC");
   ```

   上面的代码表示，消费者将接收包含TAGA或TAGB或TAGC的消息。

2. SQL过滤

   tag过滤的限制是生产者在发送消息时只能指定一个tag，这对于复杂的场景可能不起作用，在这种情况下，可以使用SQL表达式筛选消息。SQL特性可以通过发送消息时的属性来进行计算。

   发送消息时，能通过`putUserProperty`来设置消息的属性：

   ```java
   DefaultMQProducer producer = new DefaultMQProducer("please_rename_unique_group_name");
   producer.start();
   Message msg = new Message("TopicTest",
      tag,
      ("Hello RocketMQ " + i).getBytes(RemotingHelper.DEFAULT_CHARSET)
   );
   // 设置一些属性
   msg.putUserProperty("a", String.valueOf(i));
   SendResult sendResult = producer.send(msg);
   
   producer.shutdown();
   ```

   用MessageSelector.bySql来使用sql筛选消息：

   ```java
   DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("please_rename_unique_group_name_4");
   // 只有订阅的消息有这个属性a, a >=0 and a <= 3
   consumer.subscribe("TopicTest", MessageSelector.bySql("a between 0 and 3");
   consumer.registerMessageListener(new MessageListenerConcurrently() {
      @Override
      public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs, ConsumeConcurrentlyContext context) {
          return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
      }
   });
   consumer.start();
   ```


#### 事务消息

1. 流程分析

   <img src="D:\Desktop\Java学习个人笔记整理\13 RocketMQ.assets\image-20251006110114365.png" alt="image-20251006110114365" style="zoom:70%;" />

2. 事务消息状态

   事务消息共有三种状态，提交状态、回滚状态、中间状态：

   * TransactionStatus.CommitTransaction：提交事务，它允许消费者消费此消息。
   * TransactionStatus.RollbackTransaction：回滚事务，它代表该消息将被删除，不允许被消费。
   * TransactionStatus.Unknown：中间状态，它代表需要消息队列回查生产者来确定状态。

3. 创建事务性生产者

   使用 `TransactionMQProducer`类创建生产者，并指定唯一的 `ProducerGroup`，就可以设置自定义线程池来处理这些检查请求。执行本地事务后、需要根据执行结果对消息队列进行回复。

   ```java
   public class Producer {
       public static void main(String[] args) throws MQClientException, InterruptedException {
           //创建事务监听器
           TransactionListener transactionListener = new TransactionListenerImpl();
           //创建消息生产者，需要使用TransactionMQProducer
           TransactionMQProducer producer = new TransactionMQProducer("group6");
           producer.setNamesrvAddr("192.168.25.135:9876;192.168.25.138:9876");
           //生产者这是监听器
           producer.setTransactionListener(transactionListener);
           //启动消息生产者
           producer.start();
           String[] tags = new String[]{"TagA", "TagB", "TagC"};
           for (int i = 0; i < 3; i++) {
               try {
                   Message msg = new Message("TransactionTopic", tags[i % tags.length], "KEY" + i,
                           ("Hello RocketMQ " + i).getBytes(RemotingHelper.DEFAULT_CHARSET));
                   SendResult sendResult = producer.sendMessageInTransaction(msg, null);
                   System.out.printf("%s%n", sendResult);
                   TimeUnit.SECONDS.sleep(1);
               } catch (MQClientException | UnsupportedEncodingException e) {
                   e.printStackTrace();
               }
           }
           //producer.shutdown();
       }
   }
   ```

   当发送半消息成功时，使用 `executeLocalTransaction` 方法来执行本地事务。它返回前一节中提到的三个事务状态之一。`checkLocalTranscation` 方法用于检查本地事务状态，并回应消息队列的检查请求。它也是返回前一节中提到的三个事务状态之一。

   ```java
   public class TransactionListenerImpl implements TransactionListener {
   	// 执行本地业务逻辑，当生产者调用send时，MQ会调用该方法
       @Override
       public LocalTransactionState executeLocalTransaction(Message msg, Object arg) {
           System.out.println("执行本地事务");
           if (StringUtils.equals("TagA", msg.getTags())) {
               return LocalTransactionState.COMMIT_MESSAGE; // 通知Broker，消息为可消费
           } else if (StringUtils.equals("TagB", msg.getTags())) {
               return LocalTransactionState.ROLLBACK_MESSAGE;
           } else {
               return LocalTransactionState.UNKNOW;
           }
   
       }
   	// 当executeLocalTransaction返回UNKNOW时，MQ回查该方法
       @Override
       public LocalTransactionState checkLocalTransaction(MessageExt msg) {
           System.out.println("MQ检查消息Tag【"+msg.getTags()+"】的本地事务执行结果");
           return LocalTransactionState.COMMIT_MESSAGE;
       }
   }
   ```

4. 使用限制

   - 事务消息不支持延时消息和批量消息。

   - 为了避免单个消息被检查太多次而导致半队列消息累积，默认将单个消息的检查次数限制为 15 次，但是用户可以通过 Broker 配置文件的 `transactionCheckMax`参数来修改此限制。如果已经检查某条消息超过 N 次的话（ N = `transactionCheckMax` ） 则 Broker 将丢弃此消息，并在默认情况下同时打印错误日志。用户可以通过重写 `AbstractTransactionCheckListener` 类来修改这个行为。

   - 事务消息将在 Broker 配置文件中的参数 transactionMsgTimeout 这样的特定时间长度之后被检查。当发送事务消息时，用户还可以通过设置用户属性 CHECK_IMMUNITY_TIME_IN_SECONDS 来改变这个限制，该参数优先于 `transactionMsgTimeout` 参数。

   - 事务性消息可能不止一次被检查或消费。

   - 提交给用户的目标主题消息可能会失败，目前这依日志的记录而定。它的高可用性通过 RocketMQ 本身的高可用性机制来保证，如果希望确保事务消息不丢失、并且事务完整性得到保证，建议使用同步的双重写入机制。

   - 事务消息的生产者 ID 不能与其他类型消息的生产者 ID 共享。与其他类型的消息不同，事务消息允许反向查询、MQ服务器能通过它们的生产者 ID 查询到消费者。

## 高级功能和源码分析

### 消息存储

#### 存储介质

1. 持久化存储

   分布式队列因为有高可靠性的要求，所以数据要进行持久化存储。

   <img src="D:\Desktop\Java学习个人笔记整理\13 RocketMQ.assets\消息存储方式.png" style="zoom:70%;" />

   - 消息生成者发送消息

   - MQ收到消息，将消息进行持久化，在存储中新增一条记录

   - 返回ACK给生产者

   - MQ push 消息给对应的消费者，然后等待消费者返回ACK

   - 如果消息消费者在指定时间内成功返回ack，那么MQ认为消息消费成功，在存储中删除消息，即执行第6步；如果MQ在指定时间内没有收到ACK，则认为消息消费失败，会尝试重新push消息,重复执行4、5、6步骤

   - MQ删除消息

2. 存储介质选择

   - 关系型数据库：非常依赖DB，如果一旦DB出现故障，则MQ的消息就无法落盘存储会导致线上故障；
   - 文件系统：目前业界较为常用的几款产品（RocketMQ/Kafka/RabbitMQ）均采用的是消息刷盘至所部署虚拟机/物理机的文件系统来做持久化（刷盘一般可以分为异步刷盘和同步刷盘两种模式）。消息刷盘为消息存储提供了一种高效率、高可靠性和高性能的数据持久化方式。除非部署MQ机器本身或是本地磁盘挂了，否则一般是不会出现无法持久化的故障问题。

   在性能方面，文件系统>关系型数据库。

#### 存储发送的性能保证

为了提高存储和发送的性能，分别进行优化：

1. 消息存储

   磁盘如果使用得当，磁盘的速度完全可以匹配上网络 的数据传输速度。目前的高性能磁盘，顺序写速度可以达到600MB/s， 超过了一般网卡的传输速度。但是磁盘随机写的速度只有大概100KB/s，和顺序写的性能相差6000倍！因为有如此巨大的速度差别，好的消息队列系统会比普通的消息队列系统速度快多个数量级。**RocketMQ的消息用顺序写**，保证了消息存储的速度。

2. 消息发送

   Linux操作系统分为【用户态】和【内核态】，文件操作、网络操作需要涉及这两种形态的切换，免不了进行数据复制。

   一台服务器 把本机磁盘文件的内容发送到客户端，一般分为两个步骤：

   - read：读取本地文件内容； 

   - write：将读取的内容通过网络发送出去。

   这两个看似简单的操作，实际进行了4 次数据复制，分别是：

   - 从磁盘复制数据到内核态内存；

   - 从内核态内存复制到用户态内存；

   - 然后从用户态 内存复制到网络驱动的内核态内存；

   - 最后是从网络驱动的内核态内存复制到网卡中进行传输。

   ![](D:\Desktop\Java学习个人笔记整理\13 RocketMQ.assets\文件操作和网络操作.png)通过使用**零拷贝**的方式，可以省去**内核态——>用户态**这一步的内存复制，提高速度。这种机制在Java中是通过MappedByteBuffer实现的。

   RocketMQ充分利用了上述特性，提高消息存盘和网络发送的速度。

   需要注意的是：采用MappedByteBuffer这种内存映射的方式有几个限制，其中之一是一次只能映射1.5~2G 的文件至用户态的虚拟内存，这也是RocketMQ默认设置单个CommitLog日志数据文件为1G的原因。

#### 消息存储结构

RocketMQ消息的存储是由ConsumeQueue和CommitLog配合完成的，消息真正的物理存储文件是CommitLog，ConsumeQueue是消息的逻辑队列，类似数据库的索引文件，存储的是指向物理存储的地址。每个Topic下的每个Message Queue都有一个对应的ConsumeQueue文件。

<img src="D:\Desktop\Java学习个人笔记整理\13 RocketMQ.assets\消息存储结构.png" style="zoom:70%;" />

* CommitLog：存储消息的元数据；
* ConsumerQueue：存储消息在CommitLog的索引，即使丢失也能通过CommitLog进行重建；
* IndexFile：为了消息查询提供了一种通过key或时间区间来查询消息的方法，这种通过IndexFile来查找消息的方法不影响发送与消费消息的主流程。

#### 刷盘机制

1. 介绍

   RocketMQ的消息是存储到磁盘上的，这样既能保证断电后恢复， 又可以让存储的消息量超出内存的限制。RocketMQ为了提高性能，会尽可能地保证磁盘的顺序写。消息在通过Producer写入RocketMQ的时候，有两种写磁盘方式，分布式同步刷盘和异步刷盘。

   <img src="D:\Desktop\Java学习个人笔记整理\13 RocketMQ.assets\image-20251012203656698.png" alt="image-20251012203656698" style="zoom:50%;" />

2. 同步刷盘

   在返回写成功状态时，消息已经被写入磁盘。具体流程是，消息写入内存的PAGECACHE后，立刻通知刷盘线程刷盘， 然后等待刷盘完成，刷盘线程执行完成后唤醒等待的线程，返回消息写成功的状态。

3. 异步刷盘

   在返回写成功状态时，消息可能只是被写入了内存的PAGECACHE，写操作的返回快，吞吐量大；当内存里的消息量积累到一定程度时，统一触发写磁盘动作，快速写入。

4. 配置刷盘方式

   同步刷盘还是异步刷盘，都是通过Broker配置文件里的flushDiskType参数设置的，这个参数被配置成SYNC_FLUSH、ASYNC_FLUSH中的一个。

### 高可用性

#### 发送与消费高可用

1. 消息消费高可用

   Consumer可以连接Master角色的Broker，也可以连接Slave角色的Broker来读取消息。当Master不可用或者繁忙的时候，Consumer会被**自动切换**到从Slave读。有了自动切换Consumer这种机制，当一个Master角色的机器出现故障后，Consumer仍然可以从Slave读取消息，不影响Consumer程序。这就达到了消费端的高可用性。

2. 消息发送高可用

   在创建Topic的时候，把Topic的多个Message Queue创建在多个Broker组上（相同Broker名称，不同 brokerId的机器组成一个Broker组），这样当一个Broker组的Master不可用后，其他组的Master仍然可用，Producer仍然可以发送消息。

   RocketMQ目前还不支持把Slave自动转成Master，如果机器资源不足， 需要把Slave转成Master，则要手动停止Slave角色的Broker，更改配置文件，用新的配置文件启动Broker。

#### 消息主从复制

1. 同步复制

   同步复制方式是等Master和Slave均写成功后才反馈给客户端写成功状态；

   在同步复制方式下，如果Master出故障， Slave上有全部的备份数据，容易恢复，但是同步复制会增大数据写入延迟，降低系统吞吐量。

2. 异步复制

   异步复制方式是只要Master写成功，即可反馈给客户端写成功状态。

   在异步复制方式下，系统拥有较低的延迟和较高的吞吐量，但是如果Master出了故障，有些数据因为没有被写 入Slave，有可能会丢失。

3. 配置

   同步复制和异步复制是通过Broker配置文件里的brokerRole参数进行设置的，这个参数可以被设置成ASYNC_MASTER、 SYNC_MASTER、SLAVE三个值中的一个。

4. 同步异步总结

   通常情况下，应该把Master和Save配置成ASYNC_FLUSH的**刷盘方式**，主从之间配置成SYNC_MASTER的**复制方式**，这样即使有一台机器出故障，仍然能保证数据不丢，是个不错的选择。

### 负载均衡

1. 生产者负载均衡

   Producer端，每个实例在发消息的时候，默认会轮询所有的message queue发送，以达到让消息平均落在不同的queue上。而由于queue可以散落在不同的broker，所以消息就发送到不同的broker下，如下图：

   <img src="D:\Desktop\Java学习个人笔记整理\13 RocketMQ.assets\image-20251012204921565.png" alt="image-20251012204921565" style="zoom:70%;" />

   图中箭头线条上的标号代表顺序，发布方会把第一条消息发送至Queue 0，然后第二条消息发送至Queue 1，以此类推，第7条消息又会发送至Queue 0。

2. 消费者负载均衡

   （1）集群模式

   在集群消费模式下，每条消息只需要投递到订阅这个topic的Consumer Group下的一个实例即可。RocketMQ采用主动拉取的方式拉取并消费消息，在拉取的时候需要明确指定拉取哪一条message queue。

   而每当实例的数量有变更，都会触发一次所有实例的负载均衡，这时候会按照queue的数量和实例的数量平均分配queue给每个实例。

   默认的分配算法是AllocateMessageQueueAveragely，如下图：

   <img src="D:\Desktop\Java学习个人笔记整理\13 RocketMQ.assets\image-20251012205105639.png" alt="image-20251012205105639" style="zoom:67%;" />

   还有另外一种平均的算法是AllocateMessageQueueAveragelyByCircle，也是平均分摊每一条queue，只是以环状轮流分queue的形式，如下图：

   <img src="D:\Desktop\Java学习个人笔记整理\13 RocketMQ.assets\image-20251012205126627.png" alt="image-20251012205126627" style="zoom:67%;" />

   需要注意的是，集群模式下，都是一个queue只分给一个consumer实例，一个consumer实例可以允许同时分到不同的queue。

   通过增加consumer实例去分摊queue的消费，可以起到水平扩展的消费能力的作用。而有实例下线的时候，会**重新触发**负载均衡，这时候原来分配到的queue将分配到其他实例上继续消费。

   如果consumer实例的数量比message queue的总数量还多的话，多出来的consumer实例将无法分到queue，也就无法消费到消息，也就无法起到分摊负载的作用了。所以需要控制让queue的总数量大于等于consumer的数量。

   （2）广播模式

   由于广播模式下要求一条消息需要投递到一个消费组下面所有的消费者实例，所以也就没有消息被分摊消费的说法。

   在实现上，其中一个不同就是在consumer分配queue的时候，所有consumer都分到所有的queue。

   <img src="D:\Desktop\Java学习个人笔记整理\13 RocketMQ.assets\image-20251012205358836.png" alt="image-20251012205358836" style="zoom:67%;" />

### 消息重试

1. 顺序消息重试

   对于顺序消息，当消费者消费消息失败后，消息队列 RocketMQ 会自动不断进行消息重试（每次间隔时间为 1 秒），这时，应用会出现消息消费被阻塞的情况。

   因此，在使用顺序消息时，务必保证应用能够及时监控并处理消费失败的情况，避免阻塞现象的发生。

2. 无序消息重试

   对于无序消息（普通、定时、延时、事务消息），当消费者消费消息失败时，可以通过设置返回状态达到消息重试的结果。

   无序消息的重试只针对集群消费方式生效；广播方式不提供失败重试特性，即消费失败后，失败消息不再重试，继续消费新的消息。

   （1）重试次数

   消息队列 RocketMQ 默认允许每条消息最多重试 16 次，每次重试的间隔时间如下：

   | 第几次重试 | 与上次重试的间隔时间 | 第几次重试 | 与上次重试的间隔时间 |
   | :--------: | :------------------: | :--------: | :------------------: |
   |     1      |        10 秒         |     9      |        7 分钟        |
   |     2      |        30 秒         |     10     |        8 分钟        |
   |     3      |        1 分钟        |     11     |        9 分钟        |
   |     4      |        2 分钟        |     12     |       10 分钟        |
   |     5      |        3 分钟        |     13     |       20 分钟        |
   |     6      |        4 分钟        |     14     |       30 分钟        |
   |     7      |        5 分钟        |     15     |        1 小时        |
   |     8      |        6 分钟        |     16     |        2 小时        |

   如果消息重试 16 次后仍然失败，消息将不再投递。如果严格按照上述重试时间间隔计算，某条消息在一直消费失败的前提下，将会在接下来的 4 小时 46 分钟之内进行 16 次重试，超过这个时间范围消息将不再重试投递。

   **注意：** 一条消息无论重试多少次，这些重试消息的Message ID不会改变。

   （2）配置方式

   集群消费方式下，消息消费失败后期望消息重试，需要在消息监听器接口的实现中明确进行配置（三种方式任选一种）：

   - 返回 Action.ReconsumeLater （推荐）
   - 返回 Null
   - 抛出异常

   ```java
   public class MessageListenerImpl implements MessageListener {
       @Override
       public Action consume(Message message, ConsumeContext context) {
           //处理消息
           doConsumeMessage(message);
           //方式1：返回 Action.ReconsumeLater，消息将重试
           return Action.ReconsumeLater;
           //方式2：返回 null，消息将重试
           return null;
           //方式3：直接抛出异常， 消息将重试
           throw new RuntimeException("Consumer Message exceotion");
       }
   }
   ```

   若希望消费失败后，不进行重试，需要捕获消费逻辑中可能抛出的异常，最终返回 Action.CommitMessage，此后这条消息将不会再重试：

   ```java
   public class MessageListenerImpl implements MessageListener {
       @Override
       public Action consume(Message message, ConsumeContext context) {
           try {
               doConsumeMessage(message);
           } catch (Throwable e) {
               //捕获消费逻辑中的所有异常，并返回 Action.CommitMessage;
               return Action.CommitMessage;
           }
           //消息处理正常，直接返回 Action.CommitMessage;
           return Action.CommitMessage;
       }
   }
   ```

   消息队列 RocketMQ 允许 Consumer 启动的时候设置最大重试次数，重试时间间隔将按照如下策略：

   - 最大重试次数小于等于 16 次，则重试时间间隔同上表描述。
   - 最大重试次数大于 16 次，超过 16 次的重试时间间隔均为每次 2 小时。

   ```java
   Properties properties = new Properties();
   //配置对应 Group ID 的最大消息重试次数为 20 次
   properties.put(PropertyKeyConst.MaxReconsumeTimes,"20");
   Consumer consumer =ONSFactory.createConsumer(properties);
   ```

   > 注意：

   - 消息最大重试次数的设置对相同 Group ID 下的所有 Consumer 实例有效。
   - 如果只对相同 Group ID 下两个 Consumer 实例中的其中一个设置了 MaxReconsumeTimes，那么该配置对两个 Consumer 实例均生效。
   - 配置采用覆盖的方式生效，即最后启动的 Consumer 实例会覆盖之前的启动实例的配置。

   （3）获取消息重试次数

   消费者收到消息后，可按照如下方式获取消息的重试次数：

   ```java
   public class MessageListenerImpl implements MessageListener {
       @Override
       public Action consume(Message message, ConsumeContext context) {
           //获取消息的重试次数
           System.out.println(message.getReconsumeTimes());
           return Action.CommitMessage;
       }
   }
   ```


### 死信队列

1. 死信消息

   在消息队列RocketMQ中，正常情况下无法被消费的消息称为死信消息（Dead-Letter Message），存储死信消息的特殊队列称为死信队列（Dead-Letter Queue）。

2. 死信特性

   死信消息具有以下特性

   - 不会再被消费者正常消费。
   - 有效期与正常消息相同，均为 3 天，3 天后会被自动删除。因此，请在死信消息产生后的 3 天内及时处理。

   死信队列具有以下特性：

   - 一个死信队列对应一个 Group ID， 而不是对应单个消费者实例。
   - 如果一个 Group ID 未产生死信消息，消息队列 RocketMQ 不会为其创建相应的死信队列。
   - 一个死信队列包含了对应 Group ID 产生的所有死信消息，不论该消息属于哪个 Topic。

3. 死信查看与处理

   在控制台可以查看死信消息。

   对于死信消息的处理，有两种方法：

   - 在控制台重新发送死信消息，让消费者重新消费一次；
   - 写一个新的消费者订阅死信队列。

### 消费幂等

#### 消息重复的情况

之所以要进行幂等性判断，就是因为消息可能重复。

在互联网应用中，尤其在网络不稳定的情况下，消息队列 RocketMQ 的消息有可能会出现重复，这个重复简单可以概括为以下情况：

- 发送时消息重复

  当一条消息已被成功发送到服务端并完成持久化，此时出现了网络闪断或者客户端宕机，导致服务端对客户端应答失败。 如果此时生产者意识到消息发送失败并尝试再次发送消息，消费者后续会收到两条内容相同并且 Message ID 也相同的消息。

- 投递时消息重复

  消息消费的场景下，消息已投递到消费者并完成业务处理，当客户端给服务端反馈应答的时候网络闪断。 为了保证消息至少被消费一次，消息队列 RocketMQ 的服务端将在网络恢复后再次尝试投递之前已被处理过的消息，消费者后续会收到两条内容相同并且 Message ID 也相同的消息。

- 负载均衡时消息重复（包括但不限于网络抖动、Broker 重启以及订阅方应用重启）

  当消息队列 RocketMQ 的 Broker 或客户端重启、扩容或缩容时，会触发 Rebalance，此时消费者可能会收到重复消息。

#### 处理方式

因为Message ID有可能出现冲突（重复）的情况，所以真正安全的幂等处理，不建议以Message ID作为处理依据。 最好的方式是以业务唯一标识作为幂等处理的关键依据。

