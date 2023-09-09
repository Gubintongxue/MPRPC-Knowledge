## 完成 MprpcApplication::Init(argc, argv)；

**我们打开userservice.cc中，查看主函数**

```cpp
int main(int argc, char **argv)//需要配置文件
{
    //调用框架的初始化操作
    MprpcApplication::Init(argc, argv);//整个框架的初始化操作，日志，配置等等。

    //provider是一个rpc网络的服务对象。把UserService对象发布到rpc节点上。
    RpcProvider provider;
    provider.NotifyService(new UserService());//发布服务
//可以调用多次，生成多个远程RPC服务

    //启动一个rpc服务发布节点
    provider.Run();
//Run以后，进程进入阻塞状态，等待远程的rpc调用请求

    return 0;
}
```

**这个Init函数需要用户传一个命令行参数  
我们希望这么去写 provider -i config.conf**  
**读取相关的网络服务器以及相关的配置中心的IP地址和端口号。**

**我们打开mprpcapplication.cc**  
![在这里插入图片描述](image/8407f34e18d84694b798abe0ca5340fa.png)

```cpp
#include "mprpcapplication.h"
#include <iostream>
#include <unistd.h>//getopt的头文件
#include <string>

MprpcConfig MprpcApplication::m_config;

void ShowArgsHelp()
{
    std::cout<<"format: command -i <configfile>" << std::endl;//格式必须是command -i <configfile>
}

void MprpcApplication::Init(int argc, char **argv)
{
    if (argc < 2)//说明程序RPC服务站点根本没有传入任何参数
    {
        ShowArgsHelp();
        exit(EXIT_FAILURE);//退出
    }

    int c = 0;
    std::string config_file;
    while((c = getopt(argc, argv, "i:")) != -1)//我们需要-i参数，而且是必须有，所以加上：
    {
        switch (c)
        {
        case 'i'://有正确的配置文件
            config_file = optarg;//optarg是全局变量，参数值都在这里面
            break;
        case '?'://出现了不希望出现的参数，我们指定必须出现i
            ShowArgsHelp();
            exit(EXIT_FAILURE);
        case ':'://出现了-i,但是没有参数
            ShowArgsHelp();
            exit(EXIT_FAILURE);
        default:
            break;
        }
    }

    //开始加载配置文件了 rpcserver_ip=  rpcserver_port=   zookeeper_ip=  zookepper_port=
    m_config.LoadConfigFile(config_file.c_str());

    //std::cout << "rpcserverip:" << m_config.Load("rpcserverip") << std::endl;
    //std::cout << "rpcserverport:" << m_config.Load("rpcserverport") << std::endl;
    //std::cout << "zookeeperip:" << m_config.Load("zookeeperip") << std::endl;
    //std::cout << "zookeeperport:" << m_config.Load("zookeeperport") << std::endl;
}

MprpcApplication& MprpcApplication::GetInstance()//单例模式
{
    static MprpcApplication app;
    return app;
}

MprpcConfig& MprpcApplication::GetConfig()
{
    return m_config;
}
```

## 我们在将下一节阐述mprpc[框架](https://so.csdn.net/so/search?q=%E6%A1%86%E6%9E%B6&spm=1001.2101.3001.7020)的配置文件的加载