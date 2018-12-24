
[TOC]

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
#### 1. 字节序转换函数 htonl 和 ntohl
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
#### 2. 字节操纵函数 bzero 等

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
#### 3. 地址转换函数 inet_pton 和 inet_ntop

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
## 二、基本TCP套接字编程
![4-1](https://raw.githubusercontent.com/eemjwu/socket/master/%E5%9B%BE%E7%89%87/4-1.jpg)
### 1. socket 函数
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
### 2. connect函数
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
### 3. bind 函数
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
### 4. listen 函数
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
### 5. accept 函数
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
### 6. fork 函数
描述：fork 产生新进程的唯一方法；父子进程共享**fork之前打开的**描述符，描述符引用计数 +1  
原型：

```
#include <unistd.h>

pid_t fork(void);
            返回：子进程中为 0，父进程为子进程 ID，出错 -1
```
### 7. exec 函数
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
### 8. close 函数
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
### 9. getsockname 函数和 getpeername 函数
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
getpeername函数   
![4-18](https://raw.githubusercontent.com/eemjwu/socket/master/%E5%9B%BE%E7%89%87/4-18.jpg)
##  三、TCP 客户端/服务器程序范式
编写一个完整的 TCP 客户端/服务器程序：  
1. 客户从标准输入读入一行文本，并写给服务器；
2. 服务器从网络输入读入这行文本，并回射给客户；
3. 客户从网络输入读入这行回射文本，并显示在标准输出上  
// 图5-1

### 1. 其他函数准备
#### 1. TCP 回射服务器程序： str_echo 函数  
原型：  

```
#include “unp.h”

void str_echo(int sockfd)
{
    ssize_t n;
    char    buf[MAXLINE];
    
again:
    while((n = read(sockfd, buf, MAXLINE)) > 0)
        Writen(sockfd, buf, n);
    
    if(n < 0 && errno == EINTR) // 被中断后继续执行
        goto again;
    else if(n < 0)
        err_sys("str_echo: read error");
}
```
#### 2. TCP 回射客户端程序： str_cli 函数
原型：

```
#include "unp.h"

void str_cli(FILE *fp, int sockfd)
{
    char    sendline[MAXLINE], recvline[MAXLINE];
    
    while(Fgets(sendline, NAXLINE, fp) != NULL)
    {
        Writen(sockfd, sendline, strlen(sendline));
        
        if(Readline(sockfd, recvline, MALINE) == 0)
            err_quit("str_cli: server terminated prematurely");
        Fputs(recvline, stdout);
        
    }
    
}
```
### 2. TCP 客户端/服务器设计范式
#### 1. 迭代式 
##### TCP 客户端

```
#include "unp.h"

int main(int argc, char **argv)
{
        int sockfd;
        struct sockaddr_in servaddr;
        
        if(argc != 2)
            err_quit("usage: tcpcli <IPaddress>");
        
        sockfd = Socket(AF_INET, SOCK_STREAM, 0);
        
        bzero(&servaddr, sizeof(servaddr));
        servaddr.sin_family = AF_INET;
        servaddr.sin_port = htons(SERV_PORT);
        Inet_pton(AF_INET, argv[1], &servaddr.sin_addr);
        
        Connect(sockfd, (SA *) &servaddr, sizeof(servaddr));
        
        str_cli(stdin, sockfd); /*do it all*/
        
        exit(0);    
}
```
##### TCP 服务器端

```
#include "unp.h"

int main(int argc, char **argv)
{
    int listenfd, connfd;
    pid_t childpid;
    socklen_t clilen;
    struct sockaddr_in cliaddr, servaddr;
    
    void sig_chld(int);
    
    listenfd = Socket(AF_INET, SOCK_STREAN, 0);
    
    bzero(&servaddr, sizeof(servaddr));
    servaddr.sin_family = AF_INET;
    servaddr.sin_addr.s_addr = htonl(INADDR_ANY);
    servaddr.sin_port = htons(SERV_PORT);
    
    Bind(listenfd, (SA *)&servaddr, sizeof(servaddr));
    
    Listen(listenfd, LISTENQ);
    
    Signal(SIGCHLD, sig_chld);  /*必须调用 waitpid() 回收僵尸进程*/
    
    for( ; ; )
    {
        chilen = sizeof(chiaddr);
        if((connfd = accept(listenfd, (SA *) &cliaddr, &clilen)) < 0)
        {
            if(errno == EINTR)  // 被中断重新调用
                continue;
            else
                err_sys("accept error");
        }
        if((childpid = Fork()) == 0)/* 子进程*/
        {
            Close(lisenfd); /*关闭监听套接字*/ // 关闭后面没用的套接字
            str_echo(connfd);
            exit(0);
        }
        Close(connfd);  /*必须关闭 不然一直占用描述符*/
    }
    
}

/**********************************************************************************/
// 回收子进程 避免产生僵尸进程
void sig_chld(int signo)
{
    pid_t pid;
    int stat;
    
    while( (pid = waitpid(-1, &stat, WNOHANG)) > 0) // WNOHANG 不阻塞
        printf("child %d terminated\n", pid);
        
    return;
}
```
##### 问题分析
###### 1. 僵尸进程与处理
1. 产生：
   - 子进程终止时发送给父进程一个 SIGCHLD 信号，若父进程没有捕获改信号处理，则子进程变为僵尸进程。
2. POSIX 信号处理知识：  
    1) 父进程通过调用 sigaction 函数捕获信号，捕获到信号有三种处理方式：
       - 提供一个函数，只要特定信号发生，就调用该函数。SIGKILL 和 SIGSTOP 不能被捕获  
       信号处理函数原型：  
            ```
            void handler(int signo);// 无返回值， 形参为 信号
            ```
       - 把信号的处置设置为 SIG_NGN 忽略信号。SIGKILL 和 SIGSTOP 不能忽略
       - 把信号处置设置为 SIG_DEF 启动默认处置
       - [signal 函数原型](https://note.youdao.com/) // 代码  
    2) POSIX 信号处理总结
       - 一旦安装了信号处理函数，便一直有效
       - **在一个信号处理函数运行期间，正被递交的函数是阻塞的**
       - 如果一个信号在被阻塞期间产生了一次或多次，那么该信号被解除阻塞后只递交一次，**Unix 信号默认是不排队的**
3. **使用 waitpid 函数代替 wait 函数**，因为 Unix 信号是不排队的，当同时出现多个子进程的 SIGCHLD 信号时，wait 可能不能全部处理所有信号
###### 2. accept 中必须处理中断
1. 当 SIGCHLD 信号递交时，父进程阻塞于 accept 系统调用，内核会使 accept 返回一个 EINTR 错误( 被中断的系统调用 )，**所以必须在程序 accept 中处理该错误**，重新启动 accept 系统调用。  
2. 对于 accept 以及诸如read、write、select 和 open 之类的函数来说，**重启**被中断的系统调用时合适的，**但是 connect 除外**，重启 connect 将返回错误，必须调用 select 来等待连接完成，后文有解释。
###### 3. 总结
网络编程时可能会遇到的三种情况：  
1. **当 fork 子进程时，必须捕获 SIGCHLD 信号**；
2. **当捕获信号时，必须处理被中断的系统调用**；
3. **SIGCHLD 的信号处理函数必须正确编写，应使用 waitpid 函数以免留下僵尸进程**。
##### 服务器异常分析
###### 服务器进程终止
1. 服务器进程终止，发送 FIN 给客户端
2. 客户端阻塞在 fget 上不能立即响应该 FIN
   - 这就是引入 select 和 poll 的原因之一，客户端不能单纯阻塞在某个特定套接字描述符上，而应该阻塞在任意输入套接字描述
3. 等待用户输入文本后， str_cli 函数调用 writen 把数据发送给服务器
4. 服务器接收到数据，响应 RST
5. 客户端此时阻塞在 readline 上，看不到这个 RST，并且由于第 1 步中的 FIN ，readline 立即返回 0 ，所以 str_cli 第 9 行，打印 “str_cli：server terminated prematurely”
6. 客户端终止，关闭所有打开的描述符
###### SIGPIPE 信号
1. 产生：  
如上第 5 步，客户端内核收到 RST 而，客户端进程并未及时处理，假如此时进程继续向对端服务器发送数据时( 调用 write  函数)，客户端内核将向该进程发送 SIGPIPE 信号。
2. 处理  
SIGPIPE 信号默认行为是终止进程，因此进程必须捕获它以免不情愿的被终止
3. write 函数返回 EPIPE 错误
   1. 写一个接收了 FIN 的套接字正确( CLOSE_WAIT 状态)；
   2. 写一个接收了 RST 的套接字 EPIPE 错误
###### 服务器主机崩溃
1. 过程  
    1. 服务器崩溃时，客户端不会收到任何通知  
    2. 客户端调用 wtrten 时，客户端 TCP 持续重传数据，试图从服务器接收 ACK
    3. 重传过程结束还是没收到服务器 ACK ，客户端返回错误 ETIMEDOUT  
2. 处理  
    1. 对 readline 调用设置超时，提前得知服务器崩溃信息，不必等待重传机制完成  
    2. 设置 SO_KEEPALIVE 心跳保活选项
###### 服务器主机崩溃后重启
1. 服务器崩溃重启后丢失之前的所有 TCP 连接，因此服务器收到客户端的消息直接返回 RST 
2. 客户端收到 RST 返回 ECONNRESET 错误
###### 服务器主机关机
1. Unix 系统关机时，init进程给所有进程发送 SIGTERM 信号
2. 等待(5-20 秒)，留给程序小段时间来清除和终止，然后给所有仍在运行的进程发送 SIGKILL 杀死进程
3. 若不捕获 SIGTERM ，服务器进程由 SIGKILL 终止，随后发生的步骤如上服务器进程终止  

###### 总结
**必须在客户端程序中使用 select 或 poll 函数，使服务器进程终止一经发生，立刻检测到**
#### 2. select I/O复用式
##### 1. 基本 I/O 模型
###### 1. 阻塞式 I/O 模型
进程调用某系统调用，直到任务完成，或者出现错误时，该系统才返回，称进程在系统调用期间是阻塞的。  
// 图6-1
###### 2. 非阻塞式 I/O 模型
与阻塞式相反，当系统调用没出现期望的条件时，进程不进入睡眠而是立刻返回一个错误。  
// 图6-2
###### 3. I/O 复用模型
调用 select 或 poll 阻塞在这两个系统调用上，而不是阻塞在真正的 I/O 系统调用上。使用 select 的优势在于可以等待多个描述符就绪。  
// 图6-3
###### 4. 信号驱动式 I/O 模型
内核在描述符就绪时发送 SIGIO 信号。首先开启套接字信号驱动功能，并通过 sigaction 安装一个信号处理函数，系统调用立即返回，不阻塞。  
// 图6-4
###### 5. 异步 I/O 模型
告知内核启动某个操作，并让内核在**整个操作完成**之后通知进程。与信号驱动模型的区别：信号驱动 I/O是告知内核何时可以启动一个 I/O ，异步模型是由内核通知我们 I/O 操作何时**完成**。  
// 图6-5
###### 6. 各种 I/O 模型比较
阻塞式 I/O 模型、非阻塞式 I/O 模型、I/O 复用模型和信号驱动式 I/O 模型都是同步 I/O 模型；因为其真正的 I/O 操作将阻塞进程。    
// 图6-5
#### 2. select 函数
描述：允许进程指示内核等待多个事件中的任何一个发生，并且只有在有一个或多个事件发生或经历一段指定的时间后再唤醒它。
原型：  

```
#include <sys/select.h>
#include <sys/time.h>

int select(int maxfdp1, fd_set *readset, fd _set *writeset, fd_set *exceptset, const struct timeval *timeout);
                                        返回：有指定的描述符就绪则为其数目，超时为 0 ，出错为 -1
```
说明：  
1. timeout 参数：  
   - NULL 永远等待，直到一个描述符准备好
   - 指定的秒数和微秒数，直到一个描述符准备好，或者定时时间到
   - 0，根本不等待，检查描述符后立即返回，称之为轮询
2. readset，writeset，exceptset 指定内核测试读、写、异常条件的描述符，使用以下四个宏：
   - void FD_ZERO(fd_set *fdset)  // 初始化描述符集，全部置 0
   - void FD_SET(int fd, fd_set *fdset) // 将指定位置 1 
   - void FD_CLR(int fd, fd_set *fdset) // 将指定位置 0
   - int FD_ISSET(int fd, fd _set *fdset) // 测试某位是否为 1
3. maxfd1 指定待测试的描述符最大值 + 1
4. 返回：修改由指针 readset，writeset，exceptset 所指向的描述符集，就绪的置 1 ，未就绪的置 0，函数返回后调用 FD_ISSET 测试某描述符是否就绪
5. 描述符就绪条件  
// 图 6-7
6. 缓冲区问题：
   - 混合使用 stdio 和 select非常容易犯错误，因为 select 并不知道 stdio 使用了缓冲区
#### 3. shutdown 函数
描述：shutdown 不管引用计数，立刻激发 TCP 的正常连接终止序列，使客户端进入半关闭状态(不能发送但能接收)
原型：

```
#include <sys/socket.h>

int shutdown(int sockfd, int howto);
            返回：成功 0，失败 -1
```
howto 参数：
   - SHUT_RD 关闭连接的读这一半——套接字可发，不可收
   - SHUT_WR 关闭连接的写这一半——套接字可收，不可发
   - SHUT_RDWR 读写半部都关闭
#### 4. str_cli 函数
直接操作缓冲区，避免 stdio 缓冲区问题。
```
#include	"unp.h"

void
str_cli(FILE *fp, int sockfd)
{
	int			maxfdp1, stdineof;
	fd_set		rset;
	char		buf[MAXLINE];
	int		n;

	stdineof = 0;
	FD_ZERO(&rset);
	for ( ; ; ) {
		if (stdineof == 0)
			FD_SET(fileno(fp), &rset);
		FD_SET(sockfd, &rset);
		maxfdp1 = max(fileno(fp), sockfd) + 1;
		Select(maxfdp1, &rset, NULL, NULL, NULL);

		if (FD_ISSET(sockfd, &rset)) {	/* socket is readable */
			if ( (n = Read(sockfd, buf, MAXLINE)) == 0) {
				if (stdineof == 1)
					return;		/* normal termination */
				else
					err_quit("str_cli: server terminated prematurely");
			}

			Write(fileno(stdout), buf, n);
		}

		if (FD_ISSET(fileno(fp), &rset)) {  /* input is readable */
			if ( (n = Read(fileno(fp), buf, MAXLINE)) == 0) {
				stdineof = 1;
				Shutdown(sockfd, SHUT_WR);	/* send FIN */
				FD_CLR(fileno(fp), &rset);
				continue;
			}

			Writen(sockfd, buf, n);
		}
	}
}

```
#### 5. TCP 服务器程序(select 版)
当有客户端到达时，在 client 数组中的第一个可用项(值为 -1 的第一个项)中记录其已连接套接字的描述符。
```
/* include fig01 */
#include	"unp.h"

int
main(int argc, char **argv)
{
	int					i, maxi, maxfd, listenfd, connfd, sockfd;
	int					nready, client[FD_SETSIZE];
	ssize_t				n;
	fd_set				rset, allset;
	char				buf[MAXLINE];
	socklen_t			clilen;
	struct sockaddr_in	cliaddr, servaddr;

	listenfd = Socket(AF_INET, SOCK_STREAM, 0);

	bzero(&servaddr, sizeof(servaddr));
	servaddr.sin_family      = AF_INET;
	servaddr.sin_addr.s_addr = htonl(INADDR_ANY);
	servaddr.sin_port        = htons(SERV_PORT);

	Bind(listenfd, (SA *) &servaddr, sizeof(servaddr));

	Listen(listenfd, LISTENQ);

	maxfd = listenfd;			/* initialize */
	maxi = -1;					/* index into client[] array */
	for (i = 0; i < FD_SETSIZE; i++)
		client[i] = -1;			/* -1 indicates available entry */
	FD_ZERO(&allset);
	FD_SET(listenfd, &allset);
/* end fig01 */

/* include fig02 */
	for ( ; ; ) {
		rset = allset;		/* structure assignment */
		nready = Select(maxfd+1, &rset, NULL, NULL, NULL);

		if (FD_ISSET(listenfd, &rset)) {	/* new client connection */
			clilen = sizeof(cliaddr);
			connfd = Accept(listenfd, (SA *) &cliaddr, &clilen);
#ifdef	NOTDEF
			printf("new client: %s, port %d\n",
					Inet_ntop(AF_INET, &cliaddr.sin_addr, 4, NULL),
					ntohs(cliaddr.sin_port));
#endif

			for (i = 0; i < FD_SETSIZE; i++)
				if (client[i] < 0) {
					client[i] = connfd;	/* save descriptor */
					break;
				}
			if (i == FD_SETSIZE)
				err_quit("too many clients");

			FD_SET(connfd, &allset);	/* add new descriptor to set */
			if (connfd > maxfd)
				maxfd = connfd;			/* for select */
			if (i > maxi)
				maxi = i;				/* max index in client[] array */

			if (--nready <= 0)
				continue;				/* no more readable descriptors */
		}

		for (i = 0; i <= maxi; i++) {	/* check all clients for data */
			if ( (sockfd = client[i]) < 0)
				continue;
			if (FD_ISSET(sockfd, &rset)) {
				if ( (n = Read(sockfd, buf, MAXLINE)) == 0) {
						/*4connection closed by client */
					Close(sockfd);
					FD_CLR(sockfd, &allset);
					client[i] = -1;
				} else
					Writen(sockfd, buf, n);

				if (--nready <= 0)
					break;				/* no more readable descriptors */
			}
		}
	}
}
```
#### 6. poll 函数
描述：poll 提供的功能跟 select 类似，不过在处理流设备时能提供额外的信息
原型：

```
#include <poll.h>

int poll(struct pollfd *fdarray, unsigned long nfds, int timeout);
                                返回：若有就绪描述符则为其数目，若超时则为 0， 若出错则为-1
```
说明：  
1. pollfd 结构：  
    ```
    struct pollfd{
        int     fd;
        short   events; // 感兴趣的描述符
        short   revents; // 返回的描述符状态
    }
    ```
2. poll 函数的输入 events 和返回的 revents  
// 图6-23
    - 所有正规 TCP 和 UDP 数据都是普通数据
    - TCP 带外数据是优先级数据
    - TCP 读半关闭时(收到了对端的 FIN)，也是普通数据
    - TCP 存在错误可认为是普通数据也可认为是错误(POLLERR)
    - 监听套接字有新连接可认为是普通数据，也可认为是优先级数据，大多认为普通数据
    - 非阻塞式 connect 完成认为是相应套接字可写
3. nfds  
结构数组中元素数目
4. timeout 指定 poll 等待多长时间   

timeout 值 | 说明
---|---
INFTIM | 永远等待
0 | 立刻返回，不阻塞进程
>0 | 等待指定数目的毫秒数
#### 7. TCP 服务器程序(poll 版)

```
/* include fig01 */
#include	"unp.h"
#include	<limits.h>		/* for OPEN_MAX */

int
main(int argc, char **argv)
{
	int					i, maxi, listenfd, connfd, sockfd;
	int					nready;
	ssize_t				n;
	char				buf[MAXLINE];
	socklen_t			clilen;
	struct pollfd		client[OPEN_MAX];
	struct sockaddr_in	cliaddr, servaddr;

	listenfd = Socket(AF_INET, SOCK_STREAM, 0);

	bzero(&servaddr, sizeof(servaddr));
	servaddr.sin_family      = AF_INET;
	servaddr.sin_addr.s_addr = htonl(INADDR_ANY);
	servaddr.sin_port        = htons(SERV_PORT);

	Bind(listenfd, (SA *) &servaddr, sizeof(servaddr));

	Listen(listenfd, LISTENQ);

	client[0].fd = listenfd;
	client[0].events = POLLRDNORM;
	for (i = 1; i < OPEN_MAX; i++)
		client[i].fd = -1;		/* -1 indicates available entry */
	maxi = 0;					/* max index into client[] array */

	for ( ; ; ) {
		nready = Poll(client, maxi+1, INFTIM);

		if (client[0].revents & POLLRDNORM) {	/* new client connection */
			clilen = sizeof(cliaddr);
			connfd = Accept(listenfd, (SA *) &cliaddr, &clilen);
			
			for (i = 1; i < OPEN_MAX; i++)
				if (client[i].fd < 0) {
					client[i].fd = connfd;	/* save descriptor */
					break;
				}
			if (i == OPEN_MAX)
				err_quit("too many clients");

			client[i].events = POLLRDNORM;
			if (i > maxi)
				maxi = i;				/* max index in client[] array */

			if (--nready <= 0)
				continue;				/* no more readable descriptors */
		}

		for (i = 1; i <= maxi; i++) {	/* check all clients for data */
			if ( (sockfd = client[i].fd) < 0)
				continue;
			if (client[i].revents & (POLLRDNORM | POLLERR)) {
				if ( (n = read(sockfd, buf, MAXLINE)) < 0) {
					if (errno == ECONNRESET) {
							/*4connection reset by client */

						Close(sockfd);
						client[i].fd = -1;
					} else
						err_sys("read error");
				} else if (n == 0) {
						/*4connection closed by client */

					Close(sockfd);
					client[i].fd = -1;
				} else
					Writen(sockfd, buf, n);

				if (--nready <= 0)
					break;				/* no more readable descriptors */
			}
		}
	}
}
```










