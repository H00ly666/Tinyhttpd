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
    - 数据接收缓冲器区：`buf[1024]`
    - html方法：`method[255]`
    - url：`url[255]`
    - 路径：`path`
    - GET方法额外参数：`query_string`

### 2）函数

* **主进程**
    - [main](#main函数)：调用**startup**进行监听，循环调用**accept**接受新连接，开启线程处理每个新连接
    - [startup](#startup函数)：根据监听端口号`port`，创建监听套接字描述符`server_sock`，调用**bind**和**listen**进行监听
* **线程**
    - [accept_request](#accept_request函数
)：线程执行的函数。处理客户端的请求
        + [get_line](#get_line函数)：从已连接套接字描述符读取一行数据
        + 如果是POST方法：设置`cgi`为`1`，`query_string`指向`NULL`
        + 如果是GET方法：
            * 1）如果没有额外参数（url中不包含`'?'`）：设置`cgi`为`1`，`query_string`指向额外参数（`'?'`之后）
            * 2）如果含有额外参数（url中包含`'?'`）
        + 通过`path`获取文件状态
            * 1）如果文件不存在，读取html请求剩余数据，然后调用not_found处理
            * 2）如果文件存在
                - 如果文件具有可执行权限（设置了`S_IXUSR`或`S_IXGRP`或`S_IXOTH`标志，[详见](https://github.com/arkingc/note/blob/master/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F/UNIX%E7%8E%AF%E5%A2%83%E9%AB%98%E7%BA%A7%E7%BC%96%E7%A8%8B.md#4%E6%96%87%E4%BB%B6%E8%AE%BF%E9%97%AE%E6%9D%83%E9%99%90)），那么将`cgi`置为`1`
                - 根据`cgi`的值调用不同函数处理
                    + `0`：调用serve_file
                    + `1`：调用execute_cgi
        + 每次处理完一个html请求就关闭已连接套接字描述符（即关闭TCP连接）
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

### accept_request函数

```c
void accept_request(void *arg)
{
    int client = (intptr_t)arg;  //传给线程的参数是已连接套接字描述符
    char buf[1024];
    size_t numchars;             //读取到的字节数据
    char method[255];            //方法
    char url[255];               //地址
    char path[512];              //路径
    size_t i, j;
    struct stat st;
    int cgi = 0;      /* becomes true if server decides this is a CGI
                       * program */
    char *query_string = NULL;

    //从已连接套接字描述符读取一行数据
    numchars = get_line(client, buf, sizeof(buf));
    i = 0; j = 0;   //i用于遍历method、url、path数组，j用于遍历buf
    //从buf中抽取方法数据，保存到method中
    while (!ISspace(buf[i]) && (i < sizeof(method) - 1))
    {
        method[i] = buf[i];
        i++;
    }
    j=i;
    method[i] = '\0';

    //strcasecmp：忽略大小写的字符串比较
    //如果既不是GET方法，也不是POST方法，调用unimplemented函数处理
    if (strcasecmp(method, "GET") && strcasecmp(method, "POST"))
    {
        unimplemented(client);
        return;
    }

    //如果是POST方法、则将cgi设置为1
    if (strcasecmp(method, "POST") == 0)
        cgi = 1;

    i = 0;
    //跳过方法数据后的空格
    while (ISspace(buf[j]) && (j < numchars))
        j++;
    //从buf中抽取url数据
    while (!ISspace(buf[j]) && (i < sizeof(url) - 1) && (j < numchars))
    {
        url[i] = buf[j];
        i++; j++;
    }
    url[i] = '\0';

    //如果method是GET，那么需要在获取url后处理
    if (strcasecmp(method, "GET") == 0)
    {
        query_string = url;
        //下面的处理主要是判断URL中有没有包含'?'，如果包含了'?'，需要将cgi设置为1
        //然后将url从'?'处截断，并使query_string指向'?'之后
        while ((*query_string != '?') && (*query_string != '\0'))
            query_string++;
        if (*query_string == '?')
        {
            cgi = 1;
            *query_string = '\0';
            query_string++;
        }
    }

    //path："htdocs"+url 
    sprintf(path, "htdocs%s", url);
    //如果url末尾为'/'，表示是个目录，那么更新path："htdocs"+url+"index.html"
    if (path[strlen(path) - 1] == '/')
        strcat(path, "index.html");
    //获取文件的状态，保存到st中
    if (stat(path, &st) == -1) {//如果文件状态获取失败（文件不存在）
        //读取客户html请求剩余的数据，然后调用not_found向客户端返回文件不存在错误
        while ((numchars > 0) && strcmp("\n", buf))  /* read & discard headers */
            numchars = get_line(client, buf, sizeof(buf));
        not_found(client);
    }
    else//如果文件存在
    {
        //如果文件是个目录，那么path："htdocs"+url+"/index.html"
        if ((st.st_mode & S_IFMT) == S_IFDIR)
            strcat(path, "/index.html");
        if ((st.st_mode & S_IXUSR) ||
                (st.st_mode & S_IXGRP) ||
                (st.st_mode & S_IXOTH)    )
            cgi = 1;
        //如果没设置cgi，那么调用serve_file进行响应
        //如果设置了cgi，那么调用execute_cgi进行响应
        if (!cgi)
            serve_file(client, path);
        else
            execute_cgi(client, path, method, query_string);
    }

    //处理完客户端的html请求后，关闭已连接套接字描述符（关闭TCP连接）
    close(client);
}
```

### get_line函数

```c
int get_line(int sock, char *buf, int size)
{
    int i = 0;       //buf中保存读取到的字符的起始下标
    char c = '\0';   //用以保存读取到的每个字符
    int n;

    while ((i < size - 1) && (c != '\n'))
    {
        //每次读入一个字符，保存在c中
        n = recv(sock, &c, 1, 0);
        /* DEBUG printf("%02X\n", c); */
        if (n > 0)
        {
            if (c == '\r')
            {
                //MSG_PEEK允许我们查看已经可读的数据，并且系统不在recv返回后丢弃这些数据
                //如果读到'\r'，有些系统是以"\r\n"为换行，有些是以"\r"为换行
                //这里通过查看下一个字符是不是'\n'来处理"\r\n"的情况
                n = recv(sock, &c, 1, MSG_PEEK);
                /* DEBUG printf("%02X\n", c); */
                if ((n > 0) && (c == '\n'))
                    recv(sock, &c, 1, 0);
                else
                    c = '\n';
            }
            buf[i] = c; //还没读到行尾，则将读到的字符存入buf
            i++;        //更新起始下标
        }
        else //下面语句会导致循环终止
            c = '\n';
    }
    buf[i] = '\0';

    return(i);
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