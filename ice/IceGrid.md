# IceGrid

IceGrid主要为Ice应用提供定位服务和服务管理，能够管理调度Ice应用集合，官方称为网格，功能还包括应用的部署安装和更新，服务器状态和负载跟踪，节点迁移和快速部署等。

- Location Service

  定位服务实现间接代理访问，客户端无需硬编码绑定服务Endpoint以实现灵活的弹性伸缩，如果不需要IceGrid的其他特色功能可以用IceDiscovery代替IceGrid实现定位服务

- On-demand Server Activation

  按需启动，已配置的节点（Node）默认不启动，当客户端向IceGrid请求一个服务代理时，如果IceGrid发现这个服务存在但没有启动，就会自动激活Adapter所属的server

- Application distribution

  提供Ice应用分布服务器的部署功能，基于IcePatch2轻松实现文件系统的同步服务

- Replication and load balancing

  IceGrid的负载均衡是基于Adapter的，IceGrid支持多台服务器的Adapter组成一个逻辑上组Adapter，当客户端使用间接绑定的时候，基于IceGrid的负载均衡策略获得他们任意一个的对象代理

- Sessions and resource allocation

  客户端可以建立一个session来独占某个对象或者服务器，阻止其他的客户端使用这个分配的资源，直到客户端释放或者session超时

- Automatic failover

  当有多个节点的情况下，Ice支持失败重试和故障恢复

- Dynamic queries

  支持客户端动态查询Adapter的Endpoint信息，有客户端选择使用连接那个Adapter的服务

- Status monitoring

  提供状态监控Slice接口，可以使用监控接口来整合现有的监控系统

- Administration

  提供命令行方式和图形化方式管理你的应用IceGrid集群 

- Deployment

  可以通过xml文件来简单描述部署系统


## 如何部署IceGrid

基于xml部署描述文件进行IceGrid应用部署，应用部署描述文件包含以下几个组成部分

- Replica groups

  replica group是一组同类对象Adapter的逻辑组集合，一个应用能够包含任意多个replica group，每个replica group的id必须唯一

- Nodes

  一个应用必须包含一个或多个服务节点

- Servers

  每个服务包含一个唯一的服务名和服务可执行文件路径，可以包含一个或多个对象Adapter

- Object adapters

  一个对象Adapter包含一个Endpoint和任意多个well-known objects，对象Adapter还能指定Replica groups的标识作为Replica groups成员

- Objects

  well-known objects指的就是通过对象标识能够唯一定位的对象，Registry通过维护全局对象列表实现请求定位和分派


## 启动IceGrid Registry

启动之前需要先准备Registry的配置

````
#registry.conf

#提供节点连接接入Registry的端口地址
IceGrid.Registry.Client.Endpoints=tcp -p 4061
IceGrid.Registry.Server.Endpoints=tcp #用于servers注册adapter，可动态分配
IceGrid.Registry.Internal.Endpoints=tcp #内部连接端口，可动态分配

#禁用权限验证
IceGrid.Registry.AdminPermissionsVerifier=IceGrid/NullPermissionsVerifier

#registry的数据目录，保存节点注册数据等，必须预先创建
IceGrid.Registry.Data=data/registry

#允许动态注册，生产环境通常禁用
IceGrid.Registry.DynamicRegistration=1
````

启动Registry服务

````
$icegridregistry --Ice.Config=registry.conf --nochdir --daemon
````

## 服务端

有了IceGrid Registry服务，服务端无需再绑定一个明确的地址，只需要动态分配即可

参考Helloworld应用，C++代码简单修改如下：

````c++
#include <Ice/Ice.h>
#include <Printer.h>

using namespace Demo;

class ServerApplication : virtual public Ice::Application {
    virtual int run(int, char *[]) {
        //只需要指定Adapter名，AdapterId由配置文件指定，动态分配Endpoint
        Ice::ObjectAdapterPtr adapter =
            communicator()->createObjectAdapter("PrinterAdapter");
        Ice::ObjectPtr object = new PrinterI;
        adapter->add(object, ic->stringToIdentity("Printer"));
        adapter->activate();
        ic->waitForShutdown();
        
        return EXIT_SUCCESS;
   }
};

