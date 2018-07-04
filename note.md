## 1.服务器

### 1）架构：

* **主进程**：监听来自客户端的连接
    - 监听端口：`port(4000)`
    - 监听地址：`通配地址(INADDR_ANY)`
    - 协议：`IPV4协议(AF_INET)`
    - 监听套接字描述符：`server_sock`
    - 已连接套接字描述符：`client_sock`
    - 客户端地址结构：`client_name`
    - 线程ID：`newthread`
* **线程**：每个线程处理一个新的连接

### 2）函数

* **主进程**
    - [main](#main函数)：调用**startup**进行监听，循环调用**accept**接受新连接，开启线程处理每个新连接
    - [startup](#startup函数)：根据监听端口号`port`，创建监听套接字描述符`server_sock`，调用**bind**和**listen**进行监听
* **线程**
    - **accept_request**：线程执行的函数。
* **错误处理**
    - [error_die](#errordie函数)：内部调用**perror**和**exit**



## 2.客户端

## 3.源码

### main函数

```c
int main(void)
{
    int server_sock = -1;   //服务器监听套接字描述符
    u_short port = 4000;    //服务器默认监听端口号
    int client_sock = -1;   //已连接套接字描述符
    struct sockaddr_in client_name; //已连接套接字地址结构
    socklen_t  client_name_len = sizeof(client_name);
    pthread_t newthread;    //线程ID

    //根据port创建监听套接字描述符，绑定到相应地址进行监听，得到监听描述符
    server_sock = startup(&port);
    printf("httpd running on port %d\n", port);

    while (1)
    {
        //阻塞于accept，接收来自客户端的连接请求
        client_sock = accept(server_sock,
                (struct sockaddr *)&client_name,
                &client_name_len);
        if (client_sock == -1)
            error_die("accept");
        /* accept_request(&client_sock); */
        //创建线程处理来自客户端的请求
        if (pthread_create(&newthread , NULL, (void *)accept_request, (void *)(intptr_t)client_sock) != 0)
            perror("pthread_create");
    }

    close(server_sock);

    return(0);
}
```

### startup函数

```c
/**********************************************************************/
/* This function starts the process of listening for web connections
 * on a specified port.  If the port is 0, then dynamically allocate a
 * port and modify the original port variable to reflect the actual
 * port.
 * Parameters: pointer to variable containing the port to connect on
 * Returns: the socket */
/**********************************************************************/
int startup(u_short *port)
{
    int httpd = 0;            //服务器监听套接字描述符
    int on = 1;               //设置监听套接字的套接字选项时，作为参数传入，表示开启
    struct sockaddr_in name;  //ipv4套接字地址结构

    //创建服务器监听套接字描述符
    httpd = socket(PF_INET, SOCK_STREAM, 0);
    if (httpd == -1)
        error_die("socket");
    //初始化、设置套接字地址结构
    memset(&name, 0, sizeof(name));
    name.sin_family = AF_INET;                  //IPV4协议
    name.sin_port = htons(*port);               //传入的4000号端口
    name.sin_addr.s_addr = htonl(INADDR_ANY);   //通配地址
    // SO_REUSEADDR 有4个作用：
    //    1）允许启动一个监听服务器并捆绑其众所周知端口，即使以前建立的将该端口用作它们的
    //       本地端口的连接仍然存在
    //    2）允许在同一端口上启动同一服务器的多个实例，只要每个实例捆绑一个不同的本地IP地
    //       址即可
    //    3）允许单个进程捆绑同一端口到多个套接字上，只要每次捆绑指定不同的本地IP地址即可
    //    4）允许完全重复的捆绑：当一个IP地址和端口已捆绑到某个套接字上时，如果传输协议支
    //       持，同样的IP地址和端口还可以捆绑到到另一个套接字上
    // 这里应该是为了起到作用1）的效果
    if ((setsockopt(httpd, SOL_SOCKET, SO_REUSEADDR, &on, sizeof(on))) < 0)  
    {
        error_die("setsockopt failed");
    }
    //调用bind将监听套接字描述符与地址绑定
    if (bind(httpd, (struct sockaddr *)&name, sizeof(name)) < 0)
        error_die("bind");
    //如果传入的端口号为0，说明希望内核分配一个，因此调用getsockname获取内核
    //分配的地址信息，进一步获取分配的端口号
    if (*port == 0)  /* if dynamically allocating a port */
    {
        socklen_t namelen = sizeof(name);
        if (getsockname(httpd, (struct sockaddr *)&name, &namelen) == -1)
            error_die("getsockname");
        *port = ntohs(name.sin_port);  //获取内核分配的端口号，网络字节序转主机字节序
    }
    //调用listen进行监听，backlog参数为5，即已完成和未完成队列中的连接总数为5
    if (listen(httpd, 5) < 0)
        error_die("listen");
    return(httpd);
}
```

### error_die函数

```c
/**********************************************************************/
/* Print out an error message with perror() (for system errors; based
 * on value of errno, which indicates system call errors) and exit the
 * program indicating an error. */
/**********************************************************************/
void error_die(const char *sc)
{
    perror(sc);
    exit(1);
}
```