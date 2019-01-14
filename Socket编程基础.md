<!-- vscode-markdown-toc -->
* [一、套接字编程简介](#)
	* [1. 套接字地址结构](#-1)
		* [1. IPv4套接字地址结构](#IPv4)
		* [2. IPv6套接字地址结构](#IPv6)
		* [3. 通用套接字地址结构](#-1)
	* [2. Socket编程API](#SocketAPI)
		* [1. 字节序转换函数 htonl 和 ntohl](#htonlntohl)
		* [2. 字节操纵函数 bzero 等](#bzero)
		* [3. 地址转换函数 inet_pton 和 inet_ntop](#inet_ptoninet_ntop)
		* [4. sock_ntop和相关函数](#sock_ntop)
		* [5. readn 、 writen、 readline函数](#readnwritenreadline)
* [二、基本TCP套接字编程](#TCP)
	* [1. socket 函数](#socket)
	* [2. connect函数](#connect)
	* [3. bind 函数](#bind)
	* [4. listen 函数](#listen)
	* [5. accept 函数](#accept)
	* [6. fork 函数](#fork)
	* [7. exec 函数](#exec)
	* [8. close 函数](#close)
	* [9. getsockname 函数和 getpeername 函数](#getsocknamegetpeername)
* [ 三、TCP 客户端/服务器程序范式](#TCP-1)
	* [1. 其他函数准备](#-1)
		* [1. TCP 回射服务器程序： str_echo 函数](#TCPstr_echo)
		* [2. TCP 回射客户端程序： str_cli 函数](#TCPstr_cli)
	* [2. TCP 客户端/服务器设计范式](#TCP-1)
		* [1. 迭代式](#-1)
		* [2. select I/O复用式](#selectIO)
		* [3. 非阻塞式 I/O](#IO)
		* [4. 使用 fork 版的 str_cli 函数](#forkstr_cli)
		* [5. 线程版](#-1)
		* [7. 进程池版服务器(accept 无上锁保护)](#accept-1)
		* [8. 进程池版服务器(accept 使用文件上锁保护)](#accept-1)
		* [9. 进程池版服务器(accept 使用线程上锁保护)](#accept-1)
		* [10. 进程池版服务器(传递描述符)](#-1)
		* [11. 线程池版服务器(每个线程各自 accept)](#accept-1)
		* [12. 线程池版服务器(主线程统一 accept)](#accept-1)

<!-- vscode-markdown-toc-config
	numbering=false
	autoSave=true
	/vscode-markdown-toc-config -->
<!-- /vscode-markdown-toc -->
# 前言
本文从套接字API的介绍到客户端、服务器的各种范式编写，整理Socket编程的基础。  
参考书籍：  
- 《UNIX网络编程 卷1：套接字联网API》
## <a name=''></a>一、套接字编程简介
### <a name='-1'></a>1. 套接字地址结构
每个协议族有自己的套接字地址结构。名字均以sockaddr_开头，以每个协议族唯一后缀结束。  
本文使用到的数据类型说明：  
![数据类型说明](https://raw.githubusercontent.com/eemjwu/socket/master/%E5%9B%BE%E7%89%87/%E6%95%B0%E6%8D%AE%E7%B1%BB%E5%9E%8B%E8%AF%B4%E6%98%8E.bmp)
#### <a name='IPv4'></a>1. IPv4套接字地址结构
**名字**： struct sockaddr_in{ }    
**结构**：  
![image](https://raw.githubusercontent.com/eemjwu/socket/master/%E5%9B%BE%E7%89%87/IPv4%E5%A5%97%E6%8E%A5%E5%AD%97%E5%9C%B0%E5%9D%80.bmp)
#### <a name='IPv6'></a>2. IPv6套接字地址结构
**名字**：struct sockaddr_in6{ }  
**结构**：
![image](https://raw.githubusercontent.com/eemjwu/socket/master/%E5%9B%BE%E7%89%87/IPv6%E5%A5%97%E6%8E%A5%E5%AD%97%E5%9C%B0%E5%9D%80%E7%BB%93%E6%9E%84.bmp)
#### <a name='-1'></a>3. 通用套接字地址结构
**名字**：struct sockaddr{ }  
**结构**：
![image](https://raw.githubusercontent.com/eemjwu/socket/master/%E5%9B%BE%E7%89%87/%E9%80%9A%E7%94%A8%E5%A5%97%E6%8E%A5%E5%AD%97%E5%9C%B0%E5%9D%80%E7%BB%93%E6%9E%84.bmp)
### <a name='SocketAPI'></a>2. Socket编程API
#### <a name='htonlntohl'></a>1. 字节序转换函数 htonl 和 ntohl
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
#### <a name='bzero'></a>2. 字节操纵函数 bzero 等

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
#### <a name='inet_ptoninet_ntop'></a>3. 地址转换函数 inet_pton 和 inet_ntop

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
#### <a name='sock_ntop'></a>4. sock_ntop和相关函数
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

#### <a name='readnwritenreadline'></a>5. readn 、 writen、 readline函数
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
## <a name='TCP'></a>二、基本TCP套接字编程
![4-1](https://raw.githubusercontent.com/eemjwu/socket/master/%E5%9B%BE%E7%89%87/4-1.jpg)
### <a name='socket'></a>1. socket 函数
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
### <a name='connect'></a>2. connect函数
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
### <a name='bind'></a>3. bind 函数
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
### <a name='listen'></a>4. listen 函数
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
### <a name='accept'></a>5. accept 函数
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
### <a name='fork'></a>6. fork 函数
描述：fork 产生新进程的唯一方法；父子进程共享**fork之前打开的**描述符，描述符引用计数 +1  
原型：

```
#include <unistd.h>

pid_t fork(void);
            返回：子进程中为 0，父进程为子进程 ID，出错 -1
```
### <a name='exec'></a>7. exec 函数
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
### <a name='close'></a>8. close 函数
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
### <a name='getsocknamegetpeername'></a>9. getsockname 函数和 getpeername 函数
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
## <a name='TCP-1'></a> 三、TCP 客户端/服务器程序范式
编写一个完整的 TCP 客户端/服务器程序：  
1. 客户从标准输入读入一行文本，并写给服务器；
2. 服务器从网络输入读入这行文本，并回射给客户；
3. 客户从网络输入读入这行回射文本，并显示在标准输出上  
// 图5-1

### <a name='-1'></a>1. 其他函数准备
#### <a name='TCPstr_echo'></a>1. TCP 回射服务器程序： str_echo 函数  
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
#### <a name='TCPstr_cli'></a>2. TCP 回射客户端程序： str_cli 函数
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
### <a name='TCP-1'></a>2. TCP 客户端/服务器设计范式
#### <a name='-1'></a>1. 迭代式 
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
###### 僵尸进程与处理
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
###### accept 中必须处理中断
1. 当 SIGCHLD 信号递交时，父进程阻塞于 accept 系统调用，内核会使 accept 返回一个 EINTR 错误( 被中断的系统调用 )，**所以必须在程序 accept 中处理该错误**，重新启动 accept 系统调用。  
2. 对于 accept 以及诸如read、write、select 和 open 之类的函数来说，**重启**被中断的系统调用时合适的，**但是 connect 除外**，重启 connect 将返回错误，必须调用 select 来等待连接完成，后文有解释。
###### 总结
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
#### <a name='selectIO'></a>2. select I/O复用式
##### 基本 I/O 模型
###### 阻塞式 I/O 模型
进程调用某系统调用，直到任务完成，或者出现错误时，该系统才返回，称进程在系统调用期间是阻塞的。  
// 图6-1
###### 非阻塞式 I/O 模型
与阻塞式相反，当系统调用没出现期望的条件时，进程不进入睡眠而是立刻返回一个错误。  
// 图6-2
###### I/O 复用模型
调用 select 或 poll 阻塞在这两个系统调用上，而不是阻塞在真正的 I/O 系统调用上。使用 select 的优势在于可以等待多个描述符就绪。  
// 图6-3
###### 信号驱动式 I/O 模型
内核在描述符就绪时发送 SIGIO 信号。首先开启套接字信号驱动功能，并通过 sigaction 安装一个信号处理函数，系统调用立即返回，不阻塞。  
// 图6-4
###### 异步 I/O 模型
告知内核启动某个操作，并让内核在**整个操作完成**之后通知进程。与信号驱动模型的区别：信号驱动 I/O是告知内核何时可以启动一个 I/O ，异步模型是由内核通知我们 I/O 操作何时**完成**。  
// 图6-5
###### 各种 I/O 模型比较
阻塞式 I/O 模型、非阻塞式 I/O 模型、I/O 复用模型和信号驱动式 I/O 模型都是同步 I/O 模型；因为其真正的 I/O 操作将阻塞进程。    
// 图6-5
##### select 函数
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
##### shutdown 函数
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
##### str_cli 函数
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
##### TCP 服务器程序(select 版)
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
##### poll 函数
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
##### TCP 服务器程序(poll 版)

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
#### <a name='IO'></a>3. 非阻塞式 I/O
##### 非阻塞读和写：str_cli 函数
###### 描述 
，select 通常结合非阻塞 I/O 一起使用，以便判断何时可读可写。避免使用标准 I/O ，维护自己的缓冲区：  
1. 
// 图16-1
2. 
// 图16-2
###### 源码
// 链接 非阻塞 str_cli
###### 说明
见《UNP》p343 to p345
###### 问题
非阻塞式用时最少效率最高，但是编程太复杂，并不实用，使用效率稍逊，更简单的 fork 版。 
#### <a name='forkstr_cli'></a>4. 使用 fork 版的 str_cli 函数
##### 描述
使用 fork 把当前进程划分成两个进程。子进程把来自服务器的文本行复制到标准输出，父进程把来自标准输入的文本复制到服务器。  
// 图16-9
##### 源码

```
#include	"unp.h"

void
str_cli(FILE *fp, int sockfd)
{
	pid_t	pid;
	char	sendline[MAXLINE], recvline[MAXLINE];

	if ( (pid = Fork()) == 0) {		/* child: server -> stdout */
		while (Readline(sockfd, recvline, MAXLINE) > 0)
			Fputs(recvline, stdout);

		kill(getppid(), SIGTERM);	/* in case parent still running */
		exit(0);
	}

		/* parent: stdin -> server */
	while (Fgets(sendline, MAXLINE, fp) != NULL)
		Writen(sockfd, sendline, strlen(sendline));

	Shutdown(sockfd, SHUT_WR);	/* EOF on stdin, send FIN */
	pause();
	return;
}

```
##### 说明
如图，父子进程共享一个套接字，一个往里面写一个从里面读。尽管套接字只有一个，其接收缓冲区和发送缓冲区分别只有一个，然而这个套接字却有两个描述符在引用它。  
1. 终止序列问题
   1. 正常终止：在标准输入遇到 EOF 时发生，此时父进程调用 shutdown 发送 FIN （父进程不能调用 close ），父进程进入睡眠，但是这发生之后子进程继续从服务器读数据，显示到标准输出，直到遇见 EOF，发送 SIGTERM 给父进程，父进程默认终止。
   2. 服务器进程过早终止：子进程在套接字上读到 EOF，这时子进程必须告诉父进程停止将数据写入套接字发送给服务器，于是发送一个 SIGTERM 信号，以防父进程仍然运行。(另一个方法：子进程正常终止，父进程捕获 SIGCHLD 信号)
#### <a name='-1'></a>5. 线程版
fork 产生进程代价高，创建线程比创建进程快 10~100 倍；父子进程之间传递消息需要用到进程间通信机制，同一线程共享全局内存，是的线程之间易于共享信息，但却带来同步问题。
##### pthread_create 函数
描述：  
创建线程   
原型：  

```
#include<pthread.h>

int pthread_create(pthread_t *tid,const pthread_attr_t *attr, void *(*func)(void *),void *arg);
                                                        返回：成功 0，失败 error 值
```
说明：   
参数：tid 返回创建成功的线程 ID 号
参数：attr 指定创建线程的属性，默认传入 NULL
参数：func 指定新线程执行的函数
参数：arg  指定线程执行函数的参数
##### 基本线程函数
###### pthread_join 函数
描述：  
等待一个给定的线程终止，对比 UNIX 进程，pthread_create 类似于 fork，pthread_join 类似于 waitpid   
原型：  

```
#include <pthread.h>

int pthread_join(pthread_t *pid, void **status);
                        返回：成功 0， 出错正的 error 值
```
说明：    
必须指定等待的线程 tid，Pthread 没办法等待任意线程   
如果 status 非空，来自所等待进程的返回值存入由 status 指向的位置
###### pthread_self 函数
描述：  
获取自身的线程 ID
原型：   

```
#include <pthread.h>

pthread_t pthread_self(void);
                返回：调用线程的 ID
```
###### pthread_detach 函数
描述：   
使用 pthread_detach 函数把指定的线程转变为脱离状态   
原型：    

```
#incldue <pthread.h>

int pthread_detach(pthread_t tid);
                    返回：成功 0 ，失败 正的 error 值
```
说明：   
一个线程默认是可汇合的，或者是可脱离(detached)的。   
当一个可汇合的线程终止时，它的线程 ID 和退出状态将留存到另一个线程对它调用 pthread_jion。   
脱离的线程却像守护进程，当它们终止时，所有相关资源都被释放，不能等待它终止
###### pthread_exit 函数
描述：   
终止线程   
原型： 

```
#include <pthread.h>

void pthread_exit(void *status);
```
说明：   
如果本线程未曾脱离，它的线程 ID 和退出状态将一直保存到调用进程内的某个线程对它调用 pthread_join 函数  
指针 status 不能指向局部于调用线程的对象，因为线程终止时对象也消失   
让线程终止的另外两个方法：  
1. 启动线程的函数（pthread_create的第三个参数）返回。该函数返回值就是相应线程的终止状态
2. 进程的 main 函数返回，或者**任何线程**调用了 exit，**整个进程就终止**，包括它的任何线程
##### 使用线程的 str_cli 程序


描述：   
// 图26-1    
源码：  

```
#include	"unpthread.h"

void	*copyto(void *);

static int	sockfd;		/* global for both threads to access */
static FILE	*fp;

void
str_cli(FILE *fp_arg, int sockfd_arg)
{
	char		recvline[MAXLINE];
	pthread_t	tid;

	sockfd = sockfd_arg;	/* copy arguments to externals */
	fp = fp_arg;

	Pthread_create(&tid, NULL, copyto, NULL);

	while (Readline(sockfd, recvline, MAXLINE) > 0)
		Fputs(recvline, stdout);
}

void *
copyto(void *arg)
{
	char	sendline[MAXLINE];

	while (Fgets(sendline, MAXLINE, fp) != NULL)
		Writen(sockfd, sendline, strlen(sendline));

	Shutdown(sockfd, SHUT_WR);	/* EOF on stdin, send FIN */

	return(NULL);
		/* 4return (i.e., thread terminates) when EOF on stdin */
}

```
##### 使用线程的 TCP 回射服务器程序
源码：  

```
#include	"unpthread.h"

static void	*doit(void *);		/* each thread executes this function */

int
main(int argc, char **argv)
{
	int				listenfd, connfd;
	pthread_t		tid;
	socklen_t		addrlen, len;
	struct sockaddr	*cliaddr;

	if (argc == 2)
		listenfd = Tcp_listen(NULL, argv[1], &addrlen);
	else if (argc == 3)
		listenfd = Tcp_listen(argv[1], argv[2], &addrlen);
	else
		err_quit("usage: tcpserv01 [ <host> ] <service or port>");

	cliaddr = Malloc(addrlen);

	for ( ; ; ) {
		len = addrlen;
		connfd = Accept(listenfd, cliaddr, &len);
		Pthread_create(&tid, NULL, &doit, (void *) connfd);
	}
}

static void *
doit(void *arg)
{
	Pthread_detach(pthread_self());
	str_echo((int) arg);	/* same function as before */
	Close((int) arg);		/* done with connected socket */
	return(NULL);
}

```
说明：    
1. 子线程让自己变为脱离，因为主线程没有理由等待它创建的每个线程
2. 主线程不关闭已连接套接字，因为创建新线程并不影响已打开描述符的引用计数，不同于 fork
更具移植性的版本：   

```
#include	"unpthread.h"

static void	*doit(void *);		/* each thread executes this function */

int
main(int argc, char **argv)
{
	int				listenfd, *iptr;
	thread_t		tid;
	socklen_t		addrlen, len;
	struct sockaddr	*cliaddr;

	if (argc == 2)
		listenfd = Tcp_listen(NULL, argv[1], &addrlen);
	else if (argc == 3)
		listenfd = Tcp_listen(argv[1], argv[2], &addrlen);
	else
		err_quit("usage: tcpserv01 [ <host> ] <service or port>");

	cliaddr = Malloc(addrlen);

	for ( ; ; ) {
		len = addrlen;
		iptr = Malloc(sizeof(int));               //
		*iptr = Accept(listenfd, cliaddr, &len);  // 每次分配一个单独的空间 
		Pthread_create(&tid, NULL, &doit, iptr);
	}
}

static void *
doit(void *arg)
{
	int		connfd;

	connfd = *((int *) arg);
	free(arg);

	Pthread_detach(pthread_self());
	str_echo(connfd);		/* same function as before */
	Close(connfd);			/* done with connected socket */
	return(NULL);
}

```
##### 线程同步
###### 互斥锁
原型：

```
#include <pthread.h>

int pthread_mutex_lock(pthread_mutex_t *mptr);

int pthread_mutex_unlock(pthread_mutex_t *mptr);
                        返回：成功 0，失败 值为正的 error
```
使用：  
某个互斥锁变量是静态分配的(栈中)，就必须把它初始化为常量 PTHREAD_MUTEX_INITIALIZER   
如果在共享内存区中分配一个互斥锁，必须通过调用 pthread_mutex_init 函数在运行时初始化   
用法实例： 

```
#include	"unpthread.h"

#define	NLOOP 5000

int				counter;		/* incremented by threads */
pthread_mutex_t	counter_mutex = PTHREAD_MUTEX_INITIALIZER;

void	*doit(void *);

int
main(int argc, char **argv)
{
	pthread_t	tidA, tidB;

	Pthread_create(&tidA, NULL, &doit, NULL);
	Pthread_create(&tidB, NULL, &doit, NULL);

		/* 4wait for both threads to terminate */
	Pthread_join(tidA, NULL);
	Pthread_join(tidB, NULL);

	exit(0);
}

void *
doit(void *vptr)
{
	int		i, val;

	/*
	 * Each thread fetches, prints, and increments the counter NLOOP times.
	 * The value of the counter should increase monotonically.
	 */

	for (i = 0; i < NLOOP; i++) {
		Pthread_mutex_lock(&counter_mutex);  // 加锁

		val = counter;
		printf("%d: %d\n", pthread_self(), val + 1);
		counter = val + 1;

		Pthread_mutex_unlock(&counter_mutex); // 解锁
	}

	return(NULL);
}

```
###### 条件变量
描述：   
互斥锁：适合于防止同时访问某个共享变量；  
条件变量：在等待某个条件发生期间让线程进入睡眠。   
互斥锁提供互斥机制，条件变量提供信号机制。   
原型：

```
#include <pthread.h>

int pthread_cond_wait(pthread_cond_t *cptr, pthread_mutex_t *mptr);

int pthread_cond_signal(pthred_cond_t *cptr);
                            返回： 成功 0；失败 正的 error
```
说明：   
1. pthread_cond_wait 被调用前所关联的互斥锁必须是上锁的，该函数是一个原子操作：解锁该互斥锁 + 把线程投入睡眠；当被唤醒时该线程继续持有该互斥锁 
2. 条件变量必须关联一个互斥锁，因为 “条件” 通常是线程之间共享某个变量的值。允许不同线程设置和测试该变量，要求有一个与该变量关联的互斥锁

使用实例：  

```
int                     ndone;
pthread_mutex_t         ndone_mutex = PTHREAD_MUTEX_INITIALIZER;
pthread_cont_t          ndone_cond  = PTHREAD_COND_INITIALIZER;

通过在持有该互斥锁期间递增该值，并发送信号到该条件变量，一个线程通知主循环自身即将终止

Pthread_mutex_lock(&ndone_mutex);
ndone++;
Pthread_cond_signal(&ndone_cond);
Pthread_mutex_unlock(&ndone_mutex);

----------------------------------------------------------------------------------
主循环阻塞在 pthread_cond_wait 调用中，等待某个即将终止的线程发送信号到与 ndone 相关联的条件变量

while(nlefttoread > 0)
{
    while(nconn < maxnconn && nefttoconn > 0)
    {
        /* 找一个文件读 */
    }
    
    /* 等待条件变量 */
    
    Pthread_mutex_lock(&ndone_mutex); // 持有锁
    while(ndone == 0)
        Pthread_cond_wait(&ndone_cond,&ndone_mutex);
    // 条件等到了
    
    // do eles
    
    Pthread_mutex_unlock(&ndone_mutex);
}

```
条件变量其他函数
```
#include <pthread.h>

// 同时唤醒相同条件变量上的多个线程  
int pthread_cond_broadcast(pthread_cond_t *cptr);

// 设置等待时限
int pthread_cond_timedwait(pthread_cond_t *cptr,pthread_mutex_t *mptr,const struct timespec *abstime);
                                                    返回：成功 1，失败 正值的 error
```
#### <a name='accept-1'></a>7. 进程池版服务器(accept 无上锁保护)
源码：  
```
/* include serv02 */
#include	"unp.h"

static int		nchildren;
static pid_t	*pids;

int
main(int argc, char **argv)
{
	int			listenfd, i;
	socklen_t	addrlen;
	void		sig_int(int);
	pid_t		child_make(int, int, int);

	if (argc == 3)
		listenfd = Tcp_listen(NULL, argv[1], &addrlen);
	else if (argc == 4)
		listenfd = Tcp_listen(argv[1], argv[2], &addrlen);
	else
		err_quit("usage: serv02 [ <host> ] <port#> <#children>");
	nchildren = atoi(argv[argc-1]);
	pids = Calloc(nchildren, sizeof(pid_t));

	for (i = 0; i < nchildren; i++)
		pids[i] = child_make(i, listenfd, addrlen);	/* parent returns */

	Signal(SIGINT, sig_int);

	for ( ; ; )
		pause();	/* everything done by children */
}
/* end serv02 */

/* include sigint */
void
sig_int(int signo)
{
	int		i;
	void	pr_cpu_time(void);

		/* 4terminate all children */
	for (i = 0; i < nchildren; i++)
		kill(pids[i], SIGTERM);
	while (wait(NULL) > 0)		/* wait for all children */
		;
	if (errno != ECHILD)
		err_sys("wait error");

	pr_cpu_time();
	exit(0);
}
/* end sigint */

/* include child_make */
#include	"unp.h"

pid_t
child_make(int i, int listenfd, int addrlen)
{
	pid_t	pid;
	void	child_main(int, int, int);

	if ( (pid = Fork()) > 0)
		return(pid);		/* parent */

	child_main(i, listenfd, addrlen);	/* never returns */
}
/* end child_make */

/* include child_main */
void
child_main(int i, int listenfd, int addrlen)
{
	int				connfd;
	void			web_child(int);
	socklen_t		clilen;
	struct sockaddr	*cliaddr;

	cliaddr = Malloc(addrlen);

	printf("child %ld starting\n", (long) getpid());
	for ( ; ; ) {
		clilen = addrlen;
		connfd = Accept(listenfd, cliaddr, &clilen);

		web_child(connfd);		/* process the request */
		Close(connfd);
	}
}
/* end child_main */

```
说明：  
1. 会有惊群效应影响性能
   - 当有一个客户连接到来时，所有 N 个子进程都被唤醒，但是只有最先运行的那个子进程获得客户连接，其余恢复睡眠。 
2. select 冲突
   - 因为在socket结构中为存放本套接字就绪之时应该唤醒哪些进程而分配的仅仅是**一个进程ID空间**，如果多个进程等待同一个套接字，那么内核**将唤醒的是阻塞在select调用中的所有进程**，因为它不知道哪些进程受刚变得就绪的这个套接字影响。
   - 如果有多个进程阻塞在引用同一个实体（套接字，文件）的描述符上，那么最好直接阻塞在诸如 accept 之类函数，而不是 select 中
#### <a name='accept-1'></a>8. 进程池版服务器(accept 使用文件上锁保护)
描述：   
让应用进程在调用accept前后安置某种形式的锁，使任意时刻**只有一个子进程阻塞在accept调用**中，其他进程阻塞在锁上。    
说明：   
1. 使用 fcntl 函数实现 POSIX 文件锁功能
2. 源码与上一节比有两处改动：
    1. main 函数中，在派生子进程循环之前调用 my_lock_init() 初始化文件锁
    ```
    + my_lock_init("/tmp/lock.xxxx"); /* 初始化一个文件锁 */
    for (i = 0; i < nchildren; i++)
		pids[i] = child_make(i, listenfd, addrlen);	/* parent returns */
    ```
    2. child_main 函数中，在调用 accept 前后分别加锁和解锁
    
    ```
    for ( ; ; ) {
		clilen = addrlen;
	+   my_lock_wait();	
		connfd = Accept(listenfd, cliaddr, &clilen);
    +   my_lock_release();
		web_child(connfd);		/* process the request */
		Close(connfd);
	}
    ```
3. my_lock_initI()  my_lock_wait()  my_lock_release() 函数的实现
```
/* include my_lock_init */
#include	"unp.h"

static struct flock	lock_it, unlock_it;
static int			lock_fd = -1;
					/* fcntl() will fail if my_lock_init() not called */

void
my_lock_init(char *pathname)
{
    char	lock_file[1024];

		/* 4must copy caller's string, in case it's a constant */
    strncpy(lock_file, pathname, sizeof(lock_file));
    lock_fd = Mkstemp(lock_file);

    Unlink(lock_file);			/* but lock_fd remains open */

	lock_it.l_type = F_WRLCK;
	lock_it.l_whence = SEEK_SET;
	lock_it.l_start = 0;
	lock_it.l_len = 0;

	unlock_it.l_type = F_UNLCK;
	unlock_it.l_whence = SEEK_SET;
	unlock_it.l_start = 0;
	unlock_it.l_len = 0;
}
/* end my_lock_init */

/* include my_lock_wait */
void
my_lock_wait()
{
    int		rc;
    
    while ( (rc = fcntl(lock_fd, F_SETLKW, &lock_it)) < 0) {
		if (errno == EINTR)
			continue;
    	else
			err_sys("fcntl error for my_lock_wait");
	}
}

void
my_lock_release()
{
    if (fcntl(lock_fd, F_SETLKW, &unlock_it) < 0)
		err_sys("fcntl error for my_lock_release");
}
/* end my_lock_wait */

```
函数解析见 《UNP》 p660 - p661。
#### <a name='accept-1'></a>9. 进程池版服务器(accept 使用线程上锁保护)
描述：   
本节该用线程上锁保护，**最快**，既适用于同一进程中的线程，也适用于不同进程间上锁。上节使用文件锁，涉及文件系统操作耗时。
源码：    
1. main child_make child_main 都不需要改变，唯一需要改变的是上节的三个上锁函数
2. 不同进程使用线程上锁的要求：(1) 互斥锁变量必须放在由所有进程共享的存储区内；(2) 必须告知线程函数库这是在不同进程之间共享的互斥锁
3. my_lock_initI()  my_lock_wait()  my_lock_release() 函数的实现，使用 mmap 函数以及 /dev/zero 设备实现共享内存
```
/* include my_lock_init */
#include	"unpthread.h"
#include	<sys/mman.h>

static pthread_mutex_t	*mptr;	/* actual mutex will be in shared memory */

void
my_lock_init(char *pathname)
{
	int		fd;
	pthread_mutexattr_t	mattr;

	fd = Open("/dev/zero", O_RDWR, 0);

	mptr = Mmap(0, sizeof(pthread_mutex_t), PROT_READ | PROT_WRITE,
				MAP_SHARED, fd, 0);
	Close(fd);

	Pthread_mutexattr_init(&mattr);
	Pthread_mutexattr_setpshared(&mattr, PTHREAD_PROCESS_SHARED);
	Pthread_mutex_init(mptr, &mattr);
}
/* end my_lock_init */

/* include my_lock_wait */
void
my_lock_wait()
{
	Pthread_mutex_lock(mptr);
}

void
my_lock_release()
{
	Pthread_mutex_unlock(mptr);
}
/* end my_lock_wait */

```
#### <a name='-1'></a>10. 进程池版服务器(传递描述符)
描述：   
1. 本版本只让父进程调用 accept ，然后把所有接受的已连接套接字传递给某个子进程，绕过了为每个子进程 accept 上锁的需求，但是要传描述符，代码较复杂，因为父进程需要跟踪子进程忙闲状态，以便选择一个子进程传递描述符。  
2. 所以需要为每个子进程维护一个信息结构以便管理该进程状态
```
typedef struct {
  pid_t		child_pid;		/* process ID */
  int		child_pipefd;	/* 管道 parent's stream pipe to/from child */
  int		child_status;	/* 状态 0 = ready */
  long		child_count;	/* 已处理客户计数 # connections handled */
} Child;

Child	*cptr;		/* array of Child structures; calloc'ed */
```
源码：   

```
/* include serv05a */
#include	"unp.h"
#include	"child.h"

static int		nchildren;

int
main(int argc, char **argv)
{
	int			listenfd, i, navail, maxfd, nsel, connfd, rc;
	void		sig_int(int);
	pid_t		child_make(int, int, int);
	ssize_t		n;
	fd_set		rset, masterset;
	socklen_t	addrlen, clilen;
	struct sockaddr	*cliaddr;

	if (argc == 3)
		listenfd = Tcp_listen(NULL, argv[1], &addrlen);
	else if (argc == 4)
		listenfd = Tcp_listen(argv[1], argv[2], &addrlen);
	else
		err_quit("usage: serv05 [ <host> ] <port#> <#children>");

	FD_ZERO(&masterset);
	FD_SET(listenfd, &masterset);
	maxfd = listenfd;
	cliaddr = Malloc(addrlen);

	nchildren = atoi(argv[argc-1]);
	navail = nchildren;
	cptr = Calloc(nchildren, sizeof(Child));

		/* 4prefork all the children */
	for (i = 0; i < nchildren; i++) {
		child_make(i, listenfd, addrlen);	/* parent returns */
		FD_SET(cptr[i].child_pipefd, &masterset);
		maxfd = max(maxfd, cptr[i].child_pipefd);
	}

	Signal(SIGINT, sig_int);

	for ( ; ; ) {
		rset = masterset;
		if (navail <= 0)
			FD_CLR(listenfd, &rset);	/* turn off if no available children */
		nsel = Select(maxfd + 1, &rset, NULL, NULL, NULL);

			/* 4check for new connections */
		if (FD_ISSET(listenfd, &rset)) {
			clilen = addrlen;
			connfd = Accept(listenfd, cliaddr, &clilen);

			for (i = 0; i < nchildren; i++)
				if (cptr[i].child_status == 0)
					break;				/* available */

			if (i == nchildren)
				err_quit("no available children");
			cptr[i].child_status = 1;	/* mark child as busy */
			cptr[i].child_count++;
			navail--;

			n = Write_fd(cptr[i].child_pipefd, "", 1, connfd);
			Close(connfd);
			if (--nsel == 0)
				continue;	/* all done with select() results */
		}

			/* 4find any newly-available children */
		for (i = 0; i < nchildren; i++) {
			if (FD_ISSET(cptr[i].child_pipefd, &rset)) {
				if ( (n = Read(cptr[i].child_pipefd, &rc, 1)) == 0)
					err_quit("child %d terminated unexpectedly", i);
				cptr[i].child_status = 0;
				navail++;
				if (--nsel == 0)
					break;	/* all done with select() results */
			}
		}
	}
}
/* end serv05a */

void
sig_int(int signo)
{
	int		i;
	void	pr_cpu_time(void);

		/* 4terminate all children */
	for (i = 0; i < nchildren; i++)
		kill(cptr[i].child_pid, SIGTERM);
	while (wait(NULL) > 0)		/* wait for all children */
		;
	if (errno != ECHILD)
		err_sys("wait error");

	pr_cpu_time();

	for (i = 0; i < nchildren; i++)
		printf("child %d, %ld connections\n", i, cptr[i].child_count);

	exit(0);
}
/* include child_make */
#include	"unp.h"
#include	"child.h"

pid_t
child_make(int i, int listenfd, int addrlen)
{
	int		sockfd[2];
	pid_t	pid;
	void	child_main(int, int, int);

	Socketpair(AF_LOCAL, SOCK_STREAM, 0, sockfd);

	if ( (pid = Fork()) > 0) {
		Close(sockfd[1]);
		cptr[i].child_pid = pid;
		cptr[i].child_pipefd = sockfd[0];
		cptr[i].child_status = 0;
		return(pid);		/* parent */
	}

	Dup2(sockfd[1], STDERR_FILENO);		/* child's stream pipe to parent */
	Close(sockfd[0]);
	Close(sockfd[1]);
	Close(listenfd);					/* child does not need this open */
	child_main(i, listenfd, addrlen);	/* never returns */
}
/* end child_make */

/* include child_main */
void
child_main(int i, int listenfd, int addrlen)
{
	char			c;
	int				connfd;
	ssize_t			n;
	void			web_child(int);

	printf("child %ld starting\n", (long) getpid());
	for ( ; ; ) {
		if ( (n = Read_fd(STDERR_FILENO, &c, 1, &connfd)) == 0)
			err_quit("read_fd returned 0");
		if (connfd < 0)
			err_quit("no descriptor from read_fd");

		web_child(connfd);				/* process request */
		Close(connfd);

		Write(STDERR_FILENO, "", 1);	/* tell parent we're ready again */
	}
}
/* end child_main */

```
说明：   
1. 父子进程之间关系如图
// 图30-23
2. 本版本相比其他版变化在于：分配描述符集，打开与监听套接字以及各个子进程管道对应得位，计算最大描述符值，分配 Child 结构数组的内存空间，主循环由一个 select 驱动。
3. 详细说明见 《UNP》 p666 - p667
#### <a name='accept-1'></a>11. 线程池版服务器(每个线程各自 accept)
描述：   
1. 本版本是所有版本中最快的，使用互斥锁对 每个线程的 accept 加锁。
2. 定义一个用于维护每个线程信息的结构（用于实验统计数据）
```
typedef struct {
  pthread_t		thread_tid;		/* thread ID */
  long			thread_count;	/* # connections handled */
} Thread;
Thread	*tptr;		/* array of Thread structures; calloc'ed */

int				listenfd, nthreads;
socklen_t		addrlen;
pthread_mutex_t	mlock;

```

源码：   

```
/* include serv07 */
#include	"unpthread.h"
#include	"pthread07.h"

pthread_mutex_t	mlock = PTHREAD_MUTEX_INITIALIZER;

int
main(int argc, char **argv)
{
	int		i;
	void	sig_int(int), thread_make(int);

	if (argc == 3)
		listenfd = Tcp_listen(NULL, argv[1], &addrlen);
	else if (argc == 4)
		listenfd = Tcp_listen(argv[1], argv[2], &addrlen);
	else
		err_quit("usage: serv07 [ <host> ] <port#> <#threads>");
	nthreads = atoi(argv[argc-1]);
	tptr = Calloc(nthreads, sizeof(Thread));

	for (i = 0; i < nthreads; i++)
		thread_make(i);			/* only main thread returns */

	Signal(SIGINT, sig_int);

	for ( ; ; )
		pause();	/* everything done by threads */
}
/* end serv07 */

void
sig_int(int signo)
{
	int		i;
	void	pr_cpu_time(void);

	pr_cpu_time();

	for (i = 0; i < nthreads; i++)
		printf("thread %d, %ld connections\n", i, tptr[i].thread_count);

	exit(0);
}

#include	"unpthread.h"
#include	"pthread07.h"

void
thread_make(int i)
{
	void	*thread_main(void *);

	Pthread_create(&tptr[i].thread_tid, NULL, &thread_main, (void *) i);
	return;		/* main thread returns */
}

void *
thread_main(void *arg)
{
	int				connfd;
	void			web_child(int);
	socklen_t		clilen;
	struct sockaddr	*cliaddr;

	cliaddr = Malloc(addrlen);

	printf("thread %d starting\n", (int) arg);
	for ( ; ; ) {
		clilen = addrlen;
    	Pthread_mutex_lock(&mlock);
		connfd = Accept(listenfd, cliaddr, &clilen);
		Pthread_mutex_unlock(&mlock);
		tptr[(int) arg].thread_count++;

		web_child(connfd);		/* process request */
		Close(connfd);
	}
}

```
#### <a name='accept-1'></a>12. 线程池版服务器(主线程统一 accept)
描述：  
1. 启动阶段创建一个线程池之后只让主线程调用 accept 并把每个客户连接传递给池中某个可用线程，类似进程传递描述符版。
2. 因为同一进程内所有线程共享描述符，所以不用传递，只需要知道这个描述符的值
3. 维护一个与上一节等同的 Thread 结构管理线程信息，同时定义一个 clifd 数组，由主线程往中存入已接收的已连接套接字描述符，并由线程池中可用线程取出一个以服务相应的客户，iput 是主线程将往该数组中存的下一个元素的下标， iget 是线程池中某个线程从该数组中取下一个元素的下标。该共享结构由互斥锁和条件变量保护。

```
typedef struct {
  pthread_t		thread_tid;		/* thread ID */
  long			thread_count;	/* # connections handled */
} Thread;
Thread	*tptr;		/* array of Thread structures; calloc'ed */

#define	MAXNCLI	32
int					clifd[MAXNCLI], iget, iput;
pthread_mutex_t		clifd_mutex;
pthread_cond_t		clifd_cond;

```
源码:    
```
/* include serv08 */
#include	"unpthread.h"
#include	"pthread08.h"

static int			nthreads;
pthread_mutex_t		clifd_mutex = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t		clifd_cond = PTHREAD_COND_INITIALIZER;

int
main(int argc, char **argv)
{
	int			i, listenfd, connfd;
	void		sig_int(int), thread_make(int);
	socklen_t	addrlen, clilen;
	struct sockaddr	*cliaddr;

	if (argc == 3)
		listenfd = Tcp_listen(NULL, argv[1], &addrlen);
	else if (argc == 4)
		listenfd = Tcp_listen(argv[1], argv[2], &addrlen);
	else
		err_quit("usage: serv08 [ <host> ] <port#> <#threads>");
	cliaddr = Malloc(addrlen);

	nthreads = atoi(argv[argc-1]);
	tptr = Calloc(nthreads, sizeof(Thread));
	iget = iput = 0;

		/* 4create all the threads */
	for (i = 0; i < nthreads; i++)
		thread_make(i);		/* only main thread returns */

	Signal(SIGINT, sig_int);

	for ( ; ; ) {
		clilen = addrlen;
		connfd = Accept(listenfd, cliaddr, &clilen);

		Pthread_mutex_lock(&clifd_mutex);
		clifd[iput] = connfd;
		if (++iput == MAXNCLI)
			iput = 0;
		if (iput == iget)
			err_quit("iput = iget = %d", iput);
		Pthread_cond_signal(&clifd_cond);
		Pthread_mutex_unlock(&clifd_mutex);
	}
}
/* end serv08 */

void
sig_int(int signo)
{
	int		i;
	void	pr_cpu_time(void);

	pr_cpu_time();

	for (i = 0; i < nthreads; i++)
		printf("thread %d, %ld connections\n", i, tptr[i].thread_count);

	exit(0);
}

#include	"unpthread.h"
#include	"pthread08.h"

void
thread_make(int i)
{
	void	*thread_main(void *);

	Pthread_create(&tptr[i].thread_tid, NULL, &thread_main, (void *) i);
	return;		/* main thread returns */
}

void *
thread_main(void *arg)
{
	int		connfd;
	void	web_child(int);

	printf("thread %d starting\n", (int) arg);
	for ( ; ; ) {
    	Pthread_mutex_lock(&clifd_mutex);
		while (iget == iput)
			Pthread_cond_wait(&clifd_cond, &clifd_mutex);
		connfd = clifd[iget];	/* connected socket to service */
		if (++iget == MAXNCLI)
			iget = 0;
		Pthread_mutex_unlock(&clifd_mutex);
		tptr[(int) arg].thread_count++;

		web_child(connfd);		/* process request */
		Close(connfd);
	}
}

```








