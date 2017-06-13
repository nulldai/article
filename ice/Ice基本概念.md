#什么是Ice
Ice is a comprehensive RPC framework with support for C++, C#, Java, JavaScript, Python, and more.

Ice是一个支持多语言的RPC框架，它又不只是一个RPC框架，完整的技术栈还包括

- Ice Core网络通信的运行时支持
- Ice Util实用工具库(包含只能指针、线程、字符编码处理等)
- Slice语言和编译器
- IceBox服务对象Servant容器
- IceGrid位置服务和服务管理
- IcePatch2目录同步服务，适用于应用部署升级
- IceStorm高效的订阅发布服务
- Glacier2轻量的防火墙穿透解决方案
- Freeze提供Ice对象的持久化存储服务

#Object/Servant/Proxy
````

                     |
                     |
   +---------+       |                +------------+
   |  Proxy  +----------------------> |   Object   |
   +---------+       |                +------------+
        ^            |                      |
        |            |                      |
        |            |                      |
   +---------+       |      +---------+ +---v-----+ +---------+
   |  Client |       |      | Servant | | Servant | | Servant |
   +---------+       |      +---------+ +---------+ +---------+
                     |
              Client | Server


````

##Ice Object
- 是一个抽象接口，不是具体实现
- 存在于客户端和服务端，用于响应和转发请求
- 支持多facet(interfaces)，可以在各个facet之间进行转换
- 有唯一的对象标识(Object Identify)

##Ice Servant
- Ice Object的服务端具体实现
- 可以创建并添加到ObjectAdapter让客户端引用(指定实例对象的唯一标识Identity)
- Object/Servant生命周期相互独立

##Ice Proxy
- Ice Object的客户端代理，负责代理Object请求
- 负责Object定位(Object实例可能存在于多个服务端)
- 激活Object实例，接受客户端请求

#Object Adapter
- Ice Object实例对象Servant的集合
- 可以指定多个Endpoint入口
- 负责将入口请求分派到各个Servant实例处理
- 负责管理Servant实例生命周期

#Endpoint
指明服务端所使用的协议、地址、端口，例如"default -h 192.168.1.10 -p 10000 -t 10000"

- 表示协议为default，default值可以通过Ice.Default.Protocol设置，默认为TCP/IP,还可以设置成UDP、SSL  
- -h指定绑定的IP地址  
- -p指定监听的端口  
- -t设置超时时间  
- 多个Endpoint用分号:分隔


#Stringified Proxy
Proxy引用也可以用一个字符串描述，例如"SimplePrinter:default -p 10000"  

- SimplePrinter为对象标识(Servant加入Adapter时指定)  
- default -p 10000为Adapter的Endpoint

#Direct Proxy
直接代理指Proxy描述中包含需要访问的服务Endpoint的地址

#Indirect Proxy
间接代理值Proxy描述中不直接包含需要访问服务Endpoint的地址，间接代理有两种方式

- 只包含对象标识
- 对象标识和Object Adapter标识

以上两种情况都需要配合IceGrid定位服务查找对象Endpoint

#Route Proxy
Route Proxy负责将所有的请求转发到特定的目标对象，用于实现类似Glacier2的服务，允许客户端和防火墙后的服务进行通讯

#Replication
Replication使Object Adapter可以在多个Endpoint地址上进行访问，提供冗余服务提高服务器可用性，可以通过Proxy中指定对个Endpoint地址进行随机选择

#Replica Group
是Replication的升级版，提供更加强大的负载均衡能力，Replica Group实现机制是借助于Location Service来实现服务的定位，同时提供了多种分派策略实现负载均衡，Replica Group本身可以认为是一个Object Adapter，Replica Group标识即为Adapter标识，可用于Indirect Proxy访问

#Synchronous Method Invocation
默认情况下，Ice使用的请求模式是同步阻塞模式，在方法调用的过程中会阻塞当前线程，直到服务端返回结果或者网络异常

#Asynchronous Method Invocation(AMI)
与同步方式不同的是客户端调用方法时可以传入一个回调对象，而当前调用会立即返回，当服务端处理完成之后会通过回调对象返回处理结果，而服务端并不关心客户端是同步等待还是异步

#Asynchronous Method Dispatch(AMD)
AMD是服务端异步分派，服务端默认采用同步分派模式，即分派线程除了处理分派请求外还需要处理应用的业务代码，等待处理完成之后才能返回客户端继续处理下一个请求，这种模式对相对耗时业务代码处理会变得非常低效，而AMD采用的是异步分派，即请求到达之后应用代码可以选择将请求放入处理队列，交给用户线程进行处理，然后释放分派线程继续处理下一个请求，当用户线程处理完请求后可以将处理结果通过回调传给客户端

#Oneway Method Invocation
对Oneway请求，客户端将请求发送至TCP缓冲区即完成了方法调用，它并关心返回值，送达情况和服务端的处理结果等，Oneway调用是不可靠的请求方式，但可以获得更高的性能

#Batched Oneway Method Invocation
批量Oneway调用

#Datagram Invocation
与Oneway请求类似也是不可靠的请求方式，不过Datagram Invocation采用的是UDP传输机制

#Batched Datagram Invocation
批量Datagram调用
