---
layout: post
title: Socket TCP
category: 学习
tags: linux
keywords: linux
---

##使用TCP协议的流程  
{% highlight c %}
服务端：socket -> bind -> listen -> accept -> recv -> send -> close   
客户端：socket ------------------> connect -> send -> recv -> close
{% endhighlight %}

##1.socket  


{% highlight c %}
/* socket:生成一个套接口描述符  */
int socket(int domain, int type, int protocol);   
    //domain ---> AF_INET: IPv4; AF_INET6: IPv6;
    //type ---> SOCK_STREAM: tcp; SOCK_DGRAM: udp;
    //protocol ---> 指定socket传输所用的协议编号，通常为0 。
/*返回值：成功则返回套接口描述符，失败返回-1*/
{% endhighlight %}


##2.bind  

{% highlight c %}

/* bind:绑定一个端口号和ip地址，使套接口与端口号和ip地址相关联 */
int bind(int sockfd, struct sockaddr * my_addr, int addrlen);
    // sockfd为前面socket的返回值。
    // my_addr为结构体指针变量，结构体如下
    // addrlen:sockaddr的结构体长度。通常是sizeof(struct sockaddr);
/*返回值：成功则返回0，失败返回-1*/

{% endhighlight %}


{% highlight c %}

/*结构体*/
struct sockaddr  //此结构体不常用
{
    unsigned short int sa_family;  //调用socket()时的domain参数，即AF_INET值。
    char sa_data[14];  //最多使用14个字符长度
};

//使用ipv4时，其socketaddr结构定义便为
struct sockaddr_in  //常用的结构体
{
    unsigned short int sin_family;  //即为sa_family AF_INET
    uint16_t sin_port;  //为使用的port编号
    struct in_addr sin_addr;  //为IP 地址
    unsigned char sin_zero[8];  //未使用
};

struct in_addr
{
    uint32_t s_addr;
};
{% endhighlight %}
{% highlight c %}
/*bind实例*/
struct sockaddr_in my_addr;  //定义结构体变量
memset(&my_addr, 0, sizeof(struct sockaddr)); //将结构体清空
    //或bzero(&my_addr, sizeof(struct sockaddr));
my_addr.sin_family = AF_INET;  //表示采用Ipv4网络协议
my_addr.sin_port = htons(8888);  
    //表示端口号为8888，通常是大于1024的一个值。
    //htons()用来将参数指定的16位hostshort转换成网络字符顺序
my_addr.sin_addr.s_addr = inet_addr("192.168.1.13"); 
    // inet_addr()用来将IP地址字符串转换成网络所使用的二进制数字，如果为INADDR_ANY，这表示服务器自动填充本机IP地址。
int iret = bind(sfd, (struct sockaddr*)&my_str, sizeof(struct socketaddr));
if(iret == -1)
{
    perror("bind");
    close(sfd);
    exit(-1);
}
{% endhighlight %}
> **注：
my_addr.sin_port 置为 0，函数会自动为你选择一个未占用的端口来使用。
my_addr.sin_addr.s_addr 置为 INADDR_ANY，系统会自动填入本机IP地址。**


##3.listen  

{% highlight c %}
/* 使服务器的这个端口和IP处于监听状态，等待网络中某一客户机的连接请求。如果客户端有连接请求，端口就会接受这个连接 */
int listen(int sockfd, int backlog);
    //sockfd: socket描述符
    //backlog: 指定同时能处理的最大连接要求，通常为10或者5。 最大值可设至128
/*返回值：成功则返回0，失败返回-1*/
{% endhighlight %}

##4.accept  

{% highlight c %}
/*接受远程计算机的连接请求，建立起与客户机之间的通信连接*/
int accept(int sockfd, struct sockaddr * addr, int * addrlen);
    //sockfd: socket描述符
    //addr: 系统会把远程客户端主机的信息（地址和端口号信息）保存到这个指针所指的结构体中
    //addrlen: 表示结构体的长度，为整型指针
/*返回值：成功则返回新的客户端socket描述符new_fd，失败返回-1*/
{% endhighlight %}

##5.recv  

{% highlight c %}
/* 用新的套接字来接收远端主机传来的数据，并把数据存到由参数buf 指向的内存空间 */
int recv(int sockfd,void *buf,int len,unsigned int flags);
    //sockfd: socket描述符
    //buf: 接收的字符串存入缓冲区buf中
    //len: 缓冲区长度
    //flags: 通常为0
/*返回值：成功则返回实际接收到的字符数，可能会少于你所指定的接收长度。失败返回-1*/
{% endhighlight %}

