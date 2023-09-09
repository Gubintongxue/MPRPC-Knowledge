因为RPC服务器必然是要接受大量客户端的调用请求，所以需要一个**较高的并发量**，本项目采用`muduo`网络库，其核心是`one loop per thread`,相关实现代码如下：

```cpp
// rpcprovider.h
#pragma once
#include "google/protobuf/service.h"
#include <memory>
#include <muduo/net/TcpServer.h>
#include <muduo/net/EventLoop.h>
#include <muduo/net/InetAddress.h>
#include <muduo/net/TcpConnection.h>


class RpcProvider
{
public:
    // 这里参数是Service的基类，proto中定义的所有service都是其派生类
    // 框架提供给用户使用，可以发布rpc方法的接口函数
    void NotifyService(google::protobuf::Service *service);

    // 启动rpc服务节点，开始提供rpc远程网络调用服务
    void Run();

private:
    // 组合EventLoop
    muduo::net::EventLoop m_eventLoop;

    // 新socket链接回调
    void onConnection(const muduo::net::TcpConnectionPtr &conn);
    // 已建立链接用户的读写事件回调
    void onMessage(const muduo::net::TcpConnectionPtr&, muduo::net::Buffer*, muduo::Timestamp);
};

// rpcprovider.cc
#include "rpcprovider.h"
#include "mprpcapplication.h"
#include <string>
#include <functional>


void RpcProvider::NotifyService(google::protobuf::Service *service)
{

}

// 启动rpc服务节点，开赌场提供rpc远程网络调用服务
void RpcProvider::Run()
{
    std::string ip = MprpcApplication::getInstance().getConfig().Load("rpcserverip");
    uint16_t port = atoi(MprpcApplication::getInstance().getConfig().Load("rpcserverport").c_str());
    muduo::net::InetAddress address(ip, port);
    // 创建TcpServer对象
    muduo::net::TcpServer server(&m_eventLoop, address, "RpcProvider");
    // 绑定连接回调和消息读写回调方法
    server.setConnectionCallback(std::bind(&RpcProvider::onConnection, this, std::placeholders::_1));
    server.setMessageCallback(std::bind(&RpcProvider::onMessage, this, std::placeholders::_1, std::placeholders::_2, std::placeholders::_3));
    // 设置muduo库线程数量
    server.setThreadNum(4);
    server.start();
    std::cout << "RpcProvider start service at ip:" << ip << " port:" << port << std::endl; 
    m_eventLoop.loop(); // epoll_wait()
    
}

// 新socket链接回调
void RpcProvider::onConnection(const muduo::net::TcpConnectionPtr &conn)
{
    
}
void RpcProvider::onMessage(const muduo::net::TcpConnectionPtr&, muduo::net::Buffer*, muduo::Timestamp)
{

}
```