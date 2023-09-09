**首先完善mprpc目录下的CMakeLists.txt**  
![在这里插入图片描述](image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0xJTlpFWVU2NjY=,size_16,color_FFFFFF,t_70.png)  
**然后我们在src底下增加一个CMakeLists.txt**  
![在这里插入图片描述](image/b77b92d5b9fe4f2eb8edfc6b235f8dca.png)

```xml
aux_source_directory(. SRC_LIST)
add_library(mprpc ${SRC_LIST})
```

![在这里插入图片描述](image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0xJTlpFWVU2NjY=,size_16,color_FFFFFF,t_70-16935583876531.png)  
**然后我们打开callee下的CMakeLists.txt**

```xml
set(SRC_LIST userservice.cc ../user.pb.cc)
add_executable(provider ${SRC_LIST})
target_link_libraries(provider mprpc protobuf)
```

![在这里插入图片描述](image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0xJTlpFWVU2NjY=,size_16,color_FFFFFF,t_70-16935583876532.png)  
**然后我们点击**  
![在这里插入图片描述](image/30e981e3d1c040da83c75a156868a3ce.png)  
![在这里插入图片描述](image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0xJTlpFWVU2NjY=,size_16,color_FFFFFF,t_70-16935583876533.png)  
**点击最上面中间那个键 编译**  
![在这里插入图片描述](image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0xJTlpFWVU2NjY=,size_16,color_FFFFFF,t_70-16935583876534.png)

**编译后，我们查看项目工程列表，看看多出了什么**  
![在这里插入图片描述](image/d9c19cf594304d31b0f891aa16e8a392.png)