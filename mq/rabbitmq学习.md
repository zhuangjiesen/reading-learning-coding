# rabbitmq学习
### 简介：RabbitMQ是基于AMQP(Advanced Message Queue Protocol)高级消息队列协议的一种消息队列，
### 消费者与生产者
##### 生产者（producer）创建消息，发布到代理服务器（RabbitMQ）,消息包含两部分内容：有效载荷（payload）和标签（label）。有效载荷就是你想要传输的数据（json数组或者自己定义的数据），标签描述了有效载荷并且RabbitMQ用它来决定谁讲获得消息的拷贝。
##### 消费者通过连接代理服务器上，并订阅到队列（queue）上。消费者接收消息并无法知道消息发送源信息，除非发送源信息附在消息内容中。
##### 你的程序可以作为生产者，向其他应用程序发送消息；也可以作为消费者，接收消息；还可以都是。
#### 信道
##### 不论是发布消息，订阅队列还是接收消息，这些动作都是通过队列完成的，应用程序通过tcp连接与rabbitMQ服务器进行连接，一旦连接上服务器，应用服务器会创建一条AMQP信道，每条信道都会被指派一个唯一ID。AMQP命令都是通过信道发送出去的。一条TCP连接上创建多少条信道是没有限制的；使用多个信道，线程就可以共享连接，以至于不消耗大量连接操作；
#### 安装RabbitMQ 
##### MAC下安装RabbitMQ。
1. 下载http://www.rabbitmq.com/releases/rabbitmq-server/v3.5.3/rabbitmq-server-mac-standalone-3.5.3.tar.gz rabbitMQ包
2. 并且解压
`tar －zxvf  rabbitmq-server-mac-standalone-3.5.3.tar.gz`
3. 找到/sbin目录
`./rabbitmq-server start`
* 启动rabbitMQ服务
4. `./rabbitmqctl stop `
* 停止服务
`./rabbitmqctl status`
* 查看服务状态
`./rabbitmq-plugins enable rabbitmq_management`
* 启动rabbitMQ页面管理平台
http:localhost(ip):15672


### maven + java +spring +rabbitmq 实战
#### 消费者
##### 消费者类：

#### maven依赖
```xml

      <!-- rabbitmq  -->

      <!-- for rabbitmq -->
      <dependency>
          <groupId>com.rabbitmq</groupId>
          <artifactId>amqp-client</artifactId>
          <version>2.8.2</version>
      </dependency>
      <dependency>
          <groupId>org.springframework.amqp</groupId>
          <artifactId>spring-amqp</artifactId>
          <version>1.1.1.RELEASE</version>
      </dependency>
      <dependency>
          <groupId>org.springframework.amqp</groupId>
          <artifactId>spring-rabbit</artifactId>
          <version>1.1.1.RELEASE</version>
      </dependency>
      <dependency>
          <groupId>com.caucho</groupId>
          <artifactId>hessian</artifactId>
          <version>4.0.7</version>
      </dependency>
      <!-- rabbitmq  -->


```

-------

#### AmqpTemplate.java 消息发送模板类解读

```Java
/** exchange 交换器名称 ""空字符串表示默认direct交换器    */
/** routingKey 路由键 */
/** Message 消息类  */

/** 发送消息*/
    void send(Message message) throwsAmqpException;
    void send(String routingKey, Message message) throwsAmqpException;
    void send(String exchange, String routingKey, Message message) throwsAmqpException;
    
    
    
```



##### MessageConsumer.java
```Java
package com.java.rabbitmq.consumer;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.amqp.core.Message;
import org.springframework.amqp.core.MessageListener;

import java.io.File;
import java.io.IOException;

/**
 * Created by zhuangjiesen on 2017/1/14.
 */
public class MessageConsumer implements MessageListener{
//    private Logger logger = LoggerFactory.getLogger(MessageConsumer.class);
    

    public void onMessage(Message message) {
//        logger.info("receive message:{}",message);
        System.out.println("===============================");

        System.out.println("receive message: "+message.toString());
    }
}

```

-------

