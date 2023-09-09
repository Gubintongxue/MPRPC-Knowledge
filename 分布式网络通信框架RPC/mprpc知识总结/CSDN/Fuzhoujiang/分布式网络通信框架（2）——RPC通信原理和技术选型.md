## 项目实现功能

![在这里插入图片描述](image/5bf2aefa949b4ea09c6e8ed33b3d3920.png#pic_center)

## 技术选型

**黄色部分**：设计rpc方法参数的**打包和解析**，也就是数据的序列化和[反序列化](https://so.csdn.net/so/search?q=%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96&spm=1001.2101.3001.7020)，用`protobuf`做`RPC`方法调用的序列化和反序列化。

**使用protobuf的好处:**

`protobuf`是**二进制**存储，`xml`和`json`是**文本**存储；

`protobuf`不需要存储额外信息；而`json`存储`key-value`，key浪费空间

**绿色部分**：网络部分，包括寻找`rpc`服务主机，发起`rpc`调用请求和响应`rpc`调用结果，使用`muduo`网络库和左`zookeeper`服务配置中心（服务发现）

## 项目代码工程目录

`bin`：可执行文件

`build`：项目编译文件

`lib`：项目库文件

`src`：源文件

`test`：测试代码

`example`：框架代码使用范例

`CMakeLists.txt`：顶层的cmake文件

`README.md`：项目自述文件

`autobuild.sh`：一键编译脚本