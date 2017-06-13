# Bidirectional连接

## 使用场景

正常情况下Ice的连接只允许单向请求，也就是从客户端到服务端发送请求，假如应用需要从服务端发送Callback请求给客户端，通常情况下都是从服务端建立一条新的连接到客户端，如下图

![](https://doc.zeroc.com/download/attachments/16716352/Callbacks_open_network.gif?version=1&modificationDate=1470091306000&api=v2)

但是在一些网络环境下无法从服务端建立连接到客户端，例如Firewall的限制

![](https://doc.zeroc.com/download/attachments/16716352/Callbacks_with_firewall.gif?version=1&modificationDate=1470091306000&api=v2)

Bidirectional Connection提供一种机制支持从客户端到服务端建立一条双向连接，请求可以在这条连接上双向发送，服务端可以通过这条连接发送Callback请求给客户端

有两种方式可以实现Bidirectional Connection

- 基于Glacier2的Router默认自动实现Bidirectional Connection
- 手工配置

接下来主要讲解如何手工配置实现Birectional Connection，example代码可以在demo/Ice/bidir目录下找到

## Slice接口定义

````java
#pragma once
#include <Ice/Identity.ice>

module Demo {
interface CallbackReceiver {
    void callback(int num);
};
interface CallbackSender {
    void addClient(Ice::Identity ident);
};
};
````



## 客户端实现

客户端需要执行以下几步来实现Bidirectional Connection

1. 创建一个用于接收Callback请求的Object Adapter，这个Adapter不需要设置任何名称ID和Endpoint只是用于接收Callback请求
2. 在Adapter中注册Callback接收的对象
3. 激活Adapter
4. 将Adapter设置为Proxy连接的Adapter，关联Proxy连接和接收Callback的Adapter，这样回调请求才能被正常分派到Callback对象
5. 将Callback对象的Identity发送给服务端，服务通过Identity获得Callback对象的Proxy进行请求回调

示例代码如下

````C++
#include <IceUtil/IceUtil.h>
#include <Ice/Ice.h>
#include <Callback.h>

using namespace std;
using namespace Demo;

class CallbackReceiverI : public CallbackReceiver {
public:
    virtual void callback(Ice::Int num, const Ice::Current&) {
        cout << "received callback #" << num << endl;
    }
};

class CallbackClient : public Ice::Application {
public:
    virtual int run(int, char*[]) {
        CallbackSenderPrx proxy = CallbackSenderPrx::checkedCast(
          	communicator()->stringToProxy("sender:tcp -p 10000"));
        if(!proxy) {
            cerr << appName() << ": invalid proxy" << endl;
            return EXIT_FAILURE;
        }

        //创建Callback Adapter
        Ice::ObjectAdapterPtr adapter = communicator()->createObjectAdapter("");
        Ice::Identity ident;
        ident.name = Ice::generateUUID();
        ident.category = "";
        CallbackReceiverPtr cr = new CallbackReceiverI;
        adapter->add(cr, ident);
        adapter->activate();
        
        //关联Callback Adapter
        proxy->ice_getConnection()->setAdapter(adapter);
        proxy->addClient(ident);
      
        communicator()->waitForShutdown();

        return EXIT_SUCCESS;
    }
};

int main(int argc, char* argv[]) {
    CallbackClient app;
    return app.main(argc, argv);
}
````

最后一步addClient需要注意，通常实现回调传给服务端的是一个Proxy对象，但addClient发送Callback对象的Identity给服务端，这是因为为了实现Bidirectional Connection绑定的Adapter并没有设置任何Endpoint服务端无法连接Endpoint获得对象Proxy，所以服务端只能拿到Identity通过Current参数的连接调用createProxy创建回调对象代理。

## ACM配置

[Active Connection Management](https://doc.zeroc.com/display/Ice36/Active+Connection+Management) (ACM)是Ice的连接管理功能，它会自动关闭一些空闲无用的连接，正常情况下应用不需要关心，但当使用Bidirectional连接时通常会长时间等待服务端的Callback请求，这时候ACM策略就会影响Bidirectional连接的行为，通常会认为是闲置连接自动关闭，在Ice3.6之前的版本如果使用Bidirectional连接需要将ACM功能关闭，Ice3.6之后可以在启用ACM功能的同时保持Bidirectional连接，只需要enable客户端heartbeat机制即可，可以通过两种方式设置

- 全局配置

````
Ice.ACM.Client.Heartbeat=Always
````

- 通过连接setACM接口单独设置

````
proxy->ice_getCachedConnection()->setACM(IceUtil::None, IceUtil::None, Ice::HeartbeatAlways);
````

Bidirectional连接官方推荐配置如下

````
# Client
Ice.ACM.Close=0      # CloseOff
Ice.ACM.Heartbeat=3  # HeartbeatAlways
Ice.ACM.Timeout=30
 
# Server
Ice.ACM.Close=4      # CloseOnIdleForceful
Ice.ACM.Heartbeat=0  # HeartbeatOff
Ice.ACM.Timeout=30
````

客户端disable自动关闭连接并enable连接心跳；服务端enable自动关闭空闲连接并disable连接心跳，详细参考[ACM配置](https://doc.zeroc.com/display/Ice36/Active+Connection+Management#ActiveConnectionManagement-bidir)

## 服务端实现

要实现对Bidirectional连接的回调，服务端需要实现以下几步

1. 通过addClient接口获得客户端提供的Callback对象的Identity
2. 通过连接的createProxy方法创建Callback对象的代理

````c++
#include <Ice/Ice.h>                                                                    
#include <Callback.h>

using namespace std;
using namespace Demo;                                                                   

class CallbackSenderI : public Demo::CallbackSender {                                   
public:                                                                                 
    virtual void addClient(const Ice::Identity&, const Ice::Current&) { 
      	static int num = 0;
        CallbackReceiverPrx client = CallbackReceiverPrx::uncheckedCast(curr.con->createProxy(ident));
        client->callback(num++);                                                             
    }                                                                                   
};
    
class CallbackServer : public Ice::Application {                                        
public:                                                                                 
    virtual int run(int, char*[]) {                                                     
        Ice::ObjectAdapterPtr adapter = communicator()->createObjectAdapter("tcp -p 10000");
        CallbackSenderIPtr sender = new CallbackSenderI(communicator());                
        adapter->add(sender, Ice::stringToIdentity("sender"));                          
        adapter->activate();                                                            
        
        communicator()->waitForShutdown();
        
        return EXIT_SUCCESS;
    }                                                                                   
};  
                                                                                        
int main(int argc, char* argv[]) {
    CallbackServer app;                                                                 
    return app.main(argc, argv, "config.server");
}
````

### Fixed Proxies

通过连接的createProxy方法创建的代理称为Fixed Proxies，这种代理只能运行与服务端，而且通常不能设置其他参数例如ice_timeout，否则都会抛出FixedProxyException异常，代理的生命周期随着连接关闭而结束，使用已关闭连接的上的代理将会抛出CloseConnectionException异常

## Bidirectional连接的限制

- 只能应用于TCP/SSL协议的连接
- 必须绑定已存在的连接并且继承连接的配置，fixed proxy本身不能修改相关网络配置，不过在一个fixed proxy上进行oneway或者twoway调用是合法的，在一个secure连接上创建的fixed proxy调用ice_secure也是合法的，但前提是连接本身是secure的否则会出现NoEndpointException异常
- 如果使用Glacier2 router不需要手工配置客户端到router的Bidirectional连接，Ice内部运行时会自动配置



