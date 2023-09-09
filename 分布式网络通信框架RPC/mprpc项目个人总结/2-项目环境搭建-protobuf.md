### xshell vscode 

### boost库，muduo库，cmake工具



## protobuf安装

建议安装时，都用管理员权限安装

### 上传安装包到package文件夹

![image-20230901221027963](image/image-20230901221027963.png)

### 1.解压压缩包

```
unzip protobuf-master.zip
```

![image-20230901221208846](image/image-20230901221208846.png)

### 2.进入解压后的文件夹

```
cd protobuf-master
```



### 3.安装所需工具

```
sudo apt-get install autoconf automake libtool curl make g++ unzip 
```

![image-20230901221420883](image/image-20230901221420883.png)

### 4.自动生成configure配置文件

一键执行配置文件

```
./autogen.sh
```

![image-20230901221523948](image/image-20230901221523948.png)

### 5.配置环境

一键执行环境配置文件

```
./configure
```

![image-20230901221559923](image/image-20230901221559923.png)



### 6.编译源代码（时间比较长）

cmake 生成makefile

```
make
```

![image-20230901221710515](image/image-20230901221710515.png)

### 7.安装

根据makefile来编译项目

```
sudo make install
```

![image-20230901223220885](image/image-20230901223220885.png)



### 8.刷新动态库

```
sudo ldconfig
```

![image-20230901223321601](image/image-20230901223321601.png)



### 测试一下

测试protoc命令，看是否protobuf安装成功

```
protoc
```

![image-20230901223430550](image/image-20230901223430550.png)





## 创建项目目录

```
cd mprpc
mkdir bin
mkdir build
mkdir example
mkdir lib
mkdir src

```

![image-20230903144349112](image/image-20230903144349112.png)
