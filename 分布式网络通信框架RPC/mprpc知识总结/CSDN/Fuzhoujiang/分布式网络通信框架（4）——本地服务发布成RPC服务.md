## 简介

通过这个例子，目的在于了解[RPC框架](https://so.csdn.net/so/search?q=RPC%E6%A1%86%E6%9E%B6&spm=1001.2101.3001.7020)在**服务提供方**（`server`/`callee`）的使用,从而更好理解实现代码  
本项目的完整源代码在 https://github.com/StrikeCode/mprpc.git

## 流程

首先是在 `mprpc/example/user.proto` 指定对应的rpc方法和相关的返回值和参数类型，生成对应的c++文件

```protobuf
syntax = "proto3";

package fixbug;

option cc_generic_services = true;

message ResultCode
{
    int32 errcode = 1;
    bytes errmsg = 2;
}

// 定义登录消息类型
message LoginRequest
{
    bytes name = 1; // =1 代表name是这个message第一个字段，不是指name的值
    bytes pwd = 2;
}

// 定义登录响应消息
message LoginResponse
{
    ResultCode result = 1;
    bool success = 2;
}

service UserServiceRpc
{
    rpc Login(LoginRequest) returns(LoginResponse);
}
```

其次在`mprpc/example/callee/userservice.cc` 实现业务代码，并重写框架提供的虚函数

```cpp
#include <iostream>
#include <string>
#include "user.pb.h"

// UserService 是一个本地服务，提供了两个进程内的本地方法Login和GetFriendLists
class UserService : public fixbug::UserServiceRpc // 使用在rpc服务的发布端（rpc服务提供者）
{
public:
    bool Login(std::string name, std::string pwd)
    {
        std::cout << "doing local service:Login" << std::endl;
        std::cout << "name:" << name << " pwd:"<< pwd << std::endl;
    }
    // 这是在使用角度分析RPC的功能
    // 重写基类 UserServiceRpc的虚函数
    // 1. caller（client） ==> Login(LoginRequest) => muduo => callee(server)
    // 2. callee(server) ==> Login(LoginRequest) => 转发到重写的Login方法上（如下）
    void Login(::google::protobuf::RpcController* controller,
                       const ::fixbug::LoginRequest* request,
                       ::fixbug::LoginResponse* response,
                       ::google::protobuf::Closure* done)
    {
        // 1.框架给业务上报了请求参数LoginRequest，应用获取相应数据做本地业务
        std::string name = request->name();
        std::string pwd = request->pwd();

        // 2.做业务
        bool login_result = Login(name, pwd);

        // 3.把响应写入
        fixbug::ResultCode *code = response->mutable_result();
        code->set_errcode(0);
        code->set_errmsg("");
        response->set_success(login_result);

        // 4.做回调操作,执行响应对象数据的序列化和网络发送（都是由框架完成）
        done->Run();
    }
};
```