#### spring 配置文件 spring-rabbitmq-consumer.xml 
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:rabbit="http://www.springframework.org/schema/rabbit"
	   xsi:schemaLocation="http://www.springframework.org/schema/beans
     http://www.springframework.org/schema/beans/spring-beans-3.0.xsd
     http://www.springframework.org/schema/beans
     http://www.springframework.org/schema/beans/spring-beans-3.0.xsd
     http://www.springframework.org/schema/rabbit
     http://www.springframework.org/schema/rabbit/spring-rabbit-1.0.xsd">
	<!--配置connection-factory，指定连接rabbit server参数 -->
	<rabbit:connection-factory id="connectionFactory"
							   username="guest" password="guest" host="127.0.0.1" port="5672" />

	<!--定义rabbit template用于数据的接收和发送 -->
	<rabbit:template id="amqpTemplate"  connection-factory="connectionFactory"
					 exchange="exchangeTest" />

	<!--通过指定下面的admin信息，当前producer中的exchange和queue会在rabbitmq服务器上自动生成 -->
	<rabbit:admin connection-factory="connectionFactory" />

	<!--定义queue -->
	<rabbit:queue name="queueTest" durable="true" auto-delete="false" exclusive="false" />

	<!-- 定义direct exchange，绑定queueTest -->
	<rabbit:direct-exchange name="exchangeTest" durable="true" auto-delete="false">
		<rabbit:bindings>
			<rabbit:binding queue="queueTest" key="queueTestKey"></rabbit:binding>
		</rabbit:bindings>
	</rabbit:direct-exchange>

	<!-- 消息接收者 -->
	<bean id="messageReceiver" class="com.java.rabbitmq.consumer.MessageConsumer"></bean>

	<!-- queue litener  观察 监听模式 当有消息到达时会通知监听在对应的队列上的监听对象-->
	<rabbit:listener-container connection-factory="connectionFactory">
		<rabbit:listener queues="queueTest" ref="messageReceiver"/>
	</rabbit:listener-container>

</beans>
```


-------


### 生产者 
#### 生产者类 MessageProducer.java
```Java

package com.java.rabbitmq.producer;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.amqp.core.AmqpTemplate;

import javax.annotation.Resource;

/**
 * Created by zhuangjiesen on 2017/1/14.
 */
public class MessageProducer {

    private Logger logger = LoggerFactory.getLogger(MessageProducer.class);

    @Resource
    private AmqpTemplate amqpTemplate;

    public void sendMessage(Object message){
        logger.info("to send message:{}",message);
        amqpTemplate.convertAndSend("queueTestKey",message);
    }

}


```

-------
#### spring 配置文件 spring-rabbitmq-producer.xml
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:rabbit="http://www.springframework.org/schema/rabbit"
	   xsi:schemaLocation="http://www.springframework.org/schema/beans
     http://www.springframework.org/schema/beans/spring-beans-3.0.xsd
     http://www.springframework.org/schema/beans
     http://www.springframework.org/schema/beans/spring-beans-3.0.xsd
     http://www.springframework.org/schema/rabbit
     http://www.springframework.org/schema/rabbit/spring-rabbit-1.0.xsd">
	<!--配置connection-factory，指定连接rabbit server参数 -->
	<rabbit:connection-factory id="connectionFactory"
							   username="guest" password="guest" host="127.0.0.1" port="5672" />

	<!--定义rabbit template用于数据的接收和发送 -->
	<rabbit:template id="amqpTemplate"  connection-factory="connectionFactory"
					 exchange="exchangeTest" />

	<!--通过指定下面的admin信息，当前producer中的exchange和queue会在rabbitmq服务器上自动生成 -->
	<rabbit:admin connection-factory="connectionFactory" />

	<!--定义queue -->
	<rabbit:queue name="queueTest" durable="true" auto-delete="false" exclusive="false" />

	<!-- 定义direct exchange，绑定queueTest -->
	<rabbit:direct-exchange name="exchangeTest" durable="true" auto-delete="false">
		<rabbit:bindings>
			<rabbit:binding queue="queueTest" key="queueTestKey"></rabbit:binding>
		</rabbit:bindings>
	</rabbit:direct-exchange>

	<bean id="messageProducer"  class="com.java.rabbitmq.producer.MessageProducer" ></bean>

</beans>


```


-------
#### 测试
##### 博主是在spring mvc 下实现的 Java代码如下
```Java
/**
	 Description: 跳转rabbitmq 测试  <p>
	 @author : zhuangjiesen@ssit-xm.com.cn 庄杰森 2016年4月19日
	 */
	@RequestMapping("/common/indexToRabbitmq.do")
	public String indexToRabbitmq(HttpServletRequest request,HttpServletResponse response,ModelMap model){

		MessageProducer messageProducer=(MessageProducer)BeanHelper.getApplicationContext().getBean("messageProducer");
		messageProducer.sendMessage("我是庄杰森的rabbit mq !!!!  hello world !!");
		model.addAttribute("title","我是rabbitmq hello world~@!!!");
		return "/common/index.html";
	}
```


