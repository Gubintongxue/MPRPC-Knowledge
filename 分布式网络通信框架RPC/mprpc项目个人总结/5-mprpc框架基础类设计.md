## 需求

我们希望我们实现的`mprpc`可以像下面这样被使用：

```cpp
int main(int argc, char **argv)
{
    // 调用框架的初始化操作
    MprpcApplication::Init(argc, argv);

    // provider是一个rpc网络服务对象，把UserService对象发布到rpc节点上
    RpcProvider provider;
    provider.NotifyService(new UserService());

    // 启动一个rpc服务发布节点，Run之后，进程进入阻塞状态，等待远程rpc调用请求
    provider.Run();
}

```

说明：

1.参数个数argc 参数字符串argv 相当于是读取配置，一般就是ip地址和端口

框架的初始化，一般就是读取配置，初始化日志模块

2.希望框架提供一个对象provider，可以专门让我发布服务

通过NotifyService方法，将服务对象发布到rpc节点上

3.还应该具有网络功能

provider.run()启动一个网络程序，进行进入阻塞，等待远程的rpc调用



总的

```C++
#include <iostream>
#include <string>
#include "user.pb.h"
#include "mprpcapplication.h"
#include "rpcprovider.h"

/*
UserService原来是一个本地服务，提供了两个进程内的本地方法，Login和GetFriendLists
*/
class UserService : public fixbug::UserServiceRpc // 使用在rpc服务发布端（rpc服务提供者）
{
public:
    bool Login(std::string name, std::string pwd)
    {
        std::cout << "doing local service: Login" << std::endl;
        std::cout << "name:" << name << " pwd:" << pwd << std::endl;  
        return false;
    }

    /*
    重写基类UserServiceRpc的虚函数 下面这些方法都是框架直接调用的
    1. caller   ===>   Login(LoginRequest)  => muduo =>   callee 
    2. callee   ===>    Login(LoginRequest)  => 交到下面重写的这个Login方法上了
    */
    void Login(::google::protobuf::RpcController* controller,
                       const ::fixbug::LoginRequest* request,
                       ::fixbug::LoginResponse* response,
                       ::google::protobuf::Closure* done)
    {
        // 框架给业务上报了请求参数LoginRequest，应用获取相应数据做本地业务
        std::string name = request->name();
        std::string pwd = request->pwd();

        // 做本地业务
        bool login_result = Login(name, pwd); 

        // 把响应写入  包括错误码、错误消息、返回值
        fixbug::ResultCode *code = response->mutable_result();
        code->set_errcode(0);
        code->set_errmsg("");
        response->set_success(login_result);

        // 执行回调操作   执行响应对象数据的序列化和网络发送（都是由框架来完成的）
        done->Run();
    }
};


int main(int argc, char **argv)
{
    // 调用框架的初始化操作 provider -i config.conf
    MprpcApplication::Init(argc, argv);

    // provider是一个rpc网络服务对象。把UserService对象发布到rpc节点上
    RpcProvider provider;
    provider.NotifyService(new UserService());

    // 启动一个rpc服务发布节点   Run以后，进程进入阻塞状态，等待远程的rpc调用请求
    provider.Run();

    return 0;
}
```



### 之前我们说过了服务方法的设计

就是套路：

1.定义proto文件中的方法，然后继承并且重写这个方法

2.获取框架给业务上报的请求参数

3.做本地业务（自己定义好）

4.写好响应

5.执行回调，序列化和网络发送（框架来完成）

![image-20230903223630990](image/image-20230903223630990.png)



小结：可以将provider实际上是一个网络服务对象，给我们提供发布rpc服务的方法

run()之后就会等待rpc调用请求，

然后执行请求，就是框架给我们上报到Login()方法这里，做本地业务，然后将结果返回done->run()



# 框架的书写

### 前置：

完善CMakeLists.txt，将之前注释的src取消掉

```shell
# 设置cmake的最低版本和项目名称
cmake_minimum_required(VERSION 3.0)
project(mprpc)

# 生成debug版本，可以进行gdb调试
set(CMAKE_BUILD_TYPE "Debug")

# 设置项目可执行文件输出的路径
set(EXECUTABLE_OUTPUT_PATH ${PROJECT_SOURCE_DIR}/bin)
# 设置项目库文件输出的路径
set(LIBRARY_OUTPUT_PATH ${PROJECT_SOURCE_DIR}/lib)

# 设置项目编译头文件搜索路径 -I
include_directories(${PROJECT_SOURCE_DIR}/src/include)
include_directories(${PROJECT_SOURCE_DIR}/example)
# 设置项目库文件搜索路径 -L
link_directories(${PROJECT_SOURCE_DIR}/lib)

# src包含了mprpc框架所有的相关代码
add_subdirectory(src)
# example包含了mprpc框架使用的示例代码
add_subdirectory(example)
```

在src文件夹下新建include文件夹，放置头文件





## 相关类的设计

在include下新建mprpcapplication.h,rpcprovider.h如下

根据上面的需求，我们可以设计两个类`MprpcApplication` 和 `RpcProvider`

```cpp
// mprpcapplication.h
// singleton
class MprpcApplication
{
public:
    static void Init(int argc, char **argv);
    static MprpcApplication& getInstance();
private:
    MprpcApplication(){}
    MprpcApplication(const MprpcApplication&) = delete;
    MprpcApplication(MprpcApplication&&) = delete;
};


```



```C++
// rpcprovider.h
#pragma once
#include "google/protobuf/service.h"

class RpcProvider
{
public:
    // 这里参数是Service的基类，proto中定义的所有service都是其派生类
    // 框架提供给用户使用，可以发布rpc方法的接口函数
    void NotifyService(google::protobuf::Service *service);

    // 启动rpc服务节点，开始提供rpc远程网络调用服务
    void Run();
};
```



