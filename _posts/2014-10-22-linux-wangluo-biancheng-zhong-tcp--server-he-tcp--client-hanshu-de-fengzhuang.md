---
layout: post
title: Linux网络编程中tcp_server和tcp_client函数的封装
category : C++
tagline: "Supporting tagline"
tags : [C++, Python, Socket, Tornado]
---
本文的主要目的是将server套接字和client套接字的获取，做一个简易的封装，使用C语言完成。
   
  
###tcp_server
   
  服务器端fd的获取主要分为以下几步：
     1.创建socket，这一步仅仅创建一个socket，没有任何特性的属性。
      2.绑定网卡和port，一块主机可能有多块网卡，如果我们使用INADDR_ANY，`意味着后面接受的TCP连接可以绑定在任意一块网卡上`。
    例如某台主机的ip地址有两个：192.168.44.136、10.1.1.4，假设绑定的ip采用INADDR_ANY，端口采用9981，那么当接收一个TCP连接时，可能存在192.168.44.136:9981/10.1.1.4:9981/127.0.0.1:9981三种可能性。
    如果我们绑定192.168.44.136，那么这个fd接收的连接都是在这个ip上。
    有一处特殊的地方，如果主机绑定的是127.0.0.1，那么客户端也必须是本机上的进程。（为什么？）
      3.监听端口listen，实际上从这一步开始接收TCP连接。`listen函数的第二个参数表示TCP待接受队列的大小`，这个队列存储了已经完成三次握手，但还未被accept的连接。如果你设置为3，然后开启4个client，但是server端不进行accept，通过netstat –an | grep 9981，可以查看到只有三条server端的连接处于ESTABLISHED状态，其余的一个处于SYN_RCVD状态。
    所以，很多人误以为accept完成三次握手，这是一种错误的想法，三次握手是由底层的协议栈完成的。
   然后我们需要使用几个套接字选项，后面会专门写一篇文章总结套接字选项：
  1.地址复用SO_REUSEADDR，代码如下：
  

```C++
//地址复用
void set_reuseaddr(int sockfd, int optval)
{
    int on = (optval != 0) ? 1 : 0;
    if (setsockopt(sockfd, SOL_SOCKET, SO_REUSEADDR, &on, sizeof(on)) < 0)
        ERR_EXIT("setsockopt SO_REUSEADDR");
}
```
		

2.端口复用SO_REUSEPORT，有的机器不支持，所以需要判断




```Python
void set_reuseport(int sockfd, int optval)
{
#ifdef SO_REUSEPORT
    int on = (optval != 0) ? 1 : 0;
    if (setsockopt(sockfd, SOL_SOCKET, SO_REUSEPORT, &on, sizeof(on)) < 0)
        ERR_EXIT("setsockopt SO_REUSEPORT");
#else
    fprintf(stderr, "SO_REUSEPORT is not supported.\n");
#endif //SO_REUSEPORT
}
```
		

3.禁用Nagle算法，降低局域网的延迟




```C++
//设置nagle算法是否可用
void set_tcpnodelay(int sockfd, int optval)
{
    int on = (optval != 0) ? 1 : 0;
    if(setsockopt(sockfd, IPPROTO_TCP, TCP_NODELAY, &on, sizeof(on)) == -1)
        ERR_EXIT("setsockopt TCP_NODELAY");
}
```
		

4.禁用keepAlive机制




```C++
void set_keepalive(int sockfd, int optval)
{
    int on = (optval != 0) ? 1 : 0;
    if(setsockopt(sockfd, SOL_SOCKET, SO_KEEPALIVE, &on, sizeof(on)) == -1)
        ERR_EXIT("setsockopt SO_KEEPALIVE");
}
```
		

 


我们还需要为我们的tcp_server增加域名解析功能，我们可以绑定”localhost”之类的主机名，而不仅仅是ip地址，所以我们可能使用gethostbyname。


所以tcp_server代码如下：




```C++
int tcp_server(const char *host, uint16_t port)
{

    //处理SIGPIPE信号
    handle_sigpipe();

    int listenfd = socket(PF_INET, SOCK_STREAM, 0);
    if(listenfd == -1)
        ERR_EXIT("socket");

    set_reuseaddr(listenfd, 1);
    set_reuseport(listenfd, 1);
    set_tcpnodelay(listenfd, 0);
    set_keepalive(listenfd, 0);

    SAI addr;
    memset(&addr, 0, sizeof addr);
    addr.sin_family = AF_INET;
    //addr.sin_addr.s_addr = inet_addr("127.0.0.1");
    addr.sin_port = htons(port);
    if(host == NULL)
    {
        addr.sin_addr.s_addr = INADDR_ANY;
    }
    else
    {
        //int inet_aton(const char *cp, struct in_addr *inp);
        if(inet_aton(host, &addr.sin_addr) == 0)
        {
            //DNS
            //struct hostent *gethostbyname(const char *name);
            struct hostent *hp = gethostbyname(host);
            if(hp == NULL)
                ERR_EXIT("gethostbyname");
            addr.sin_addr = *(struct in_addr*)hp->h_addr;
        }
    }

    if(bind(listenfd, (SA*)&addr, sizeof addr) == -1)
        ERR_EXIT("bind");

    if(listen(listenfd, SOMAXCONN) == -1)
        ERR_EXIT("listen");

    return listenfd;
}
```
		

 



###tcp_client


 


`客户端也可以绑定port`，只不过它不是必须的。代码较简单，如下：




```C++
int tcp_client(uint16_t port)
{
    int peerfd = socket(PF_INET, SOCK_STREAM, 0);
    if(peerfd == -1)
        ERR_EXIT("socket");

    set_reuseaddr(peerfd, 1);
    set_reuseport(peerfd, 1);
    set_keepalive(peerfd, 0);
    set_tcpnodelay(peerfd, 0);

    //如果port为0，则不去绑定
    if(port == 0)
        return peerfd;

    SAI addr;
    memset(&addr, 0, sizeof addr);
    addr.sin_family = AF_INET;
    addr.sin_port = htons(port);
    addr.sin_addr.s_addr = inet_addr(get_local_ip());
    if(bind(peerfd, (SA*)&addr, sizeof(addr)) == -1)
        ERR_EXIT("bind client");

    return peerfd;
}
```
		

上面需要获取本机的ip地址，代码如下：




```C++
const char *get_local_ip()
{
    static char ip[16];

    int sockfd;
    if((sockfd = socket(PF_INET, SOCK_STREAM, 0)) == -1)
    {
        ERR_EXIT("socket");
    }

    struct ifreq req;
    bzero(&req, sizeof(struct ifreq));
    strcpy(req.ifr_name, "eth0");
    if(ioctl(sockfd, SIOCGIFADDR, &req) == -1)
        ERR_EXIT("ioctl");

    struct sockaddr_in *host = (struct sockaddr_in*)&req.ifr_addr;
    strcpy(ip, inet_ntoa(host->sin_addr));
    close(sockfd);
    
    return ip;
}
```
		

 


完毕。

			