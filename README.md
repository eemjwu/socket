# 前言
本文从套接字API的介绍到客户端、服务器的各种范式编写，整理Socket编程的基础。  
参考书籍：  
- 《UNIX网络编程 卷1：套接字联网API》
## 一、套接字编程简介
### 1. 套接字地址结构
每个协议族有自己的套接字地址结构。名字均以sockaddr_开头，以每个协议族唯一后缀结束。  
本文使用到的数据类型说明：  
![数据类型说明](https://raw.githubusercontent.com/eemjwu/socket/master/%E5%9B%BE%E7%89%87/%E6%95%B0%E6%8D%AE%E7%B1%BB%E5%9E%8B%E8%AF%B4%E6%98%8E.bmp)
#### 1. IPv4套接字地址结构
**名字**： struct sockaddr_in{ }    
**结构**：  
![image](https://raw.githubusercontent.com/eemjwu/socket/master/%E5%9B%BE%E7%89%87/IPv4%E5%A5%97%E6%8E%A5%E5%AD%97%E5%9C%B0%E5%9D%80.bmp)
#### 2. IPv6套接字地址结构
**名字**：struct sockaddr_in6{ }  
**结构**：
![image](https://raw.githubusercontent.com/eemjwu/socket/master/%E5%9B%BE%E7%89%87/IPv6%E5%A5%97%E6%8E%A5%E5%AD%97%E5%9C%B0%E5%9D%80%E7%BB%93%E6%9E%84.bmp)
#### 3. 通用套接字地址结构
**名字**：struct sockaddr{ }  
**结构**：
![image](https://raw.githubusercontent.com/eemjwu/socket/master/%E5%9B%BE%E7%89%87/%E9%80%9A%E7%94%A8%E5%A5%97%E6%8E%A5%E5%AD%97%E5%9C%B0%E5%9D%80%E7%BB%93%E6%9E%84.bmp)
### 2. Socket编程API
#### 1. 字节序转换函数
功能：将本地**端口号**转换为网络使用的端口号字节序
```
#include<netinet/in.h>  

// host -> net 主机字节序转为网络字节序
uint16_t    htons(uint16_t host16bitvalue);
uint32_t    htonl(uint32_t host32bitvalue);

// net -> host 网络字节序转为主机字节序
uint16_t    ntohs(uint16_t net16bitvalue);
uint32_t    ntohl(uint32_t net32bitvalue);
```
#### 2. 字节操纵函数

Berkeley 版：
```
#include <strings.h>

void bzero(void *dest, size_t nbytes);// 将目标设置为 0
void bcopy(const void *src, void *dest, size_t nbytes);
int bcmp(const void *ptrl, const void *ptr2, size_t nbytes);
```
ANSI C 版：

```
#include<string.h>

void *memset(void *dest, int c, size_t len);
void *memcpy(void *dest,const void *src, size_t nbytes);
int memcmp(const void *ptrl, const void *ptr2, sise_t nbytes);
```
tip：memcpy 类似 bcopy 但是指针参数相反  
memcpy参数顺序记忆：  
    dest = src；
#### 3. 地址转换函数

功能：将地址的表达形式（ASCII字符串），转换为网络地址字节序形式（二进制值）  

IPv4 版：
```
#include <arpa/inet.h>

// 例：“206.168.112.96” -> struct in_addr（32bit IPv地址）
int inet_aton(const char *strptr, struct in_addr *addrptr);
                                            返回：字符串有效为 1，否则为 0

// 例：struct in_addr（32bit IPv地址）-> “206.168.112.96” 
char *inet_ntoa(struct in_addr inaddr);
```
IPv4 IPv6 通用版：

```
#include <arpa/inet.h>

int inet_pton(int family, const char *strptr, void*addrptr);

const char *inet_ntop(int family, const void *addrptr, char *strptr, size_t len);
```
简析：  
- p 代表表达（presentation），n 代表数值（numeric）
- family 可以是AF_INET 或 AF_INET6
- len 是目标存储单元大小,在 <netinet/in.h>中
    ```
    #define INET_ADDRSTRLEN 16
    #define INET6_ADDRSTRLEN 46
    ```
示例：  

```
inet_pton(AF_INET, cp, &foo.sin_addr);
//替代：
inet_aton(cp,&foo.sin_addr);

char str[INET_ADDRSTRLEN];
ptr = inet_ntop(AFINET,&foo.sin_addr,str,sizeof(str));
//替代：
ptr = inet_ntoa(foo.sin_addr);
```
#### 4. sock_ntop和相关函数
说明：该系列函数是《UNP》封装的函数  
因为：  
inet_ntop 要求调用者传递一个指向某个二进制地址的指针，而该指针包含在套接字地址结构中，要求调用者熟悉套接字结构，例如必须这样写：  

```
// IPV4 时：
struct sockaddr_in addr;
inet_ntop(AF_INET,&addr.sin_addr,str,szieof(str));

// IPv6 时：
struct sockaddr_in6 addr;
inet_ntop(AF_INET6,&addr.sin6_addr,str,szieof(str));
```
使得代码与协议相关了。
其他函数：
![image](https://raw.githubusercontent.com/eemjwu/socket/master/%E5%9B%BE%E7%89%87/%E8%87%AA%E5%AE%9A%E4%B9%89%E5%87%BD%E6%95%B0.bmp)

#### 5. readn 、 writen、 readline函数
说明：该系列函数是《UNP》封装的函数  
因为：  
字节流套接字上的read 和 write 函数输入或输出字节可能比请求的少，然而这并不是出错状态，原因可能是内核中套接字的缓冲区到达了极限，需要再次调用read 或write，以输入输出剩余字节。  

```
#include "unp.h"

ssize_t readn(int filedes, void *buff, size_t nbytes);

ssize_t written(int filedes, const void *buff, size_t nbytes );

ssize_t readline(int filedes, void *buff, size_t maxlen);
                                返回：读或写的字节数，若出错则为-1
```
具体实现见《UNP》pag：72-75
### 二、基本TCP套接字编程
![4-1](https://raw.githubusercontent.com/eemjwu/socket/master/%E5%9B%BE%E7%89%87/4-1.jpg)
#### 1. socket 函数
描述：创建新的套接字  
原型：  

```
#include<sys/socket.h>

int socket(int family, int type, int protocol);
                                    返回：成功返回描述符，失败-1
```
参数值：  
family | 说明
--- | ---
AF_INET | IPv4协议
AF_INT6 | IPv6协议
AF_LOCAL | Unix域协议
AF_ROUTE| 路由套接字
AF_KEY | 秘钥套接字

type | 说明
--- | ---
SOCK_STREAM | 字节流套接字
SOCK_DGRAM | 数据报套接字
SOCK_SEQPACKET | 有序分组套接字
SOCK_RAM | 原始套接字

protocol | 说明
--- | ---
IPPROTO_TCP | TCP传输协议
IPPROTO_UDP | UDP传输协议
IPPROTO_SCTP | SCTP传输协议
#### 2. connect函数
描述：建立与服务器的连接  
原型：   

```
#include <sys/socket.h>

int connect(int sockfd, const struct sockaddr *servaddr,socklen_t addrlen);
                                                    返回：成功 0，失败 -1
```
说明：  
1. 调用connect函数触发三路握手，仅在连接建立成功或出错时才返回，出错返回有以下情况：
   1. TCP客户端没有收到SYN分节的**响应**，返回 **ETIMEDOUT** 错误(在多次尝试失败后)；
   2. TCP客户端收到的 SYN 响应是 RST，立即返回 **ETIMEDOUT** 错误(**硬错误**，表明服务器主机在指定的端口上没有进程等待连接)
   3. TCP客户端发出的 SYN 在中间的路由上引发 “destination unreachable”，继续尝试重发，失败则返回相应的 ICMP 错误信息(**软错误**)
   4. **产生 RST 错误分节的条件**：
      1. SYN 到达目的，但指定的端口没有进程
      2. TCP 取消一个已有连接
      3. TCP 接收到一个不存在的连接的分节
2. 若 connect 失败，则该套接字不能用，必须 close，再重新调用 socket 创建新的套接字。
#### 3. bind 函数
描述：给套接字绑定一个本地协议地址(ip port)  
原型：

```
#include <sys/socket.h>

int bind(int sockfd,const struct sockaddr *myaddr, socklen_t addrlen);
                                                返回：成功0 失败 -1
```
说明：  
给 bind 函数指定要绑定的 IP 地址和/或端口号产生的结果  
通配地址：INADDR_ANY
进程指定IP | 指定端口 | 结果
--- | --- | ---
通配地址 | 0 | 内核选择 IP 地址和端口
通配地址 | 非0 | 内核选择 IP 地址，进程指定端口
本地 IP 地址 | 0 | 进程指定 IP 地址，内核选择端口
本地 IP 地址 | 非 0 | 进程指定 IP 地址和选择端口
#### 4. listen 函数
描述：listen 函数仅由 TCP 服务器调用  
原型：  
```
#include <sys/socket.h>

int listen(int sockfd, int backlog);
            返回：成功 0 失败 -1
```
说明：  
backlog：相应套接字允许排队的**最大连接个数**  
内核为每个监听套接字维护两个队列：  
(1) 未完成连接队列(SYN_RCVD 状态)  
(2) 已完成连接队列(ESTABLISHED 状态)  
![4-7](https://raw.githubusercontent.com/eemjwu/socket/master/%E5%9B%BE%E7%89%87/4-7.jpg)
#### 5. accept 函数
描述：服务器调用，从已完成连接队列队列头返回下一个已完成的连接  
原型：

```
#include <sys/socket.h>

int accept(int sockfd, struct sockaddr *cliaddr, socklen_t *addrlen);
                                        返回：成功 返回描述符， 出错 -1
```
说明:  
返回： **已连接套接字描述符**，形参： **监听套接字描述符**  
cliaddr 和 addrlen 用来返回已连接的对端的协议地址。  
成功返回一个全新的**描述符**，代表与所返回客户端的 TCP 连接。
#### 6. fork 函数
描述：fork 产生新进程的唯一方法；父子进程共享**fork之前打开的**描述符，描述符引用计数 +1  
原型：

```
#include <unistd.h>

pid_t fork(void);
            返回：子进程中为 0，父进程为子进程 ID，出错 -1
```
#### 7. exec 函数
描述：把当前进程替换成新的程序文件，进程 ID 不变 ；原进程打开的描述符通常跨 exec 继续打开  
原型：

```
#include <unistd.h>

int execl(const char *pathname, const char *arg0, .../* (char *) 0 */);

int execv(const char *pathname, char *const *argv[]);

int execle(const char *pathname, const char *arg0, .../* (cahr *) 0, char *const envp[] */);

int execve(const char *pathname, char *const *argv[], char *const envp[]);

int execlp(const char *fliename, const char *arg0, .../* (char *) 0 */);

int execvp(const char *filename, char *const argv[]);
                                        返回：成功不返回，出错 -1
```
![4-12](https://raw.githubusercontent.com/eemjwu/socket/master/%E5%9B%BE%E7%89%87/4-12.jpg)
#### 8. close 函数
描述：关闭套接字
原型：  

```
#include <unistd.h>

int close(int sockfd);
            返回： 成功 0，失败 -1
```
说明：  
- 调用close **套接字描述符引用计数减 1**，若减 1 后引用计数为 0 ，发送 FIN 开启 TCP 连接终止过程
- 立即结束 TCP 连接，使用 shutdown 函数
- 并发服务器编程时，**必须在父进程关闭已连接描述符**，否则导致套接字描述符总是大于 1， **连接不会被真正终**止，并可能**耗尽可用描述符**
#### 9. getsockname 函数和 getpeername 函数
描述：  
getsockname 返回与该套接字关联的**本地协议地址**  
getpeername 返回与该套接字关联的**外地协议地址**  
原型：

```
#include <sys/socket.h>

int getsockname(int sockfd, struct sockaddr *localaddr, socklen_t *addrlen);

int getpeername(int sockfd, struct sockaddr *peeraddr, socklen_t *addrlen);
                                                    返回：成功 0，失败 -1
```
说明：   
当服务器是由通过accept的某个进程通过调用exec执行程序时，获取客户身份的唯一途径就是调用
getpeername 
![4-18](https://raw.githubusercontent.com/eemjwu/socket/master/%E5%9B%BE%E7%89%87/4-18.jpg)





