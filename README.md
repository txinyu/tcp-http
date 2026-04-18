# C++ 主从 Reactor + HTTP/1.1 服务器

纯 C++11 手写 | 无第三方依赖 | 主从 Reactor 多线程 | epoll LT | 动静分离 | 生产可用

## 项目介绍

本项目完全融合两篇博客源码，是一套从底层网络框架到上层 HTTP 业务全自研的工业级服务器：

1. 底层网络框架：主从 Reactor 多线程、epoll、Buffer、Channel、Poller、EventLoop、时间轮定时器、线程池、连接管理。
2. 上层 HTTP/1.1 服务：状态机解析、动静资源分离、正则路由、长短连接自动管理、防目录穿越、完整 MIME / 状态码。

整套代码可直接编译、学习 Linux 网络编程、二次开发上线。

## 核心特性

- 纯 C++11，无任何第三方依赖
- 主从 Reactor 多线程架构，one loop per thread
- epoll LT + 非阻塞 IO，高并发支撑
- 双偏移 Buffer，自动扩容，解决粘包 / 半包
- 时间轮定时器 O (1) 处理超时连接
- HTTP 分阶段状态机解析，完美兼容 HTTP/1.1
- 静态资源托管（html/css/js/png/ico）
- 动态正则路由（GET/POST/PUT/DELETE）
- 防目录穿越、防超长请求、安全加固
- 全自动 keep-alive 长连接
- 完整 HTTP 状态码 + 80+ 种 MIME 映射
- 模块化分层，易扩展、易维护

## 项目文件结构树

```
.
├── CMakeLists.txt                # 构建配置
├── main.cpp                      # 项目入口（HTTP 示例）
├── server.hpp                    # Reactor 核心框架（第一篇全部）
│   ├── 日志宏、Buffer、Socket、Channel
│   ├── Poller、TimerWheel、EventLoop
│   ├── LoopThread、LoopThreadPool
│   ├── Any、Connection、Acceptor、TcpServer
│
├── http/                         # HTTP 业务层（第二篇全部）
│   ├── HttpDef.hpp               # 状态码 + MIME 映射
│   ├── Util.hpp                  # 工具类：编解码/文件/路径校验
│   ├── HttpRequest.hpp           # 请求结构体
│   ├── HttpResponse.hpp          # 响应结构体
│   ├── HttpContext.hpp           # 状态机解析器
│   └── HttpServer.hpp             # HTTP 顶层服务器
│
└── wwwroot/                      # 静态资源目录
    ├── index.html
    ├── 404.html
    └── favicon.ico
```
## 架构分层

```
【工具层】       Logger / Any / Util / Buffer
【内核封装层】   Socket / Poller(epoll)
【事件驱动层】   Channel / EventLoop / TimerWheel
【线程模型层】   LoopThread / LoopThreadPool
【连接管理层】   Connection / Acceptor / TcpServer
【HTTP 协议层】  HttpRequest / HttpResponse / HttpContext
【HTTP 业务层】  HttpServer（动静分离 + 路由分发）
```
## 核心模块说明
### Reactor 网络核心（server.hpp）

- Buffer：双偏移读写分离、自动扩容、空间复用
- Socket：RAII 封装、非阻塞、地址 / 端口重用
- Channel：fd + 事件 + 回调，Reactor 最小单元
- Poller：epoll 封装，事件监听与分发
- EventLoop：事件循环、无锁串行、线程安全
- TimerWheel：O (1) 时间轮，超时连接清理
- LoopThreadPool：主从 Reactor 线程池，轮询负载均衡
- Connection：TCP 连接全生命周期管理
- TcpServer：顶层入口，一键启动服务器
###  HTTP 业务核心（http/）

- HttpDef：HTTP 状态码 + MIME 类型映射
- Util：URL 编解码、文件读写、路径安全校验（防目录穿越）
- HttpRequest/HttpResponse：请求 / 响应结构化封装
- HttpContext：5 阶段状态机解析，解决粘包半包
- HttpServer：动静分离、正则路由、响应生成、连接管理

### 快速上手（main.cpp）

```
#include "http/HttpServer.hpp"
using namespace std;

int main() {
    // 1. 创建 HTTP 服务器，监听 8080，超时 10s
    HttpServer server(8080, 10);

    // 2. 设置静态资源根目录
    server.SetBaseDir("./wwwroot");

    // 3. 设置工作线程数
    server.SetThreadCount(4);

    // 4. 注册动态接口
    server.Get("^/hello$", [](const HttpRequest &req, HttpResponse *rsp) {
        rsp->SetContent("{\"code\":200,\"msg\":\"Hello HTTP Server!\"}", "application/json");
    });

    server.Get("^/user/(\\d+)$", [](const HttpRequest &req, HttpResponse *rsp) {
        string uid = req._matches[1];
        rsp->SetContent("{\"code\":200,\"uid\":\""+uid+"\"}", "application/json");
    });

    // 5. 启动服务
    server.Listen();
    return 0;
}

```
### 编译与运行

```
mkdir build && cd build
cmake ..
make
./HttpServer
```

访问测试：
- 静态页面：http://127.0.0.1:8080/index.html
- 动态接口：http://127.0.0.1:8080/hello
- 参数路由：http://127.0.0.1:8080/user/10086

## 请求处理完整流程

1. 客户端建立 TCP 连接
2. Acceptor 接收新连接，分配子 Reactor 线程
3. 数据写入 Buffer，触发 OnMessage
4. HttpContext 状态机解析 HTTP 请求
5. 路由分发：静态资源 / 动态接口
6. 构建 HttpResponse，序列化为响应报文
7. 发送响应，自动判断长短连接
8. 超时连接由时间轮自动清理

## 亮点
- 状态机解析彻底解决 TCP 粘包 / 半包
- 防目录穿越、防超长请求、安全加固
- 无锁串行设计，天然线程安全
- 全自动长连接，性能最优
- 完整错误处理，服务不崩溃
- 极简 API，3 分钟快速部署

## 可扩展方向
- 支持 gzip 压缩
- 接入 HTTPS (OpenSSL)
- 文件上传（multipart/form-data）
- 异步日志系统
- 限流、黑名单、IP 封禁
- HTTP/2 支持
- 协程化改造

## 适用场景
- C++ Linux 网络编程学习
- 高性能 HTTP 接口服务
- 轻量级 Web 服务器
- 嵌入式 HTTP 服务

## 参考资料
- [手写 C++ 高性能 Reactor 网络服务器](https://blog.csdn.net/m0_62807361/article/details/157101505)
- [（续篇）手写 C++ 完整 HTTP/1.1 服务器](https://blog.csdn.net/m0_62807361/article/details/157131292)



