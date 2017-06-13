#Slice语言
Slice (Specification Language for Ice)是Ice的接口定义语言，用于分离接口定义和实现，它有自己独立的语法规则（类似于Java/C++语言），是连接Ice应用客户端和服务端的协议描述，通过slice转换器能够转换映射成C++, Java, C#, Python, Objective-C, Ruby和PHP语言接口定义和实现。

简单C++构建应用客户端和服务端部署映射图
![](https://doc.zeroc.com/download/attachments/16715996/Client_server_same_development_environment.gif?version=1&modificationDate=1432836963000&api=v2)

C++/Java混合应用客户端和服务端部署映射图
![](https://doc.zeroc.com/download/attachments/16715996/Development_process_different_development_environment.gif?version=1&modificationDate=1432836970000&api=v2)

# Slice基本约束

- 文件扩展名为.ice,如Printer.ice
- 支持类似C++的预处理宏

````
// File Clock.ice
#ifndef _CLOCK_ICE
#define _CLOCK_ICE
// 预处理宏必须放在文件头
// #include directives here...
// Definitions here...
// 
#endif _CLOCK_ICE
````
这个预处理宏是为了防止头文件重复引用，当然还可以用#pragma指令更简单达到同样的结果

````
// File Clock.ice
#pragma once
//
// #include directives here...
// Definitions here...
````
- 支持#include指令引用其他定义文件,官方推荐使用(<>)代替双引号
- 支持预定义宏来检测Ice版本

````
#if defined(__ICE_VERSION__) && __ICE_VERSION__ >= 030500
enum Fruit { Apple, Pear = 3, Orange };
#else
enum Fruit { Apple, Pear, Orange };
#endif
````
- 支持预定义宏检测Slice编译器

````
//__SLICE2JAVA__
//__SLICE2JS__
//__SLICE2CPP__
//__SLICE2CS__
//__SLICE2PY__
//__SLICE2PHP__
//__SLICE2RB__
//__SLICE2FREEZE__
//__SLICE2FREEZEJ__
//__SLICE2HTML__
//__TRANSFORMDB__
//__DUMPDB__
````
- 支持类C/C++风格的注释
- 除了Object/LocalObject两个关键字是大写以外所有关键字都小写
- 命名大小写不敏感，官方建议采用驼峰风格命名如TimeOfDay
- 所有Ice开头的命名都保留
- 所有的定义都应该在Module内，Module映射成C++的命名空间和Java包名(Java可以添加包名注释)

````
[["java:package:com.demo"]]
module Demo {
    module Client {
        // Definitions here...
    };
    module Server {
        // Definitions here...
    };
};
````

#基本类型

| Type | Range of Mapped Type | Size of Mapped Type |
| ---- | -------- | -------- |
| bool | false or true | ≥ 1bit
| byte | -128-127 or 0-255 | ≥ 8 bits
| short | -2^15 to 2^15 -1 | ≥ 16 bits
| int | -2^31 to 2^31 -1 | ≥ 32 bits
| long | -2^63 to 2^63 -1 | ≥ 64 bits
| float | IEEE single-precision | ≥ 32 bits
| double | IEEE double-precision | ≥ 64 bits
| string | All Unicode characters, excluding the character with all bits zero. |Variable-length

#复合数据类型
- Enumerations

````
module M {
    enum Color { red, green, blue };
};
module N {
    struct Pixel {
        M::Color c = M::blue;
    };
};
````
- Structures

````
struct Point {
    short x = 0; //支持默认值
    short y;
};
 
struct TwoPoints {
    Point coord1; //不支持struct嵌套定义，struct支持引用
    Point coord2;
};
````
- Sequences

````
sequence<long> SerialOpt; //Array集合，支持基本类型和复合类型
 
struct Part {
    string    name;
    string    description;
    // ...
    SerialOpt serialNumber; // optional: zero or one element
};
````
- Dictionaries

````
struct Employee {
    long   number;
    string firstName;
    string lastName;
};
 
dictionary<long, Employee> EmployeeMap; //Map集合
````

#常量
````
const bool      AppendByDefault = true;
const byte      LowerNibble = 0x0f;
const string    Advice = "Don't Panic!";
const short     TheAnswer = 42;
const double    PI = 3.1416;
 
enum Fruit { Apple, Pear, Orange };
const Fruit     FavoriteFruit = Pear;
````
默认不支持下划线命名，如果需要使用下划线需要加--underscore参数

#接口定义
````
struct TimeOfDay {
    short hour;         // 0 - 23
    short minute;       // 0 - 59
    short second;       // 0 - 59
};
 
interface Clock {
    idempotent TimeOfDay getTime();
    void setTime(TimeOfDay time);
};
````
idempotent关键字，表示该方法是幂等的，即调用1次和调用2次其结果是一样的,比如常见的查询操作基本上都是幂等的，而update和create等方法则不是,如果方法是幂等的则增加idempotent修饰后，可以让Ice更好的实现“自动恢复的机制”，即在某个Ice Object调用失败的情况下，无法区分是否已经调用过，但是在因为网络错误导致没有正确返回结果的情况下，Ice会再次调用有idempotent修饰的方法，透明恢复故障

- 不支持函数重载

````

interface CircadianRhythm {
    void modify(TimeOfDay startTime, TimeOfDay endTime);
    void modify(    TimeOfDay startTime,        // Error
                    TimeOfDay endTime,
                out timeOfDay prevStartTime,
                out TimeOfDay prevEndTime);
};
````

- 支持返回值返回数据和out参数返回数据

````
//out参数必须放在末尾
interface CircadianRhythm {
    void setSleepPeriod(TimeOfDay startTime, TimeOfDay stopTime);
    void getSleepPeriod(out TimeOfDay startTime, out TimeOfDay stopTime);
    // ...
};
````

- Optional参数和返回值

````
//每个optional参数必须附加一个正整数tag值，且方法内唯一，这个tag值在Ice请求打包和解包是用到，应用不会用到
optional(3) bool example(optional(2) string name, out optional(1) int value);
````

- 用户异常

````
exception Error {
    string reason;
};
 
exception FatalApplicationError extends Error {
    // 支持异常继承
};
 
interface Application {
    void doSomething() throws Error;
};
````

Ice异常分为运行时异常和用户异常，运行时异常都是LocalException，而用户定义的异常都是UserException，完整异常继承树如下
![](https://doc.zeroc.com/download/attachments/16716010/Ice_runtime_exception_hierarchy.gif?version=1&modificationDate=1469736186000&api=v2)

- 支持Proxies对象

````
interface Link {
    idempotent SomeType getValue();
    idempotent Link* next();
};
````
`Link*`并不是C/C++中的指针，而是Ice的Proxies对象，即引用的是服务端对象，`Link*`只是服务端对象的代理对象

- 接口支持继承

````
interface B { /* ... */ };
interface I1 extends B { /* ... */ };
interface I2 extends B { /* ... */ };
interface D extends I1, I2 { /* ... */ };
````

- 接口继承父类方法不能重名

````
interface Clock {
    void set(TimeOfDay time);                   // set time
};
 
interface Radio {
    void set(long hertz);                       // set frequency
};
 
interface RadioClock extends Radio, Clock {     // Illegal!
    // ...
};
````

#类
类是struct和interface的混合体，既能想struct一样传值又能像interface一样拥有方法和传引用，slice类的定于与C++/Java的类定义类似.

````
class TimeOfDay {
    short hour;         // 0 - 23
    short minute;       // 0 - 59
    short second;       // 0 - 59
};
````

- 支持空类

````
class EmptyClass {};    // OK
struct EmptyStruct {};  // Error
````

- 支持继承，但不支持多继承，不允许出现重名成员

````
class TimeOfDay {
    short hour;         // 0 - 23
    short minute;       // 0 - 59
    short second;       // 0 - 59
};
 
class DateTime extends TimeOfDay {
    short day;          // 1 - 31
    short month;        // 1 - 12
    short year;         // 1753 onwards
};

//实现多态
class Shape {
    // Definitions for shapes, such as size, center, etc.
};
 
class Circle extends Shape {
    // Definitions for circles, such as radius...
};
 
class Rectangle extends Shape {
    // Definitions for rectangles, such as width and length...
};
 
sequence<Shape> ShapeSeq;
 
interface ShapeProcessor {
    void processShapes(ShapeSeq ss);
};

````

- 支持自引用,实现对象链

````
class Link {
    SomeType value;
    Link next;//语义与Link*完全不同，Link*的地址空间在远程
};
````

- 避免在class中定义接口，或者class实现接口类

````
interface Time {
    idempotent TimeOfDay getTime();
    idempotent void setTime(TimeOfDay time);
};
 
class Clock implements Time {
    TimeOfDay time;
};
````
class的方法都是本地实现，通过本地对象工厂创建具体的实现对象，这种方式主要为满足特殊性能要求，所以尽量避免使用

#metadata指令
方括号包含一个或多个字符串指令，可以修饰slice类型，metadata指令不会影响客户端与服务端的交互联系，本身不影响slice类型系统，只是修改目标代码映射，例如使用LinkedList代替默认的Array

````
["java:type:java.util.LinkedList<Integer>"] sequence<int> IntSeq;
````

全局的metadata指令采用双方括号定义，例如指定java包名

````
[["java:package:com.acme"]]
````

>详细的metadata指令请参考官方文档 
>https://doc.zeroc.com/display/Ice36/Slice+Metadata+Directives

