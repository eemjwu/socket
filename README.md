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








