![在这里插入图片描述](image/3ee18a18bc83454ab928889d737919fb.png)

## [RPC](https://so.csdn.net/so/search?q=RPC&spm=1001.2101.3001.7020)调用方（caller）的调用(消费)过程

**caller就是按照提供rpc服务的那一方(callee)提供的proto协议，发起调用**  
![在这里插入图片描述](image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBALeael-azveWuhw==,size_20,color_FFFFFF,t_70,g_se,x_16.png)

**我们在caller下创建文件：calluserservice.cc**  
![在这里插入图片描述](image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0xJTlpFWVU2NjY=,size_16,color_FFFFFF,t_70.png)  
**UserServiceRpc\_Stub的构造函数必须要有RpcChannel参数**  
![在这里插入图片描述](image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0xJTlpFWVU2NjY=,size_16,color_FFFFFF,t_70-16935587080001.png)  
**我们知道，里面那些方法最终都是调用channel的CallMethod方法。**  
![在这里插入图片描述](image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0xJTlpFWVU2NjY=,size_16,color_FFFFFF,t_70-16935587080002.png)  
**这个RpcChannel是一个抽象类，new不了对象，我们得在框架上定义一个类从RpcChannel继承而来，然后重写CallMethod方法。**  
![在这里插入图片描述](image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBALeael-azveWuhw==,size_20,color_FFFFFF,t_70,g_se,x_16-16935587080003.png)

**所以，我们现在要在框架上写代码，针对RPC的调用方。**

## 我们在src的include下创建文件：mprpcchannel.h

![在这里插入图片描述](image/d77830195703414faac8a8b888001ed1.png)

```cpp
#pragma once

#include <google/protobuf/service.h>
#include <google/protobuf/descriptor.h>
#include <google/protobuf/message.h>

class MprpcChannel : public google::protobuf::RpcChannel
{
public:
    //所有通过stub代理对象调用的rpc方法，都走到这里了，统一做rpc方法调用的数据数据序列化和网络发送 
    void CallMethod(const google::protobuf::MethodDescriptor* method,
                          google::protobuf::RpcController* controller, 
                          const google::protobuf::Message* request,
                          google::protobuf::Message* response,
                          google::protobuf:: Closure* done);
};
```

![在这里插入图片描述](image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0xJTlpFWVU2NjY=,size_16,color_FFFFFF,t_70-16935587080004.png)

## 我们更新一下example下的CmakeLists.txt

```xml
add_subdirectory(callee)
add_subdirectory(caller)
```

![在这里插入图片描述](image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0xJTlpFWVU2NjY=,size_16,color_FFFFFF,t_70-16935587080005.png)  
**我们在example下的caller下创建CMakeLists.txt**

```xml
set(SRC_LIST calluserservice.cc ../user.pb.cc)
add_executable(consumer ${SRC_LIST})
target_link_libraries(consumer mprpc protobuf)
```

![在这里插入图片描述](image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0xJTlpFWVU2NjY=,size_16,color_FFFFFF,t_70-16935587080006.png)

## 我们在src下创建mprpcchannel.cc

```cpp
#include "mprpcchannel.h"


//所有通过stub代理对象调用的rpc方法，都走到这里了，统一做rpc方法调用的数据数据序列化和网络发送 
void MprpcChannel::CallMethod(const google::protobuf::MethodDescriptor* method,
                                google::protobuf::RpcController* controller, 
                                const google::protobuf::Message* request,
                                google::protobuf::Message* response,
                                google::protobuf:: Closure* done)
{
}
```

## 我们再简单书写一下calluserservice.cc

![在这里插入图片描述](image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0xJTlpFWVU2NjY=,size_16,color_FFFFFF,t_70-16935587080007.png)  
更新一下src的CMakeLists.txt  
![在这里插入图片描述](image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0xJTlpFWVU2NjY=,size_16,color_FFFFFF,t_70-16935587080008.png)  
**编译成功**  
![在这里插入图片描述](image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0xJTlpFWVU2NjY=,size_16,color_FFFFFF,t_70-16935587080009.png)