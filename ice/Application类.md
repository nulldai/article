# Application类

开发Ice应用的过程中，如果不使用IceBox，无论是开发客户端应用还是服务端应用，都需要进行一些重复的操作如初始化Ice::Communicator、捕获异常、处理系统信号、退出销毁等，Ice运行时库提供了Ice::Application类来完成这些繁琐重复的工作，Ice::Application类本身实现非常简单，主要是初始化Ice::Communicator和系统信号处理，但确非常实用。

Ice::Application类使用主要注意以下两点  

- Ice::Application类本身是一个抽象类，其run()函数为纯虚函数，应用继承Ice::Application在run函数中实现应用初始化代码
- Ice::Application是一个单例类，一个进程只允许一个Ice::Application实例，同时Application只会初始化一个Ice::Communicator对象，如果需要多个需要手工创建

## 主要接口说明

````c++
// Application的入口函数，提供了丰富的初始化方式，一般使用第一个
// 直接透传主函数参数
int main(int, char*[]);
int main(int, char*[], const char*);
int main(int, char*[], const Ice::InitializationData&);
int main(int, char*[], const char*, const Ice::LoggerPtr&);
int main(const StringSeq&);
int main(const StringSeq&, const char*);
int main(const StringSeq&, const Ice::InitializationData&);

// 应用初始化后的入口函数由子类实现
virtual int run(int, char*[]) = 0;

// 信号回调函数，默认为空函数
// 如果需要自己对信号进行处理，则需要继承和重写这个函数
// 注意:需在run()函数中调用callbackOnInterrupt()来向设置用户回调模式
virtual void interruptCallback(int);
 
// 返回应用名，即argv[0]
static const char* appName();

// 返回当前使用的Ice::Communicator实例指针
static CommunicatorPtr communicator();

// 设置信号处理模式为销毁模式：收到系统信号则自动销毁Ice::Communicator，是Application默认的模式
static void destroyOnInterrupt();

// 设置信号处理模式为关闭模式：收到系统信号则关闭Ice::Communicator，但不销毁
static void shutdownOnInterrupt();

// 设置信号处理模式为忽略模式：收到系统信号不做任何处理
static void ignoreInterrupt();

// 设置信号处理模式为回调模式：收到系统信号将调用interruptCallback()函数
static void callbackOnInterrupt();

// 挂起信号处理
static void holdInterrupt();
// 恢复信号处理
static void releaseInterrupt();

// 判断当前是否被信号中断
// 可用于判断Application的退出是否由于信号造成的
static bool interrupted();
````

## 使用之前

````c++
#include <Ice/Ice.h>
...
int
main(int argc, char* argv[])
{
    int status = 0;
    Ice::CommunicatorPtr ic;
    try {
        ic = Ice::initialize(argc, argv);
        Ice::ObjectAdapterPtr adapter =
            ic->createObjectAdapterWithEndpoints("SimplePrinterAdapter", "default -p 10000");
        //create servant object and add to adapter
        adapter->activate();
        ic->waitForShutdown();
    } catch (const Ice::Exception& e) {
        cerr << e << endl;
        status = 1;
    } catch (const char* msg) {
        cerr << msg << endl;
        status = 1;
    }
    if (ic) {
        try {
            ic->destroy();
        } catch (const Ice::Exception& e) {
            cerr << e << endl;
            status = 1;
        }
    }
    return status;
}
````
## 使用之后

````C++
#include <Ice/Ice.h>
...
class MyApplication : virtual public Ice::Application {
	virtual int run(int, char *[]) {
		Ice::ObjectAdapterPtr adapter =
            communicator()->createObjectAdapterWithEndpoints("SimplePrinterAdapter", "default -p 10000");
        //create servant object and add to adapter
        adapter->activate();
        communicator()->waitForShutdown();
        return EXIT_SUCCESS;
   }
};
int
main(int argc, char* argv[])
{
    MyApplication app;
    return app.main(argc, argv);
}
````