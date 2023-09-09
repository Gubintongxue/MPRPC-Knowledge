## 介绍

`RpcProvider`，利用`muduo`库的来实现数据的收发，利用`protobuf`来实现数据的序列化（发送前）和[反序列化](https://so.csdn.net/so/search?q=%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96&spm=1001.2101.3001.7020)（接收后）

发布服务主要在`NotifyService` 实现，重点在于**生成一张表，记录服务对象和其发布的所有服务方法**

```cpp
struct ServiceInfo
{
    google::protobuf::Service *m_service; // 保存服务对象的地址
    std::unordered_map<std::string, const google::protobuf::MethodDescriptor*> m_methodMap; // 保存服务方法映射<方法名, 方法描述符指针>
};
 // 存储注册成功的 服务对象 和 其服务的所有信息
std::unordered_map<std::string, ServiceInfo> m_serviceMap;
```

## NotifyService 函数实现代码

```cpp
void RpcProvider::NotifyService(google::protobuf::Service *service)
{
    ServiceInfo service_info;
    // 获取服务对象的描述信息
    const google::protobuf::ServiceDescriptor *pserviceDesc = service->GetDescriptor();
    // 获取服务的名字
    std::string service_name = pserviceDesc->name();
    // 获取服务对象service的方法数量
    int methodCnt = pserviceDesc->method_count();
    std::cout << "service_name:" << service_name << std::endl;

    for(int i = 0; i < methodCnt; ++i)
    {
        // 获取了服务对象指定下标的服务方法的描述符指针
        const google::protobuf::MethodDescriptor *pmethodDesc = pserviceDesc->method(i);
        std::string method_name = pmethodDesc->name();
        // 将方法映射关系添加到对应服务对象的方法表中
        service_info.m_methodMap.insert({method_name, pmethodDesc});
        std::cout << "method_name:" << method_name << std::endl;
    }

    service_info.m_service = service;
    m_serviceMap.insert({service_name, service_info});
}
```