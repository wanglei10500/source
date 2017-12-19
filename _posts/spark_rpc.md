---
title: spark RPC
tags:
 - spark
categories: 经验分享
---

### 概要
Spark Rpc被deploy scheduler shuffle storage等多个模块使用，最初使用akka的actor实现 1.4+标准化了相关接口 1.6+转变为基于netty，解决了可能存在的akka版本问题

### actor
重要概念：ActorSystem、Actor和actorRef
```
object ActorTest{
  class HelloActor extends Actor{
    def receive={
      case msg:String=>println("hello "+msg)
      case _=>println("unexpected messsage.")
    }
  }
  def main(args:Array[String]):Unit={
    val system=ActorSystem("HelloSystem")
    val actorRef=system.actorOf(Props[HelloActor],name="hello")
    actorRef!"smith"
    system.shutdown()
  }
}
```
定义HelloActor用于处理信息，接着在main函数中实例化ActorSystem，调用其actorOf方法注册定义的actor并取名hello ，返回这个actor的引用actorRef。利用这个引用和actor通信

接下来发送消息 "smith"给actor HelloActor接收到信息并打印到屏幕
### Rpc接口
* RpcEndpoint=>Actor
* RpcEndpointRef=>actorRef
* RpcEnv=>ActorSystem

#### RpcEndpoint
用于处理信息 其中有两个重要方法：receive 和 receiveAndReply,区别是后者处理完信息后会返回信息给发送者，类似tcp和udp

RpcEndpoint的生命周期：onStart->receive(receiveAndReply)* ->onStop，剩下的几个方法分别是处理连接建立断开和错误事件，deploy模块的Master Worker是RpcEndpoint的具体实现
#### RpcEnv
注册并维护RpcEndpoint和RpcEndpointRef 主要方法为setupEndpoint,用于注册RpcEndpoint，内部使用Dispatcher维护注册的RpcEndpoint，也提供了多种获取RpcEndpoint的方法，如asyncSetupEndpointRefByURI、setupEndpointRefByURI、setupEndpointRef，以及移除RpcEndpoint的方法stop，关闭RpcEnv的方法shutdown，其还维护了RpcEnvFileServer，用于上传下载jar和file。最后实例化RpcEnv时，需指定是server模式i还是client(默认是server)，server模式下底层启动netty
#### RpcEndpointRef
向对应的RpcEndpoint发送信息 主要方法send和ask，send方法只发送信息，ask方法发送信息的同时接受返回值，ask方法又有几个相似方法，涉及到retry和timeout.此外 还有两个属性address和name 用于对应这个RpcEndpointRef所属的RpcEndpoint.

NettyRpcEndpointRef是其具体实现，内部使用Dispatcher、Inbox、Outbox等组件发送信息
##### Dispatcher、Inbox、Outbox
Dispatcher和Inbox作用于server端，处理请求，Outbox作用于client端，处理和远端netty server通信的情况

Dispatcher主要职责如下：

1. 内部使用集合endpoints和endpointRefs维护Endpoint、EndpointRef，对外通过registerRpcEndpoint、removeRpcEndpointRef、getRpcEndpointRef等方法提供Endpoint注册删除和获取EndpointRef等服务。
2. 利用EndpointData和Inbox结构完成消息的存储。
3. 创建线程池threadpool，执行MessageLoop线程，消费消息。

Inbox绑定了消息InboxMessage和其对应的消费者Endpoint.Inbox的作用：

1. 内部了维护了链表messages，用于存储消息，同时维护该消息对应消费者Endpoint。
2. 提供了post和process两个方法，分别用于添加消息到messages和消费消息，process方法在MessageLoop中被调用。 消息类型为InboxMessage

Outbox作用于client端，当RpcEndpointRef请求RpcEndpoint时，若RpcEndpointRef和RpcEndpoint位于同一机器时，走的是Inbox的逻辑 否则，即RpcEndpointRef和RpcEndpoint不在一台机器，则RpcEndpointRef将信息发送到Outbox。

和Inbox类似，Outbox内部也维护了messages用于存储消息，此外还有TransportClient，用于和相应的netty server通信。方法send用于将消息添加到messages并调用drainOutbox方法消费消息。

Outbox对应的消息类型为OutboxMessage，对应子类如下[RpcOutboxMessage处理有返回值的情况]

和Inbox略有不同，OutBox并没有启动单独线程MessageLoop，仅是在方法中消费，最终调用TransportClient的send或sendRpc方法发送消息。

ndpointRef、OutBox、TransportClient是根据RpcAddress一一对应的，意思是当一个节点需要和多个节点通信时，会为每个节点创建对应的OutBox，由outboxes集合维护，同时每个OutBox对象内创建对应的要访问节点的TransportClient

Dispatcher、Inbox、OutBox的组成，及作用:
* Dispatcher、Inbox作用于server端，分发并处理EndpointRef发送的信息。
* OutBox作用于client端，当访问远程server时，使用TransportClient发送消息，如果访问的server在一个程序中，直接交给Dispatcher、Inbox，不需要OutBox。
![spark](http://img.blog.csdn.net/20170308143702509?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMTU2NDE3Mg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
![spark](http://img.blog.csdn.net/20170308161513502?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMTU2NDE3Mg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

#### RpcAddress和RpcCallContext
RpcAddress用于维护host和port，以及SparkURL格式和其他格式之间的转换

RpcCallContext作用于RpcEndpoint中，当RpcEndpoint调用receiveAndReply处理完信息后，使用RpcCallContext的reply方法将处理结果返回。

![spark](http://img.blog.csdn.net/20170220161915598?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMTU2NDE3Mg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

#### spark-netty
spark对netty做了封装，在spark-network-common模块，如下
1. TransportContext维护Transport的上下文环境，用于创建TransportServer和TransportClientFactory。
2. TransportServer通过构造函数启动netty，提供底层通信服务。
3. TransportClientFactory用来创建TransportClient。
4. TransportClient用以和对应的TransportServer通信。
此外还有用于处理信息的MessageEncoder、MessageDecoder、RpcHandler

1. spark中启动netty的过程，即创建RpcEnv时，通过TransportContext实例化TransportServer，TransportServer构造器中启动netty。
2. spark对netty的封装，主要包括TransportContext、TransportContext、TransportClientFactory和TransportClient。
### Spark RPC通信机制

Spark RPC中最为重要的三个抽象：RpcEnv、RpcEndpoint、RpcEndpointRef，这样做的好处有：
* 对上层API来说，屏蔽了底层的具体实现，使用方便
* 可以通过不同的实现来完成指定的功能，方便扩展
* 促进了底层实现层的良性竞争，Spark 1.6.3中默认使用了Netty作为底层的实现，但Akka的依赖依然存在；而Spark 2.1.0中的底层实现只有Netty，这样用户可以方便的使用不同版本的Akka或者将来某种更好的底层实现
