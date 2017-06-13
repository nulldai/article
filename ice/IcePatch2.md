# IcePatch2

IcePatch2为应用提供安全高效的目录同步服务，它解决了应用在分布式环境的升级部署问题，通过IcePatch2只需要简单配置就可以在分布式各节点之间同步文件，类似于rsync同步工具，但IcePatch2更加安全高效，同时提供Api开发接口支持二次开发，而且能够方便与IceGrid进行整合。

IcePatch2包含以下几个组件

- icepatch2server IcePatch服务端
- icepatch2client IcePatch客户端
- icepatch2calc 用于压缩文件和计算目录checksums
- 支持Slice API和C++库用于定制IcePatch2客户端

## 简单同步

### 第一步使用icepatch2calc处理目录

假设有一个目录如下
![](https://doc.zeroc.com/download/attachments/16716566/An_example_data_directory.gif?version=1&modificationDate=1432841391000&api=v2)

使用icepatch2calc命令对目录进行压缩和checksums计算

````
$ icepatch2calc .
````

处理之后的目录如下
![](https://doc.zeroc.com/download/attachments/16716566/Contents_of_data_directory_after_running_icepatch2calc.gif?version=1&modificationDate=1432841399000&api=v2)

icepatch2calc会压缩目录下所有非空文件，同时在根目录下生成一个IcePatch2.sum文件用于记录文件的checksums信息，文件内容如下

````
. 3a52ce780950d4d969792a2559cd519d7ee8c727 -1
./bin bd362140a3074eb3edb5e4657561e029092c3d91 -1
./bin/hello 77b11db586a1f20aab8553284241bb3cd532b3d5 70
./emptyFile 082c37fc2641db68d195df83844168f8a464eada 0
./nonEmptyFile aec7301c408e6ce184ae5a34e0ea46e0f0563746 72
````

### 第二步运行icepatch2server

````
$ icepatch2server --IcePatch2.Directory=./ --IcePatch2.InstanceName=IcePatch2Server --IcePatch2.Endpoints="tcp -p 10000"
````
- IcePatch2.Directory参数指定同步目录
- IcePatch2.Endpoints参数指定监听端口

还可以通过Ice.Config参数指定配置文件，配置更多参数

### 第三步运行icepatch2client

````
$ icepatch2client --IcePatch2Client.Proxy="IcePatch2Server/server:tcp -p 10000" .
````
同步服务端文件到当前目录，IcePatch2Client.Proxy指定服务端的Endpoint，首次同步还可以增加参数-t来指定进行全面同步，icepatch2client主要完成以下几步操作

1. 如果本地不存在IcePatch2.sum文件则创建，并进行全面同步
2. 对比本地IcePatch2.sum文件和远程服务端的IcePatch2.sum文件
- 如果本地存在远程不存在则删除本地文件，可以设置IcePatch2Client.Remove参数不进行本地删除
- 如果本地不存在远程存在则拉取文件
- 如果本地文件checksum值与远程服务端的checksum值不同则更新文件

## IceGrid集成IcePatch2进行应用部署

- 首先假设两台服务器A和服务器B  
- 同时这两台服务均已安装Ice环境  
- A部署Registry、Node1(icepatch2server/PrinterServer)  
- B部署Node2(PrinterServer)  

### IcePatch2 Server Template

````
<server-template id="IcePatch2">
    <parameter name="instance-name" default="${application}.IcePatch2"/>
    <parameter name="endpoints" default="default"/>
    <parameter name="directory"/>
 
    <server id="${instance-name}" exe="icepatch2server"
        application-distrib="false" activation="on-demand">
        <adapter name="IcePatch2" endpoints="${endpoints}">
            <object identity="${instance-name}/server" type="::IcePatch2::FileServer"/>
        </adapter>
        <property name="IcePatch2.InstanceName" value="${instance-name}"/>
        <property name="IcePatch2.Directory" value="${directory}"/>
    </server>
</server-template>
````
IcePatch2 Server Template是官方文档中提供的服务模板，用于快速创建IcePatch2 Server  
模板有三个参数，只有directory必传参数，用于指定icepatch2server的同步目录

### 将模板加入到IceGrid的Application配置中

````xml
<icegrid>
    <application name="Printer">
        <!-- IcePatch2 Server的模板 -->
        <server-template id="IcePatch2Server">
            <parameter name="instance-name" default="${application}.IcePatch2"/>
            <parameter name="endpoints" default="default"/>
            <parameter name="directory"/>
            <server id="${instance-name}.server" exe="icepatch2server" application-distrib="false" activation="on-demand">
                <adapter name="IcePatch2" endpoints="${endpoints}">
                    <object identity="${instance-name}/server" type="::IcePatch2::FileServer"/>
                </adapter>
                <properties>
                    <property name="IcePatch2.InstanceName" value="${instance-name}"/>
                    <property name="IcePatch2.Directory" value="${directory}"/>
                </properties>
            </server>
        </server-template>
        
        <!-- 用于指定应用公共的部署文件，不指定directory默认下载IcePatch2.Directory目录下所有文件 -->
        <distrib>
        	<directory>libs</directory>
        </distrib>
        
        <!-- PrinterServer服务模板 -->
        <server-template id="PrinterServer">
            <parameter name="index"/>
            <server id="PrinterServer${index}" exe="${server.distrib}/bin/server" activation="on-demand">
                  <env>LD_LIBRARY_PATH=${application.distrib}/libs:$LD_LIBRARY_PATH</env>
                  <adapter name="PrinterAdapter" 
                           replica-group="PrinterReplicatedAdapter"
                           endpoints="tcp">
                      <!-- <object identity="Printer" type="::Demo::Printer"/> -->
                  </adapter>
              	  <!-- 用于指定服务部署需要下载的文件 -->
                  <distrib>
        			<directory>printer</directory>
        		 </distrib>
            </server>
        </server-template>

        <!-- 负载均衡组，定义负载均衡策略 -->
        <replica-group id="PrinterReplicatedAdapter">
            <load-balancing type="adaptive" load-sample="5" n-replicas="2"/>
            <object identity="Printer" type="::Demo::Printer"/>
        </replica-group>

        <!-- 节点定义 -->
        <node name="Node1">
            <server-instance template="IcePatch2Server" directory="data/packages/" />
            <server-instance template="PrinterServer" index="1" />
        </node>

        <node name="Node2">
            <server-instance template="PrinterServer" index="2" />
        </node>
    </application>
</icegrid>
````
在节点数据目录下会存储应用的或者服务的distributions文件，下文的目录结构也能看出来在node的数据目录和服务的数据目录分别有两个文件夹distrib，这个文件夹就是客户端同步的数据文件夹，有两个保留变量用于引用distrib文件夹路径

- application.distrib 用于引用应用的distrib目录 
- server.distrib 用于引用server的distrib目录

### 启动IceGrid Registry服务

registry服务配置

````
#registry.conf
IceGrid.Registry.Client.Endpoints=tcp -p 4061
IceGrid.Registry.Server.Endpoints=tcp
IceGrid.Registry.Internal.Endpoints=tcp
IceGrid.Registry.AdminPermissionsVerifier=IceGrid/NullPermissionsVerifier
IceGrid.Registry.Data=data/registry #目录必须手工创建
IceGrid.Registry.DynamicRegistration=1
````

启动icegridregistry服务

````
$icegridregistry --Ice.Config=registry.conf --nochdir --daemon
````

### 启动IceGrid Node服务

节点配置

````
#node1.conf
IceGrid.Node.Endpoints=tcp
IceGrid.Node.Name=Node1
IceGrid.Node.Data=data/node1 #目录必须手工创建
Ice.Default.Locator=IceGrid/Locator:tcp -p 4061
#
#node2.conf
IceGrid.Node.Endpoints=tcp
IceGrid.Node.Name=Node2
IceGrid.Node.Data=data/node2 #目录必须手工创建
Ice.Default.Locator=IceGrid/Locator:tcp -p 4061
````

启动icegridnode服务

````
$icegridnode --Ice.Config=node1.conf --nochdir --daemon
$icegridnode --Ice.Config=node2.conf --nochdir --daemon
````

### 启动IceGrid Admin查看集群状态

客户端配置

````
#client.conf
Ice.Default.Locator=IceGrid/Locator:tcp -h dev01 -p 4061
Ice.Override.Timeout=10000
IceGridAdmin.Username=dev
IceGridAdmin.Password=dev
````

查看集群状态

````
$ icegridadmin --Ice.Config=client.conf
Ice 3.6.3  Copyright (c) 2003-2016 ZeroC, Inc.
>>> node list
Node1
Node2
>>> node ping Node1
node is up
>>> node ping Node2
node is up
>>> server list
Printer.IcePatch2.server
PrinterServer1
PrinterServer2
>>> 
````

### 手工初始化icepatch2server数据目录

````
$icepatch2calc data/packages
````

初始化完整体目录结构如下

````
data/
├── packages
│   ├── libs
│   │   ├── libIce.so
│   │   └── libIce.so.bz2
│   ├── printer
│   │   ├── server
│   │   └── server.bz2
│   └── IcePatch2.sum
├── node1
│   ├── distrib
│   ├── servers
│   │   ├── Printer.IcePatch2.server
│   │   │   ├── config
│   │   │   │   └── config
│   │   │   ├── dbs
│   │   │   ├── distrib
│   │   │   └── revision
│   │   └── PrinterServer1
│   │       ├── config
│   │       │   └── config
│   │       ├── dbs
│   │       ├── distrib
│   │       └── revision
│   └── tmp
└── node2
    ├── distrib
    ├── servers
    │   └── PrinterServer2
    │       ├── config
    │       │   └── config
    │       ├── dbs
    │       ├── distrib
    │       └── revision
    └── tmp
````

### 启动icepatch2server

````
$ icegridadmin --Ice.Config=client.conf -e "server start Printer.IcePatch2.server"
````
注:如果数据目录有变更需要重启icepatch2server

### 部署Node1上的PrinterServer服务

````
$ icegridadmin --Ice.Config=client.conf -e "server patch PrinterServer1"
````

patch完之后的目录结构如下

````
data/
├── packages
│   ├── libs
│   │   ├── libIce.so
│   │   └── libIce.so.bz2
│   ├── printer
│   │   ├── server
│   │   └── server.bz2
│   └── IcePatch2.sum
├── node1
│   ├── distrib
│   │   └── Printer #已同步data/packages/libs目录文件
│   │       ├── libs
│   │       │   └── libIce.so
│   │       └── IcePatch2.sum
│   ├── servers
│   │   ├── Printer.IcePatch2.server
│   │   │   ├── config
│   │   │   │   └── config
│   │   │   ├── dbs
│   │   │   ├── distrib
│   │   │   └── revision
│   │   └── PrinterServer1
│   │       ├── config
│   │       │   └── config
│   │       ├── dbs
│   │       ├── distrib
│   |       |   ├── printer #已同步data/packages/printer目录文件
│   │       |   |   └── server
│   │       |   └── IcePatch2.sum
│   │       └── revision
│   └── tmp
└── node2
    ├── distrib
    ├── servers
    │   └── PrinterServer2
    │       ├── config
    │       │   └── config
    │       ├── dbs
    │       ├── distrib
    │       └── revision
    └── tmp
````
patch时需要停止服务，最好disable避免自动重启，patch完之后再enable服务