## 介绍

分发[rpc](https://so.csdn.net/so/search?q=rpc&spm=1001.2101.3001.7020)服务，我们实现的**框架代码**就是主要完成如下红圈部分的任务：

![在这里插入图片描述](image/fa35515352454cae8ebbeb6f1ef6dfab.png#pic_center)

客户端（请求调用方）发送请求时需要发 `service_name`,`method_name`、`args_size`, 其中`args_size`防止粘包问题，于是采用如下的消息格式设计

`header_size`（消息成员长度，占4B） + `header_str`(消息成员)

![在这里插入图片描述](image/c2757f91da3b4edda7f7ac15a8f596c5.png#pic_center)

```protobuf
// rpcheader.proto 
syntax = "proto3";

package mprpc;

// 定义RPC请求头消息格式
message RpcHeader
{
    bytes service_name = 1;  // 请求的服务名
    bytes method_name = 2;  // 请求的方法名
    uint32 args_size = 3;   // 请求的方法的参数大小
}
```

## 分发rpc服务实现代码

分发rpc服务的实现在`RpcProvider::OnMessage`函数中：

```cpp
void RpcProvider::onMessage(const muduo::net::TcpConnectionPtr &conn, muduo::net::Buffer *buffer, muduo::Timestamp)
{
    // 1. 网络上接收远程rpc调用请求的字符流
    std::string recv_buf = buffer->retrieveAllAsString();

    // 2. 从字符流中（recv_buf）读取前4个字节内容
    uint32_t header_size = 0;
    recv_buf.copy((char *)&header_size, 4, 0);

    // 3.根据header_size读取数据头的原始字符流，反序列化数据，得到rpc请求的详细信息
    std::string rpc_header_str = recv_buf.substr(4, header_size);
    mprpc::RpcHeader rpcHeader;
    std::string service_name;
    std::string method_name;
    uint32_t args_size;

    if (rpcHeader.ParseFromString(rpc_header_str))
    {
        service_name = rpcHeader.service_name();
        method_name = rpcHeader.method_name();
        args_size = rpcHeader.args_size();
    }
    else
    {
        std::cout << "rpc_header_str" << rpc_header_str << "parse error" << std::endl;
        return;
    }

    // 获取rpc方法参数的字符流数据
    std::string args_str = recv_buf.substr(4 + header_size, args_size);

    // 打印调试信息
    std::cout << "============================================" << std::endl;
    std::cout << "header_size: " << header_size << std::endl;
    std::cout << "rpc_header_str: " << rpc_header_str << std::endl;
    std::cout << "service_name: " << service_name << std::endl;
    std::cout << "method_name: " << method_name << std::endl;
    std::cout << "args_str: " << args_str << std::endl;
    std::cout << "============================================" << std::endl;

    // 4.获取service对象和method对象
    auto it = m_serviceMap.find(service_name);
    if (it == m_serviceMap.end())
    {
        std::cout << service_name << "is not exist!" << std::endl;
        return;
    }
    // <std::string, const google::protobuf::MethodDescriptor *>::iterator
    auto mit = it->second.m_methodMap.find(method_name);
    if (mit == it->second.m_methodMap.end())
    {
        std::cout << service_name << ":" << method_name << "is not exist" << std::endl;
        return;
    }
// it->second 类型是ServiceInfo
    google::protobuf::Service *service = it->second.m_service; // 获取service对象
    const google::protobuf::MethodDescriptor *method = mit->second;

    // 5.生成rpc方法调用的请求request和响应response参数
    google::protobuf::Message *request = service->GetRequestPrototype(method).New();
    if (!request->ParseFromString(args_str))
    {
        std::cout << "request parse error, content:" << args_str << std::endl;
    }
    google::protobuf::Message *response = service->GetResponsePrototype(method).New();

    // 绑定一个方法调用后的回调
    google::protobuf::Closure *done = google::protobuf::NewCallback<RpcProvider, const muduo::net::TcpConnectionPtr &, google::protobuf::Message *>(this,
                                                                                                                                                    &RpcProvider::SendRpcResponse,
                                                                                                                                                    conn,
                                                                                                                                                    response);

    // 6.在框架上根据远端rpc请求， 调用当前rpc节点上发布的方法
    // done是执行完本地节点提供的方法后的回调，一般是把结果序列化发送回对端
    service->CallMethod(method, nullptr, request, response, done);
}
```