##6.send  

{% highlight c %}
/* 发送消息到指定IP */
int send(int s,const void * msg,int len,unsigned int flags);
    //sockfd: socket描述符
    //msg: 一般为常量字符串，发出的消息
    //len: msg长度
    //flags: 通常为0
/*返回值：成功则返回实际传送出去的字符数，可能会少于你所指定的发送长度。失败返回-1*/
{% endhighlight %}

##7.close  

{% highlight c %}
close(sock_fd) ;
{% endhighlight %}


##8.connect       (in client) 

{% highlight c %}
/*用来请求连接远程服务器，将参数sockfd 的socket 连至参数serv_addr 指定的服务器IP和端口号上去*/
int connect(int sockfd,struct sockaddr * serv_addr,int addrlen);
    //sockfd: socket描述符
    //serv_addr: 结构体指针变量，存储着远程服务器的IP与端口号信息
    //addrlen: 结构体变量的长度
/*返回值：成功则返回0，失败返回-1*/
{% endhighlight %}
{% highlight c %}
/*connect实例*/
struct sockaddr_in seraddr;    //请求连接服务器
memset(&seraddr, 0, sizeof(struct sockaddr));
seraddr.sin_family = AF_INET;
seraddr.sin_port = htons(8888);    //服务器的端口号
seraddr.sin_addr.s_addr = inet_addr("192.168.0.101");  //服务器的ip
if(connect(sfd, (struct sockaddr*)&seraddr, sizeof(struct sockaddr)) == -1)
{
    perror("connect");
    close(sfd);
    exit(-1);
}
{% endhighlight %}

##实例
{% highlight c %}
/*tcp_server.c*/

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <netinet/in.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <unistd.h>
#include <sys/types.h>

int main(int argc, char* argv[])    //  ---> ./exe [port]
{
	//1.socket
	int fd_server ,fd_client;
	if(-1 == (fd_server= socket(AF_INET,SOCK_STREAM, 0)))
	{
		perror("socket");
		exit(-1);
	}

	//2.bind
	struct sockaddr_in server_addr ;
	memset(&server_addr, 0, sizeof(server_addr));
	server_addr.sin_family = AF_INET ;
	server_addr.sin_port = htons(atoi(argv[1]));
	server_addr.sin_addr.s_addr = INADDR_ANY ;
	if(0 != bind(fd_server, (struct sockaddr*)&server_addr, sizeof(server_addr)))
	{
		perror("bind");
		close(fd_server);
		exit(-1);
	}
	
	//3.listen
	if(-1 == listen(fd_server, 5))
	{
		perror("listen");
		close(fd_server);
		exit(-1);
	}
	
	//accept
	struct sockaddr_in client_addr ;
	memset(&client_addr, 0, sizeof(client_addr));
	int len = sizeof(client_addr);
	fd_client = accept(fd_server, (struct sockaddr*)&client_addr, &len);
	printf("ip: %s:%d connection establited !\n",inet_ntoa(client_addr.sin_addr), ntohs(client_addr.sin_port) );
	
	//4.recv
	char buf[128]  ;
	int iret = recv(fd_client, buf, 128, 0);
	buf[iret] = '\0';
	write(1, buf, strlen(buf));

	//5.send
	memset(buf, 0, 128);
	strcpy(buf, "I am the server, your msg has been received !\n");
	send(fd_client, buf, strlen(buf), 0);
	
	//6.close
	close(fd_server);
	close(fd_client);

	return 0 ;
}
{% endhighlight %}
{% highlight c %}
/*tcp_client.c*/

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <netinet/in.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <unistd.h>
#include <sys/types.h>

int main(int argc, char * argv[])    // ---> ./exe [ip] [port]
{
	//1.socket
	int fd_client ;
	fd_client = socket(AF_INET, SOCK_STREAM, 0);
	if(fd_client <= 0)
	{
		perror("socket");
		exit(-1);
	}

	//2.connect
	struct sockaddr_in server_addr ;
	memset(&server_addr, 0, sizeof(server_addr));
	server_addr.sin_family = AF_INET ;
	server_addr.sin_port = htons(atoi(argv[2]));
	server_addr.sin_addr.s_addr = inet_addr(argv[1]);
	if(-1 == connect(fd_client, (struct sockaddr*)&server_addr, sizeof(server_addr)))
	{
		perror("connect");
		close(fd_client);
		exit(-1);
	}

	//3.send
	send(fd_client, "Hello, I am the client !\n", 25, 0);

	//4.recv
	char buf[128] ="";
	int iret = recv(fd_client, buf, 128, 0);
	buf[iret] = 0 ;
	write(1, buf, strlen(buf));
	close(fd_client);

	return 0 ;
}
{% endhighlight %}