int main(int argc, char* argv[]) {
    ServerApplication app;
    return app.main(argc, argv);
}
````

参考Helloworld应用，Java代码简单修改如下：

````java
public class ServerApplication extends Ice.Application {
    @Override
    public int run(String[] args) {
        //只需要指定Adapter名，AdapterId由配置文件指定，动态分配Endpoint
        Ice.ObjectAdapterPtr adapter =
            communicator()->createObjectAdapter("PrinterAdapter");
        Ice.ObjectPtr object = new PrinterI;
        adapter->add(object, ic->stringToIdentity("Printer"));
        adapter->activate();
        
        communicator().waitForShutdown();

        return 0;
    }

    public static void main(String[] args) {
        ServerApplication app = new ServerApplication();

        System.exit(app.main("ServerApplication", args));
    }
}
````

服务节点配置

````
#server.conf
PrinterAdapter.AdapterId=PrinterAdapter
PrinterAdapter.Endpoints=tcp
Ice.Default.Locator=IceGrid/Locator:tcp -h dev01 -p 4061
````

启动服务

````
#C++
$./server --Ice.Config=server.conf

#Java
$java -classpath classes:$ICE_HOME/lib/Ice.jar ServerApplication --Ice.Config=server.conf

#通过icegridadmin可以查看icegrid集群情况
$icegridadmin --Ice.Default.Locator="IceGrid/Locator:tcp -h dev01 -p 4061"
user id: guest
password: 
Ice 3.6.3  Copyright (c) 2003-2016 ZeroC, Inc.
>>> application list
>>> server list
>>> adapter list
PrinterAdapter #PrinterAdapter已经成功注册
>>> object list
IceGrid/Query
IceGrid/Locator
IceGrid/Registry
IceGrid/LocatorRegistry
IceGrid/InternalRegistry-Master
>>> 
````

## 客户端

客户端同样不需要指定服务端的Adapter的Endpoint地址，只需要指定Adapter标识和对象标识

参考Helloworld应用，C++代码简单修改如下

````c++
#include <Ice/Ice.h>
#include <Printer.h>

using namespace Demo;

class ClientApplication : virtual public Ice::Application {
    virtual int run(int, char *[]) {
        //只需要指定服务的Adapter和对象标识，不需要再指定目标服务Adapter的Endpoint
        Ice::ObjectPrx proxy =
            communicator()->stringToProxy("Printer@PrinterAdapter");
        PrinterPrx printer = PrinterPrx::checkedCast(proxy);
        if (!printer)
            throw "Invalid proxy";

        printer->printString("Hello World!");
        
        return EXIT_SUCCESS;
   }
};

int main(int argc, char* argv[]) {
    ClientApplication app;
    return app.main(argc, argv);
}
````

参考Helloworld应用，Java代码简单修改如下

````java
public class ClientApplication extends Ice.Application {
    @Override
    public int run(String[] args) {
        //只需要指定服务的Adapter和对象标识，不需要再指定目标服务Adapter的Endpoint
        Ice.ObjectPrx base = communicator().stringToProxy("Printer@PrinterAdapter");
        Demo.PrinterPrx printer = Demo.PrinterPrxHelper.checkedCast(base);
        if (printer == null)
        	throw new Error("Invalid proxy");
 
        printer.printString("Hello World!");
      
        return 0;
    }

    public static void main(String[] args) {
        ClientApplication app = new ClientApplication();

        System.exit(app.main("ClientApplication", args));
    }
}
````

启动客户端

````
#C++
./client --Ice.Default.Locator="IceGrid/Locator:tcp -h dev01 -p 4061"

#Java
$java -classpath classes:$ICE_HOME/lib/Ice.jar ClientApplication --Ice.Default.Locator="IceGrid/Locator:tcp -h dev01 -p 4061"
````

客户端直接命令行参数指定Ice.Default.Locator地址



## IceGrid应用部署