### AMQP架构中最重要的组件： 交换器，队列和绑定
#### 绑定
##### 根据绑定规则将队列绑定到交换器上（hash算法 ，发送消息时生成的路由键进行匹配）
#### 交换器
#### 有三种类型交换器：direct 、 fanout 、 topic
#### direct交换器（普通的交换器）
##### rabbit服务器默认的交换器，声明一个队列时，它会自动绑定到默认交换器，并以队列名称作为路由键；
#### fanout交换器
##### 当你发送一条消息到fanout小唤起时，它会把消息投递给所有附加在此交换器上的队列--->多个队列（queue绑定一个fanout）
##### JAVA + spring中实现
#### 生产者
##### 在spring 配置文件中添加exchange交换器类型，并且绑定queue
```xml
spring-rabbitmq-producer.xml
```
```xml

	<!-- fanout queue-->
	<rabbit:queue name="fanout-queue-1" durable="true" auto-delete="false" exclusive="false" />
	<rabbit:queue name="fanout-queue-2" durable="true" auto-delete="false" exclusive="false" />



<!-- 定义fanout exchange，绑定 fanout-exchange-test-->
	<rabbit:fanout-exchange name="fanout-exchange-test"  auto-delete="false">
		<rabbit:bindings>
			<rabbit:binding queue="fanout-queue-1"   ></rabbit:binding>
			<rabbit:binding queue="fanout-queue-2"  ></rabbit:binding>
		</rabbit:bindings>
	</rabbit:fanout-exchange>
```

```Java
/**发送消息*/
        MessageProperties messageProperties=new MessageProperties();
        /**消息持久化 投递模式*/messageProperties.setDeliveryMode(MessageDeliveryMode.PERSISTENT);
        Message msg=new Message(message.getBytes(), messageProperties);
        /**发送消息*/
        amqpTemplate.send("fanout-exchange-test",msg);



```

#### 消费者
#### 在spring 配置文件中添加exchange交换器类型 ，并且绑定接收消息的queue 
```xml
spring-rabbitmq-consumer.xml
```
```xml
	<!-- fanout queue-->
	<rabbit:queue name="fanout-queue-1" durable="true" auto-delete="false" exclusive="false" />
	<rabbit:queue name="fanout-queue-2" durable="true" auto-delete="false" exclusive="false" />


<!--定义fanout交换器 消息接收类  定义两个测试顺序-->
	<bean id="fanoutMessageReceiver_1" class="com.java.rabbitmq.consumer.FanoutMessageConsumer_1"></bean>
	<bean id="fanoutMessageReceiver_2" class="com.java.rabbitmq.consumer.FanoutMessageConsumer_2"></bean>





	<!-- queue litener  观察 监听模式 当有消息到达时会通知监听在对应的队列上的监听对象-->
	<rabbit:listener-container  connection-factory="connectionFactory">


		<rabbit:listener ref="fanoutMessageReceiver_1" response-exchange="fanout-exchange" queues="fanout-queue-1" />
		<rabbit:listener ref="fanoutMessageReceiver_2" response-exchange="fanout-exchange" queues="fanout-queue-2" />
	</rabbit:listener-container>


```

```Java 
FanoutMessageConsumer_1.java (FanoutMessageConsumer_2 的代码也是类似就不贴了 )
```
```Java 
package com.java.rabbitmq.consumer;

import org.springframework.amqp.core.Message;
import org.springframework.amqp.core.MessageListener;

import java.io.UnsupportedEncodingException;

/**
 * Created by zhuangjiesen on 2017/1/14.
 */
public class FanoutMessageConsumer_1 implements MessageListener{
//    private Logger logger = LoggerFactory.getLogger(MessageConsumer.class);


    public void onMessage(Message message) {
//        logger.info("receive message:{}",message);
        String threadName=Thread.currentThread().getName();

        System.out.println("=============FanoutMessageConsumer_1============= threadName : "+threadName);

        System.out.println(" FanoutMessageConsumer_1 : receive message: "+message.toString());

        try {
            System.out.println("  message: "+new String (message.getBody(),"utf-8"));
        } catch (UnsupportedEncodingException e) {
            e.printStackTrace();
        }
    }
}

```


#### topic交换器
#### Topic型交换器比较强大跟其它交换器很相似。
#### 当一个队列以”#”作为绑定键时，它将接收所有消息，而不管路由键如何，类似于fanout型交换器。
#### 当特殊字符”*”、”#”没有用到绑定时，topic型交换器就好比direct型交换器了。
### 路由键上限长度255字符 

-------

#### topic交换器通过路由键通配符 匹配消息；

### RabbitMQ消息持久化
#### 将队列和交换器的durable属性设置为true,同时在消息发布前，通过把他的“投递模式”（delivery mode ） 选项设置为2 （MessageDeliveryMode.PERSISTENT ）来把消息标记成持久化
#### 消息持久化所要付出的代价是：性能，存储磁盘与内存相比性能相差肯定大，10倍以上的差别的情况并不少见；


