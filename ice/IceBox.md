# IceBox

IceBox是Ice应用服务的容器，通过配置来动态加载和集中管理Ice应用服务，基于IceBox让开发只需要关注服务组件业务逻辑的开发，而不需要关注Ice应用框架本身的开发工作，IceBox服务组件不再是一个独立的可执行程序，而是一个可动态加载的组件，如C/C++的动态库，用IceBox服务代替传统Ice服务应用有以下几个好处

- 同一个IceBox加载的服务之间的调用可以进行优化
- 只需要修改配置就可以在一个应用中组合各种服务而不需要重新编译
- 对比单体服务应用，同一个进程中加载多个服务可以有效节约系统资源
- IceBox服务支持集中管理，并提供管理接口进行二次开发
- IceBox支持IceGrid集成


## IceBox服务开发

### IceBox服务接口

实现IceBox服务需要继承IceBox::Service并实现以下两个接口

- void start(string name, Ice::Communicator communicator, Ice::StringSeq args)

  服务启动初始化方法，通常包括创建对象Adapter和Servant，name和args分别为Adapter名和配置参数，communicator则是服务管理器为service创建的Ice::Comminicator对象，通常IceBox内的多个服务会共享一个communicator对象

- void stop()

  服务停止方法，处理服务停止并释放所有引用资源，通常会在stop方法中调用对象Adapter的deactivates方法停止对象Adapter或者调用waitForDeactivate方法来确保所有未完成请求在资源清除之前得到处理，而communicator对象则有服务管理器负责释放

### IceBox Service Example C++

````c++
#include <Ice/Ice.h>
#include <Printer.h>
 
using namespace std;
 
class PrinterServiceI : public ::IceBox::Service {
public:
    void start(
        const string& name,
        const Ice::CommunicatorPtr& communicator,
        const Ice::StringSeq& args) {
        _adapter = communicator->createObjectAdapter(name);
        Ice::ObjectPtr object = new PrinterI;
        _adapter->add(object, communicator->stringToIdentity("Printer"));
        _adapter->activate();
    }
    void stop() {
        _adapter->deactivate();
    }

private:
    Ice::ObjectAdapterPtr _adapter;
};

//Service Entry Point
extern "C" {
    IceBox::Service* create(Ice::CommunicatorPtr communicator) {
        return new PrinterServiceI;
    }
}
````

Service Entry Point要求指定编译器以c语言编译链接规则编译Entry Point代码，因为create函数需要通过动态库方式加载，create函数返回一个PrinterServiceI实例

### IceBox Service Example Java

````java
public class PrinterServiceI implements IceBox.Service {
    public void start(String name, Ice.Communicator communicator, String[] args) {
        _adapter = communicator.createObjectAdapter(name);//创建一个与服务同名的Adapter
        Ice.Object object = new PrinterI();
        _adapter.add(object, communicator.stringToIdentity("Printer"));
        _adapter.activate();
    }
 
    public void stop() {
        _adapter.deactivate();
    }
 
    private Ice.ObjectAdapter _adapter;
}
````

注：IceBox Service实现类要求必须有一个默认的构造函数



### IceBox Service配置

1. 服务配置

```
#IceBox.Service.name=entry_point [args]
IceBox.Service.Printer=printer:create ;for c++
IceBox.Service.Printer=PrinterServiceI Printer ;for java
```

- name服务名，用实际服务名代替，对应start方法中的name参数
- entry_point配置服务的加载Entry Point
  - C++程序格式为*library[,version]:symbol*
  - Java程序格式为[ class directory or JAR file]:class_name 如:/opt/libs/service.jar:PrinterServiceI
- args第一个参数如果格式为*--name=value*则将作为property设置communicator对象，其他参数将作为start方法的args传递

2. 指定IceBox服务加载顺序

````
IceBox.LoadOrder=Service1,Service2 #多个服务有效
````

3. 指定共享Communicator

````
IceBox.UseSharedCommunicator.Printer=1 #多个服务有效
````

4. 是否继承IceBox服务的属性配置

````
IceBox.InheritProperties=1 #设置为1则IceBox服务将默认继承IceBox配置文件的属性配置
````



简单完整地示例配置如下

````
#icebox.conf
IceBox.Service.Printer=printer:create --Ice.Trace.Network=1
Printer.Endpoints=default -p 10000
IceBox.UseSharedCommunicator.Printer=1
Ice.Trace.Network=2
IceBox.InheritProperties=1
````

## 启动IceBox服务

````
$ icebox --Ice.Config=icebox.conf
````

## IceGrid集成IceBox服务

IceBox能够非常简单地集成到IceGrid集群部署，IceBox部署描述文件如下

````xml
<icegrid>
    <application name="Printer">
        <node name="Node1">
            <icebox id="IceBoxServer" exe="icebox" activation="on-demand">
                <service name="PrinterService" entry="printer:create">
                    <adapter name="${service}" endpoints="tcp"/>
                </service>
            </icebox>
        </node>
    </application>
</icegrid>
````

这是一个最基本的部署描述文件定义，通过模板可以定义个更加通用的配置文件如下

````xml
<icegrid>
    <application name="Printer">
        <service-template id="ServiceTemplate">
            <parameter name="name"/>
            <service name="${name}" entry="DemoService:create">
                <adapter name="${service}" endpoints="default"/>
                <property name="${name}.Identity" value="${server}-${name}"/>
            </service>
        </service-template>
        <server-template id="ServerTemplate">
            <parameter name="id"/>
            <icebox id="${id}" endpoints="default" exe="icebox" activation="on-demand">
                <service-instance template="ServiceTemplate" name="PrinterService"/>
            </icebox>
        </server-template>
        <node name="Node1">
            <server-instance template="ServerTemplate" id="IceBoxServer"/>
        </node>
    </application>
</icegrid>
````

