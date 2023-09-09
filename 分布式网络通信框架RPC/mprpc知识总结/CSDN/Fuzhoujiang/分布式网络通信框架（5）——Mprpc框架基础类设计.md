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

## 相关类的设计

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