以上示例代码只是用到了IceGrid的Location Service功能，下面我们来看看是IceGrid集群是如何部署管理的

### 应用部署描述文件

IceGrid集群定义描述文件，包括服务、节点和负载均衡路由策略等

````xml
<!-- app.xml -->
<icegrid>
    <application name="Printer">
        <node name="Node1">
            <server id="PrinterServer1" exe="./server" activation="on-demand">
                  <adapter name="PrinterAdapter" 
                           id="PrinterAdapter" 
                           endpoints="tcp">
                       <object identity="Printer" type="::Demo::Printer"/>
                  </adapter>
                  <!-- 跟踪打印网络连接日志 -->
                  <property name="Ice.Trace.Network" value="1"/>
            </server>
        </node>
    </application>
</icegrid>
````

定义一个Printer的应用

- 包含一个节点Node1
- 节点包含一个服务，并指定了可执行程序为./server，激活方式为on-demand按需启动
- 服务包含一个Adapter PrinterAdapter
- PrinterAdapter包含一个Printer Servant

### 部署Printer应用

用icegridadmin工具连接ice grid registry进行应用部署

````
$icegridadmin --Ice.Default.Locator="IceGrid/Locator:tcp -h dev01 -p 4061"
user id: guest
password: 
Ice 3.6.3  Copyright (c) 2003-2016 ZeroC, Inc.
>>> application add app.xml
>>> application list
Printer #已成功部署Printer应用
>>> node ping Node1
node is down #服务节点还未启动
````

### 启动Node

Node启动前先进行Node配置

````
# node1.conf
IceGrid.Node.Endpoints=tcp
IceGrid.Node.Name=Node1
IceGrid.Node.Data=data/node1 #节点数据目录需要预先创建，该目录会用来保存同步配置等
Ice.Default.Locator=IceGrid/Locator:tcp -p 4061 #registry地址
````

启动节点

````
$icegridnode --Ice.Config=node1.conf --nochdir --daemon
````

到此为止icegrid应用已经部署完毕，当有客户端请求时icegridnode会自动激活启动./server程序接收处理请求

### 管理应用

````
$icegridadmin --Ice.Default.Locator="IceGrid/Locator:tcp -h dev01 -p 4061"
user id: guest
password: 
Ice 3.6.3  Copyright (c) 2003-2016 ZeroC, Inc.
>>> node ping Node1
node is up
>>> server help
server list               List all registered servers.
server remove ID          Remove server ID.
server describe ID        Describe server ID.
server properties ID      Get the run-time properties of server ID.
server property ID NAME   Get the run-time property NAME of server ID.
server state ID           Get the state of server ID.
server pid ID             Get the process id of server ID.
server start ID           Start server ID.
server stop ID            Stop server ID.
server patch ID           Patch server ID.
server signal ID SIGNAL   Send SIGNAL (e.g. SIGTERM or 15) to server ID.
server stdout ID MESSAGE  Write MESSAGE on server ID's stdout.
server stderr ID MESSAGE  Write MESSAGE on server ID's stderr.
server show [OPTIONS] ID [log | stderr | stdout | LOGFILE ]
                          Show server ID Ice log, stderr, stdout or log file LOGFILE.
                          Options:
                           -f | --follow: Wait for new data to be available
                           -t N | --tail N: Print the last N log messages or lines
                           -h N | --head N: Print the first N lines (not available for Ice log)
server enable ID          Enable server ID.
server disable ID         Disable server ID (a disabled server can't be
                          started on demand or administratively).
>>> 
````

### IceGrid模板

应用部署文件通常需要配置几十台服务器，大量的Node配置都是相同的，如果按照先前的应用配置文件进行配置需要写大量重复的配置，IceGrid应用部署文件支持模板配置，一个server-template示例如下:

````xml
<!-- app.xml -->
<icegrid>
    <application name="Printer">
         <server-template id="PrinterServer">
            <!-- 模板参数 -->
            <parameter name="index"/>
            <server id="PrinterServer${index}" exe="./server" activation="on-demand">
                  <adapter name="PrinterAdapter" 
                           endpoints="tcp">
                       <object identity="Printer" type="::Demo::Printer"/>
                  </adapter>
                  <!-- 跟踪打印网络连接日志 -->
                  <property name="Ice.Trace.Network" value="1"/>
            </server>
        </server-template>
      
        <!-- Java程序模板示例 -->
        <server-template id="PrinterServerForJava">
            <parameter name="index"/>
            <server id="PrinterServer${index}" activation="on-demand" exe="java">
                <option>-classpath</option>
                <option>classes:$ICE_HOME/lib/Ice.jar</option>
                <option>com.example.ServerApplication</option>
                <adapter name="PrinterAdapter" endpoints="tcp" id="${server}.PrinterAdapter">
                </adapter>
            </server>
        </server-template>

        <node name="Node1">
            <server-instance template="PrinterServer" index="1" />
        </node>
    </application>
</icegrid>
````

这样可以轻松负责多个Node配置，server-template同时还支持模板参数和server-instance个性化参数配置

````xml
<icegrid>
    <application>
        ...
        <node name="Node2"> 
            <server-instance template="PrinterServer" index="2"> 
                <properties>
                    <property name="Ice.Trace.Network" value="2"/>
                </properties>
            </server-instance>
        </node> 
    </application> 
</icegrid> 
````

IceGrid Registry还能通过参数IceGrid.Registry.DefaultTemplates可以指定一个默认的模板文件，用于将常用的模板抽离成公共的模板文件，简化应用配置

### 负载均衡

IceGrid的Adapter的负载均衡是基于Object Adapter Replication实现的，它是IceGrid Request Location Service的一种实现方式，通过定义Replica Group并将指定的Adapter加入到Replica Group中实现请求的Location和Load Balancing服务，简单示例如下

````xml
<!-- app.xml -->
<icegrid>
    <application name="Printer">
        <replica-group id="PrinterReplicatedAdapter">
            <load-balancing type="adaptive" load-sample="5" n-replicas="2"/>
            <object identity="Printer" type="::Demo::Printer"/>
        </replica-group>
        <node name="Node1">
            <server id="PrinterServer1" exe="./server" activation="on-demand">
                  <adapter name="PrinterAdapter" 
                           replica-group="PrinterReplicatedAdapter"
                           endpoints="tcp">
                       <object identity="Printer" type="::Demo::Printer"/>
                  </adapter>
            </server>
        </node>
    </application>
</icegrid>
````

load-balancing负载均衡标签详细说明如下

- type

  指定负载均衡算法Random(随机)|Adaptive(动态负载)|Round Robin(轮转)|Ordered(优先级)

- load-sample

  节点负载采样周期单位分钟

- n-replicas

  指示Registry一次返回多少个匹配的Endpoint Object Adapter，如果设置大于1则每次最多返回N个Adapter，如果设置为0则返回所有的Adapter

## Registry 主备部署

IceGrid Registry支持主备模式提高Registry的可用性，当Slaver Registry将实时同步Master的Node信息，当Master出现故障时客户端会自动切到Slaver Registry，Master配置示例如下

````
IceGrid.InstanceName=IceGrid
IceGrid.Registry.Client.Endpoints=default -p 12000
IceGrid.Registry.Server.Endpoints=default
IceGrid.Registry.Internal.Endpoints=default
IceGrid.Registry.Data=data/registry-master
````

Slaver示例配置如下

````
Ice.Default.Locator=IceGrid/Locator:default -p 12000 #指定master endpoint
IceGrid.Registry.Client.Endpoints=default -p 12001 #slaver endpoint
IceGrid.Registry.Server.Endpoints=default
IceGrid.Registry.Internal.Endpoints=default
IceGrid.Registry.Data=data/registry-replica1
IceGrid.Registry.ReplicaName=Replica1 #必须指定ReplicaName
````

客户端Locator配置需要将master和slaver的endpoint同时加到Registry的Locator中，当master故障时会自动切换到endpoint，在生产环境至少包含两个endpoint

````
Ice.Default.Locator=IceGrid/Locator:default -p 12000:default -p 12001
````

