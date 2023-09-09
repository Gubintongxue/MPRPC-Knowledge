## 例子1

test.[proto](https://so.csdn.net/so/search?q=proto&spm=1001.2101.3001.7020)文件如下：

```protobuf
syntax = "proto3"; // 声明了protobuf版本

package fixbug; // 声明了代码所在的包（生成C++代码后就是namespace 名字）

// 定义登录消息类型
message LoginRequest
{
    string name = 1; // =1 代表name是这个message第一个字段，不是指name的值
    string pwd = 2;
}

// 定义登录响应消息
message LoginResponse
{
    int32 errcode = 1;
    string errmsg = 2;
    bool success = 3;
}

```

对proto文件进行编译：

```shell
protoc test.proto --cpp_out=./
```

产生两个cpp文件

![在这里插入图片描述](image/b756f777ccfc4c98839a615b50752a9c.png#pic_center)

测试代码main.cc如下：

```cpp
#include "test.pb.h"
#include <iostream>
#include <string>
using namespace fixbug; // 工作中不要这样！！防止命名空间污染

int main()
{
    // 封装login请求对象的数据
    LoginRequest req;
    req.set_name("lisi");
    req.set_pwd("123456");

    std::string send_str;
    // 将成员值序列化为字符串
    if(req.SerializeToString(&send_str))
    {
        std::cout << send_str.c_str() << std::endl;
    }

    // 从send_str反序列化一个login请求对象
    LoginRequest reqB;
    if(reqB.ParseFromString(send_str))
    {
        std::cout << reqB.name() << std::endl;
        std::cout << reqB.pwd() << std::endl;
    }

    return 0;
}
```

```shell
jyhlinux@ubuntu:~/share/mprpc/test/protobuf$ g++ main.cc test.pb.cc -o main -lprotobuf -L/usr/local/lib
jyhlinux@ubuntu:~/share/mprpc/test/protobuf$ ./main
```

实验结果

lisi123456

lisi

123456

### 注意

建议将`string`类型替换为`bytes`类型，因为`bytes`直接存二进制文件，效率更高一点，如果用`string`，最后还要将其转换为字节数据，而`bytes`则不需要。最后结果和上面相同

```protobuf
syntax = "proto3"; // 声明了protobuf版本

package fixbug; // 声明了代码所在的包（生成C++代码后就是namespace 名字）

// 定义登录消息类型
message LoginRequest
{
    bytes name = 1; // =1 代表name是这个message第一个字段，不是指name的值
    bytes pwd = 2;
}

// 定义登录响应消息
message LoginResponse
{
    int32 errcode = 1;
    bytes errmsg = 2;
    bool success = 3;
}
```

## 例子2——列表

### protobuf文件test.proto

列表需要在遍变量声明前加`repeated`

```protobuf
syntax = "proto3"; // 声明了protobuf版本

package fixbug; // 声明了代码所在的包（生成C++代码后就是namespace 名字）

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
    bool success = 3;
}

message GetFriendListRequest
{
    uint32 userid = 1;
}

message User
{
    bytes name =1;
    uint32 age = 2;
    enum Sex
    {
        MAN = 0;
        WOMAN = 1;
    }
    Sex sex = 3;
}

message GetFriendListResponse
{
    ResultCode result = 1;
    repeated User friend_list = 2; // repeated表示列表类型！！！
}

```

### 测试源代码 main.cc

```cpp
#include "test.pb.h"
#include <iostream>
#include <string>
using namespace fixbug; // 工作中不要这样！！防止命名空间污染

int main()
{
    // LoginResponse rsp;
    // // 对象成员的修改方法，要先获取成员的指针
    // ResultCode *rc = rsp.mutable_result();
    // rc->set_errcode(1);
    // rc->set_errmsg("登录失败");

    GetFriendListResponse rsp;
    ResultCode *rc = rsp.mutable_result();
    rc->set_errcode(0);
   
    User *user1 = rsp.add_friend_list(); // 返回一个指向User类型指针，我们往里填数据就是在新的位置加入元素了
    user1->set_name("zhangsan");
    user1->set_age(20);
    user1->set_sex(User::MAN);

    User *user2 = rsp.add_friend_list(); // 返回一个指向User类型指针，我们往里填数据就是在新的位置加入元素了
    user2->set_name("lisi");
    user2->set_age(21);
    user2->set_sex(User::MAN);

    //std::cout << rsp.friend_list_size() << std::endl;

    // 打印列表元素
    int n = rsp.friend_list_size();
    for(int i = 0; i < n; ++i)
    {
        std::cout << rsp.friend_list(i).name() << " "<< rsp.friend_list(i).age() << " " << rsp.friend_list(i).sex() << std::endl;
    }
    return 0;
}

```

编译执行

```shell
jyhlinux@ubuntu:~/share/mprpc/test/protobuf$ protoc test.proto  --cpp_out=./
jyhlinux@ubuntu:~/share/mprpc/test/protobuf$ g++ main.cc test.pb.cc -o main -lprotobuf -L/usr/local/lib
jyhlinux@ubuntu:~/share/mprpc/test/protobuf$ ./main 
```

执行结果

zhangsan 20 0

lisi 21 0

## 例子3——定义方法类型（用于描述rpc方法）的成员

下面的例子使用`service` 关键字定义描述`rpc`方法的类型，并且注意开头要设置`option`

```protobuf
syntax = "proto3"; // 声明了protobuf版本

package fixbug; // 声明了代码所在的包（生成C++代码后就是namespace 名字）

// 定义下面的选项，表示生成service服务类和rpc方法描述，默认不生成
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
    bool success = 3;
}

message GetFriendListRequest
{
    uint32 userid = 1;
}

message User
{
    bytes name =1;
    uint32 age = 2;
    enum Sex
    {
        MAN = 0;
        WOMAN = 1;
    }
    Sex sex = 3;
}

message GetFriendListResponse
{
    ResultCode result = 1;
    repeated User friend_list = 2; // repeated表示列表类型
}

// 在protobuf里定义描述rpc方法的类型
service UserServiceRpc
{
    rpc Login(LoginRequest) returns(LoginResponse);
    rpc GetFriendList(GetFriendListRequest) returns(GetFriendListResponse);
}
```

通过`protoc`会生成的`UserServiceRpc`类(服务提供方使用)和 `UserServiceRpc_Stub`类（服务调用方使用）

![在这里插入图片描述](image/cf7087cc9dc245ee911ec78f9b767646.png#pic_center)

这也是`ServiceProvider`，即`rpc`服务提供者。

`UserServiceRpc_Stub`类是服务消费者即`serviceConsumer`

![在这里插入图片描述](image/5ffbf06f749b4cd5a605cd98b0f55988.png#pic_center)



