## 日志模块的设计

本项目所实现的日志模块采用**异步**工作方式，多个线程往队列写日志，一个专门的写线程从队列中读取日志信息写入磁盘中的日志文件。

**要点**：线程安全，[线程间通信](https://so.csdn.net/so/search?q=%E7%BA%BF%E7%A8%8B%E9%97%B4%E9%80%9A%E4%BF%A1&spm=1001.2101.3001.7020)，队列，生产者消费者模型

日志模块设计图如下：

![在这里插入图片描述](image/5c6dee47852c4a1d827e5a5b3c0ffa31.png#pic_center)

## 源代码实现

### 队列的实现

[模板类](https://so.csdn.net/so/search?q=%E6%A8%A1%E6%9D%BF%E7%B1%BB&spm=1001.2101.3001.7020)的声明和成员函数实现最好都是实现在同一个头文件，分文件编写可能造成链接错误，日志模块是生产者消费者模型。

```cpp
// mprpc/src/include/lockqueue.h
#pragma once
#include <queue>
#include <thread>
#include <mutex> 
#include <condition_variable>

// 模板类的声明和成员函数实现最好都是实现在同一个头文件
// 分文件编写可能造成链接错误
// 异步写日志使用的队列
// 生产者消费者模型
template<typename T>
class LockQueue
{
public:
    // producer
    void Push(const T &data)
    {
        std::lock_guard<std::mutex> lock(m_mutex);
        m_queue.push(data);
        m_cond.notify_one(); // 消费者只有一个写线程
    }

    // 一个写线程读日志queue，将其写入磁盘中的日志文件
    T Pop()
    {
        std::unique_lock<std::mutex> lock(m_mutex);
        while(m_queue.empty()) // 防止虚假唤醒
        {
            // 日志队列为空，进入wait状态
            m_cond.wait(lock);
        }
        T data = m_queue.front();
        m_queue.pop();
        return data;
    }
private:
    std::queue<T> m_queue;
    std::mutex m_mutex;
    std::condition_variable m_cond;
};

```

### 日志模块Logger类实现

类的声明如下

```cpp
// logger.h 
#pragma once
#include <string>
#include "lockqueue.h"

class Logger;
// 提供用户使用日志模块的宏定义方法 ,LOG_INFO("xxx %d %s", 20, "abc")
#define LOG_INFO(fmt, ...)                         \
    do                                             \
    {                                              \
        Logger &logger = Logger::GetInstance();    \
        logger.setLogLevel(INFO);                  \
        char c[1024] = {0};                        \
        sprintf(c, fmt, ##__VA_ARGS__);            \
        logger.Log(c);                             \
    } while (0)

#define LOG_ERROR(fmt, ...)                         \
    do                                             \
    {                                              \
        Logger &logger = Logger::GetInstance();    \
        logger.setLogLevel(ERROR);                 \
        char c[1024] = {0};                        \
        sprintf(c, fmt, ##__VA_ARGS__);            \
        logger.Log(c);                             \
    } while (0)

// 定义日志级别
enum LogLevel
{
    INFO,
    ERROR,
};

// 单例类
class Logger
{
public:
    static Logger &GetInstance();
    void setLogLevel(LogLevel level);
    void Log(std::string msg);

private:
    int m_loglevel;
    LockQueue<std::string> m_lckQue; // 日志缓冲队列

    Logger();
    // 拷贝构造和移动构造
    Logger(const Logger &) = delete;
    Logger(Logger &&) = delete;
};
```

类的实现如下

```cpp
// logger.cc
#include "logger.h"
#include <time.h>
#include <iostream>

// 懒汉式
Logger& Logger::GetInstance()
{
    static Logger logger;
    return logger;
}

Logger::Logger()
{
    // 启动专门写日志的线程
    std::thread writeLogTask([&](){
        for(;;)
        {
            // 获取当前日期
            time_t now = time(nullptr);
            tm *nowtm = localtime(&now);

            // 组织文件名
            char file_name[128] = {0};
            sprintf(file_name,"%d-%d-%d-log.txt", nowtm->tm_year + 1900, nowtm->tm_mon + 1, nowtm->tm_mday);

            FILE *pf = fopen(file_name, "a+");
            if(pf == nullptr)
            {
                std::cout << "logger file:" << file_name << "open error!" << std::endl;
                exit(EXIT_FAILURE);
            }

            // 往文件中写日志
            std::string msg = m_lckQue.Pop();

            char time_buf[128] = {0};
            sprintf(time_buf, "%d:%d:%d=>[%s]",
                    nowtm->tm_hour,
                    nowtm->tm_min,
                    nowtm->tm_sec,
                    (m_loglevel == INFO ? "INFO" : "ERROR"));
            msg.insert(0, time_buf); // 消息前面加上时间 和 日志级别,查看时便于搜索
            msg.append("\n"); 

            fputs(msg.c_str(), pf);
            fclose(pf); // 写一行日志就关闭文件
        }
    });
    //
    writeLogTask.detach();
}


void Logger::setLogLevel(LogLevel level)
{
    m_loglevel = level;
}
void Logger::Log(std::string msg)
{
    m_lckQue.Push(msg);
}
```

### 日志文件输出

![在这里插入图片描述](image/db27639ec49e40c8a6537b37a8919d09.png#pic_center)

 