##基于TCP的文件传输 

{% highlight c %}
/*tcp_server.c*/

#include <stdio.h>
#include <stdlib.h>
#include <sys/socket.h>
#include <sys/types.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <netinet/in.h>
#include <string.h>
#include <fcntl.h>
#include <pthread.h>

#define SIZE 5*1024*1024    //设置发送缓冲区大小

void* sendfile(void * arg)    //线程负责发送文件
{
	int fd_file ;
	int fd_client = *(int*)arg ;
	char send_buf[SIZE] ;
	int readn ;
	fd_file = open("../../../../Downloads/VMware-Workstation-Full-10.0.1-1379776.x86_64.bundle", O_RDONLY) ;
	if(fd_file == -1)
	{
		perror("file open") ;
		exit(-1) ;
	}

	while(memset(send_buf, 0, SIZE) , (readn = read(fd_file, send_buf, SIZE)) > 0)
	{
		printf("read from file : %d\n", readn) ;    //一次读取的文件长度
		int sendn = send(fd_client, send_buf, readn, 0) ;    //send
		printf("send to client : %d\n", sendn) ;    //一次发送的文件长度
	}
	close(fd_file) ;
	close(fd_client) ;
	printf("send over !\n") ;    //发送完成
	pthread_exit(NULL) ;
}

int main(int argc, char * argv[])    // ---> ./exe [port]
{
	int fd_server ;
	pthread_t thd_send ;	
	fd_server = socket(AF_INET, SOCK_STREAM, 0) ;    //socket
	if(fd_server == -1)
	{
		perror("socket") ;
		exit(-1) ;
	}
	struct sockaddr_in server_addr ;
	memset(&server_addr, 0, sizeof(server_addr));
	server_addr.sin_family = AF_INET ;
	server_addr.sin_port = htons(atoi(argv[1])) ;    //设置服务器端口号
	server_addr.sin_addr.s_addr =  INADDR_ANY;
	if(-1 == bind(fd_server, (struct sockaddr*)&server_addr, sizeof(server_addr)))
	{
		perror("bind") ;
		close(fd_server) ;
		exit(-1) ;
	}
	listen(fd_server, 5) ;    //监听端口是否有新的访问
	
	while(1)
	{
		int fd_client = accept(fd_server, NULL, NULL) ;    //accept
		printf("a client connect !\n") ;
		pthread_create(&thd_send, NULL, sendfile, (void*)&fd_client) ;    //创建线程发送文件
	}
	close(fd_server) ;
	return 0 ;
}
{% endhighlight %}
{% highlight c %}
/*tcp_client.c*/

#include <stdio.h>
#include <stdlib.h>
#include <sys/socket.h>
#include <sys/types.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <netinet/in.h>
#include <string.h>
#include <fcntl.h>

#define SIZE 5*1024*1024    //设置接收缓冲区大小

int main(int argc, char * argv[])        // ---> ./exe [ip] [port]
{
	char recv_buf[SIZE] ;
	int fd_client ;
	fd_client = socket(AF_INET, SOCK_STREAM, 0) ;    //socket
	if(fd_client == -1)
	{
		perror("socket") ;
		exit(-1) ;
	}
	struct sockaddr_in server_addr ;
	memset(&server_addr, 0, sizeof(server_addr)) ;
	server_addr.sin_family = AF_INET ;
	server_addr.sin_port = htons(atoi(argv[2])) ;
	server_addr.sin_addr.s_addr = inet_addr(argv[1]) ;
	
	int iret = connect(fd_client, (struct sockaddr*)&server_addr, sizeof(server_addr)) ;    //connect
	if(iret == -1)
	{
		perror("connect") ;
		close(fd_client) ;
		exit(-1) ;
	}
	int fd_file = open("./a.out", O_WRONLY | O_CREAT, 0755) ;
	if(fd_file == -1)
	{
		perror("file open") ;
		exit(-1) ;
	}
	int nrecv ;
	while(memset(recv_buf, 0, SIZE), (nrecv = recv(fd_client, recv_buf, SIZE, 0)) > 0)    //recv
	{
		printf("recv from server : %d\n", nrecv) ;
		int nwrite = write(fd_file, recv_buf, nrecv) ;    //接收到的信息写入文件
		printf("write to file : %d\n", nwrite) ;
	}
	printf("Download over~\n") ;    //下载完成
	close(fd_file) ;
	close(fd_client) ;

	return 0 ;
}
{% endhighlight %}




