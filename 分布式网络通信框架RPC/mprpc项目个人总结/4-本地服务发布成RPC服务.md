# 前置知识

我们做的是框架，但是也要是业务去驱动框架的开发

并且按照之前集群聊天服务器的框架来写

example底下就是业务的代码

src下就是框架的源码

lib就是生成的库文件

build就是cmake编译产生的中间文件

bin里放的是服务的提供者和消费者的可执行文件



先把cmakelist.txt写出来

仿照集群聊天服务器写

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

# src包含了mprpc框架所有的相关代码   目前还用不到
#add_subdirectory(src)
# example包含了mprpc框架使用的示例代码
add_subdirectory(example)
```

autobuild.sh编写

```shell
#!/bin/bash

#这一行设置了 Bash 的选项 -e，它的作用是在脚本中任何一个命令执行失败时就立即退出脚本。这有助于在出现错误时及早停止脚本执行。
set -e
#这一行删除 build 目录下的所有文件和子目录。pwd 命令用于获取当前工作目录的路径。
rm -rf `pwd`/build/*
#cd pwd/build: 这一行切换到 build 目录，通常用于进行编译。
#这一行使用 CMake 工具来生成项目的构建系统。.. 表示返回到上一级目录，通常在 build 目录中运行以配置项目。
#这一行用于编译项目，使用 make 命令来执行 Makefile 文件中的编译规则。
cd `pwd`/build &&
	cmake .. &&
	make

#这一行将当前工作目录切换回脚本所在的目录，通常用于完成编译后的清理和拷贝操作。
cd ..
#这一行将源代码目录中的 include 目录复制到 lib解释 目录中。 -r 选项表示递归复制，将目录和子目录都复制过去。
cp -r `pwd`/src/include `pwd`/lib
```



## 简介

通过这个例子，目的在于了解[RPC框架](https://so.csdn.net/so/search?q=RPC%E6%A1%86%E6%9E%B6&spm=1001.2101.3001.7020)在**服务提供方**（`server`/`callee`）的使用,从而更好理解实现代码  
本项目的完整源代码在 https://github.com/StrikeCode/mprpc.git

## 流程

在example中使我们业务代码，我们创建服务的提供方callee和调用放caller



### 我们需要提供好调用方和提供方的请求和响应的格式

首先是在 `mprpc/example/user.proto` 指定对应的rpc方法和相关的返回值和参数类型，生成对应的c++文件

==我们这里只是先做测试，所以proto内容较少==，只做业务上的登录请求和响应

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

//定义rpc方法
service UserServiceRpc
{
    rpc Login(LoginRequest) returns(LoginResponse);
}
```



在example文件夹下新建CMakeLists.txt用于编译

因为顶层CMakeLists.txt就已经写好了搜索路径，包括example

```
add_subdirectory(callee)
add_subdirectory(caller)
```

继续搜索路径

在callee和caller下新建CMakeLists.txt

callee

```
#通过userservice和上一级的user.pb.cc生成
set(SRC_LIST userservice.cc ../user.pb.cc)

add_executable(provider ${SRC_LIST})
#target_link_libraries(provider mprpc protobuf)
```



calleer（暂时不用写）

```

```



其次在`mprpc/example/callee/userservice.cc` 实现业务代码，并重写框架提供的虚函数



理解为什么这样写 可以查看 12 本地服务怎么发布成rpc服务 笔记

```cpp
#include <iostream>
#include <string>
#include "user.pb.h"

// UserService 是一个本地服务，提供了两个进程内的本地方法Login和GetFriendLists
class UserService : fixbug::UserServiceRpc // 使用在rpc服务的发布端（rpc服务提供者）
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



编译

```
cd buildd
cmake ..
```

