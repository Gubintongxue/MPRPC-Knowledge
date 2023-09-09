## 我们先完善好calluserservice.cc(RPC消费者）

```cpp
#include <iostream>
#include "mprpcapplication.h"
#include "user.pb.h"
#include "mprpcchannel.h"

int main(int argc, char **argv)
{
    //整个程序启动以后，想使用mprpc框架来享受rpc服务调用，一定需要先调用框架的初始化函数（只初始化一次）
    MprpcApplication::Init(argc, argv);

    //演示调用远程发布的rpc方法Login
    fixbug::UserServiceRpc_Stub stub(new MprpcChannel());//protobuf service生成的两个类，其中stub类作为是消费者（rpc调用者）
    //rpc方法的请求参数
    fixbug::LoginRequest request;//登录请求
    request.set_name("linzeyu");
    request.set_pwd("123456");
    //rpc方法的响应
    fixbug::LoginResponse response;
    //发起rpc方法的调用 ， 这是同步的rpc调用过程  MprpcChannel::callmethod，数据序列化然后发送，等待远程的rpc给我响应这个执行的方法结果
    stub.Login(nullptr, &request, &response, nullptr);//RpcChannel->RpcChannel::callMethod 集中来做所有rpc方法调用的参数序列化和网络发送

    //程序跑到这，说明一次rpc调用完成，读调用的结果
    if (0 == response.result().errcode())
    {
        std::cout << "rpc login response success:" << response.sucess() << std::endl;
    }
    else
    {
        std::cout << "rpc login response error : " << response.result().errmsg() << std::endl;
    }
    return 0;
}
```

## 我们看一下调用示意图

![在这里插入图片描述](image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0xJTlpFWVU2NjY=,size_16,color_FFFFFF,t_70.png)

## 完善mprpcchannel.cc

```cpp
#include "mprpcchannel.h"
#include <string>
#include "rpcheader.pb.h"
#include <sys/types.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <netinet/in.h>
#include <unistd.h>
#include <errno.h>
#include "mprpcapplication.h"
#include <iostream>

/*
消息的格式是：
header_size + service_name method_name args_size + args
*/
// 所有通过stub代理对象调用的rpc方法，都走到这里了，统一做rpc方法调用的数据数据序列化和网络发送 
void MprpcChannel::CallMethod(const google::protobuf::MethodDescriptor* method,
                                google::protobuf::RpcController* controller, 
                                const google::protobuf::Message* request,
                                google::protobuf::Message* response,
                                google::protobuf:: Closure* done)
{
    const google::protobuf::ServiceDescriptor* sd = method->service();//知道这个方法属于哪个服务对象的
    std::string service_name = sd->name();//service_name
    std::string method_name = method->name();//method_name

    //获取参数的序列化字符串长度args_size
    uint32_t args_size = 0;
    std::string args_str;
    if (request->SerializeToString(&args_str))//序列化成功
    {
        args_size = args_str.size();
    }
    else//序列化失败
    {
        std::cout<<"serialize request error!"<<std::endl;
        return;
    }
    
    //定义rpc的请求header
    mprpc::RpcHeader rpcHeader;
    rpcHeader.set_service_name(service_name);
    rpcHeader.set_method_name(method_name);
    rpcHeader.set_args_size(args_size);

    uint32_t header_size = 0;
    std::string rpc_header_str;
    if (rpcHeader.SerializeToString(&rpc_header_str))//序列化成功
    {
        header_size = rpc_header_str.size();
    }
    else//序列化失败
    {
        controller->SetFailed("serialize rpc header error!");
        return;
    }

    //组织待发送的rpc请求的字符串
    std::string send_rpc_str;
    send_rpc_str.insert(0, std::string((char*)&header_size, 4));//header_size，从开头开始，写4个字节，二进制存储head_size，就是一个整数
    send_rpc_str += rpc_header_str;//rpcheader
    send_rpc_str += args_str;//args
//string类的好处

    //打印调试信息
    std::cout << "============================================" << std::endl;
    std::cout << "header_size: " << header_size << std::endl; 
    std::cout << "rpc_header_str: " << rpc_header_str << std::endl; 
    std::cout << "service_name: " << service_name << std::endl; 
    std::cout << "method_name: " << method_name << std::endl; 
    std::cout << "args_str: " << args_str << std::endl; 
    std::cout << "============================================" << std::endl;

    //因为是客户端，消费者，不需要高并发，只需要使用tcp编程，完成rpc方法的远程调用
    int clientfd = socket(AF_INET, SOCK_STREAM, 0);//连接套接字
    if (-1 == clientfd)
    {
        std::cout<<"create socket error! errno:"<<errno<<std::endl; 
        return;
    }
//读取配置文件rpcserver的信息
    std::string ip = MprpcApplication::GetInstance().GetConfig().Load("rpcserverip");
    uint16_t port = atoi(MprpcApplication::GetInstance().GetConfig().Load("rpcserverport").c_str());

    struct sockaddr_in server_addr;
    server_addr.sin_family = AF_INET;
    server_addr.sin_port = htons(port);
    server_addr.sin_addr.s_addr = inet_addr(ip.c_str());

    //发起连接。连接rpc服务节点
    if (-1 == connect(clientfd, (struct sockaddr*)&server_addr, sizeof(server_addr)))
    {
        close(clientfd);//关闭连接描述符
        std::cout<<"connect error! errno:"<<errno<<std::endl;
        return;
    }

    //连接成功后，发送rpc请求
    if (-1 == send(clientfd, send_rpc_str.c_str(), send_rpc_str.size(), 0))
    {
        close(clientfd);//关闭连接描述符
std::cout<<"send error! errno:"<<errno<<std::endl;
        return;
    }

    // 接收rpc请求的响应值
    char recv_buf[1024] = {0};
    int recv_size = 0;
    if (-1 == (recv_size = recv(clientfd, recv_buf, 1024, 0)))
    {
        close(clientfd);
        std::cout<<"recv error! errno:"<<errno<<std::endl;
        return;
    }

    // 反序列化rpc调用的响应数据
    // std::string response_str(recv_buf, 0, recv_size);//有bug,会出现问题，recv_buf中遇到\0后面的数据就存不下来了，导致反序列化失败
    // if (!response->ParseFromString(response_str))
    if (!response->ParseFromArray(recv_buf, recv_size))//反序列化，填到response里面
    {
        close(clientfd);
        std::cout<<"parse error! response_str:"<<recv_buf<<std::endl;
        return;
    }

    close(clientfd);
}
```

**编译成功**  
![在这里插入图片描述](image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0xJTlpFWVU2NjY=,size_16,color_FFFFFF,t_70-16935588346611.png)