

C1000kPracticeGuide
===================

Document:
   https://github.com/xiaojiaqi/C1000kPracticeGuide/blob/master/docs/cn/c1000K.pdf

 

**For details, please refer to the original link. ** <br/>

https://github.com/xiaojiaqi/C1000kPracticeGuide 



## Cplus version

#### 目录

* [network.h](#network.h)
* [network.cc](#network.cc)
* [timers.h](#timers.h)
* [timers.cc](#timers.cc)
* [socket_util.h](#socket_util.h)
* [socket_util.cc](#socket_util.cc)
* [listenfd.cc](#listenfd.cc)
* [client.cc](#client.cc)
* [worker.cc](#worker.cc)
* [main_event.h](#main_event.h)
* [main_event.cc](#main_event.cc)
* [main.cc](#main.cc)

#### File structure

```markdown
cppserver
├── Debug 
│   ├── src 
│   │   ├── subdir.mk
│   ├── objects.mk 
│   ├── sources.mk 
│   └── makefile 
├── Release 
│   ├── src 
│   │   ├── subdir.mk
│   ├── objects.mk 
│   ├── sources.mk 
│   └── makefile  
├── doc 
│   └── protocol.txt 
└── src
    ├── client.cc
    ├── listenfd.cc
    ├── main.cc
    ├── main_event.cc
    ├── main_event.h
    ├── network.cc
    ├── network.h
    ├── socket_util.cc
    ├── socket_util.h
    ├── timers.cc
    ├── timers.h
    └── worker.cc
```

****

#### 代码分析

 ##### network.h

说明：头文件的引用及函数声明。 <br/>

```c++
#include <pthread.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <sys/epoll.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <netinet/tcp.h>
#include <netdb.h>
#include <unistd.h>
#include <fcntl.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <errno.h>
#include <time.h>
#include <signal.h>
#include <assert.h>
#include "timers.h"

#include <string>
#include <iostream>
#include <sched.h>
#include <sys/types.h> /* See NOTES */
#include <sys/socket.h>
#include "socket_util.h"
```

头文件(.h)中判断是否引用过<br/>

```c++
#ifndef NETWORK_H
#define NETWORK_H
<code body>
#endif
```

[目录](#目录)

****

##### network.cc

```c++
//network.cc
PFdProcess gFdProcess[MAX_FD];

//network.h
typedef int (*readfun)(int epoll, int fd, timer_link *);
typedef int (*writefun)(int epoll, int fd, timer_link *);
typedef int (*closefun)(int epoll, int fd, timer_link *);
typedef int (*timeoutfun)(int epoll, int fd, timer_link *, time_value tnow);

struct FdProcess {
    readfun m_readfun;
    writefun m_writefun;
    closefun m_closefun;
    timeoutfun m_timeoutfun;
    int fd;
    int m_sended;
    time_value m_lastread;
    time_value m_timeout;  // timeout的时间
    bool m_activeclose;

    FdProcess() { init(); }
    void init() {
        m_readfun = NULL;
        m_writefun = NULL;
        m_closefun = NULL;
        m_timeoutfun = NULL;
        m_lastread = get_now();
        m_sended = 0;
        m_lastread = 0;
        m_activeclose = false;
    }
};

typedef struct FdProcess *PFdProcess;
```

代码中的问题：init函数中，m_lastread被赋值两次。m_timeout没有被赋值。<br/>

**解析代码：** <br/>

1. `typedef int (*readfun)(int epoll, int fd, timer_link *);` 

   定义了一个自定义类型readfun，它是一个指向函数的指针，所指向的函数有三个参数，返回值为int。

2. `typedef struct FdProcess *PFdProcess;` 

   相当于 `PFdProcess = *FdProcess`，即PFdProcess为一个指向FdProcess的指针。<br/>

3. 类型time_value: `typedef long long time_value;` <br/>

   timer_link是一个class，在`time.h`中介绍；**为甚么timer_link* 后面没有形参的名字**。 <br/>

reference url: <br/>

-  https://blog.csdn.net/nohackcc/article/details/16961691
- https://www.cnblogs.com/charley_yang/archive/2010/12/15/1907384.html

[目录](#目录)

****

##### time.h

```c++
typedef long long time_value;

struct timer_event {
    time_value times;
    void *arg;
    int eventtype;
};

time_value get_now();  

class timer_link {
    static const int64_t mintimer = 1000 * 1;

   public:
    timer_link() {}

    virtual ~timer_link() {}

    // add new timer
    int add_timer(void *, time_value);

    //
    void *get_timer(time_value tnow);

    // remove timer
    void remote_timer(void *);

    time_value get_mintimer() const;

    int get_arg_time_size() const { return timer_arg_time.size(); }
    int get_time_arg_size() const { return timer_time_arg.size(); }

    void show() const;

   private:
    std::multimap<void *, time_value> timer_arg_time;

    std::multimap<time_value, void *> timer_time_arg;
};
```

**解析代码：** <br/>

1. Multimap允许重复元素，map不允许重复。

2. const类型的类成员函数：<br/>

   - 只有被声明为const的成员函数才能被一个const类对象调用
   - 在类体之外定义const成员函数时，还必须加上const关键字
   - **若将成员成员函数声明为const，则该函数不允许修改类的数据成员**
- 如果据成员是指针，**则const成员函数并不能保证不修改指针指向的对象**，编译器不会把这种修改检测为错误。
   - const成员函数可以被具有相同参数列表的非const成员**函数重载**。在这种情况下，类对象的常量性决定调用哪个函数。

[目录](#目录)

****

##### time.cc

说明：实现 `time.h` 内声明的函数。<br/>

```c++
time_value get_now() {
    struct timeval tv;
    gettimeofday(&tv, NULL);
    return tv.tv_sec * 1000 + tv.tv_usec / 1000; //返回ms。 1s=1000ms,1000us=1ms。
}
/*
struct timeval结构体是Linux系统定义(time.h)。
struct timeval
{
	__time_t tv_sec;        // Seconds. 
	__suseconds_t tv_usec;  // Microseconds. 
};
v_sec为Epoch到创建struct timeval时的秒数，tv_usec为微秒数，即秒后面的零头。
其中__time_t和__suseconds_t都是long int类型。在32位下为4个字节，64位下长度为8个字节。
--------------------------------------------------------------------------------------------

gettimeofday函数：返回自1970-01-01 00:00:00到现在经历的秒数。
函数原型： int gettimeofday(struct timeval *tv, struct timezone *tz)
需要头文件： #include <sys/time.h>
*/
```

```c++
/*
该函数的作用（即传入的void *arg是什么）：
*/
int timer_link::add_timer(void *arg, time_value t) {
    timer_time_arg.insert(std::make_pair(t, arg)); 
    timer_arg_time.insert(std::make_pair(arg, t));
    return 0;
}

//打印一些信息
void timer_link::show() const {
    return;
    printf("arg_time %zu   time_arg %zu \n", timer_arg_time.size(),
           timer_time_arg.size());
}
```

```c++
/*
该函数的作用：

*/
void *timer_link::get_timer(time_value tnow) {
    std::multimap<time_value, void *>::iterator pos_time_arg =
        timer_time_arg.begin();
    void *arg = pos_time_arg->second;
    if (pos_time_arg == timer_time_arg.end()) { //说明timer_time_arg为空
        return NULL;
    }
    if (pos_time_arg->first > tnow) {
        show();
        return NULL;
    }
    timer_time_arg.erase(pos_time_arg); //以待删除兀素的迭代器作为参数，此版本没有返回值

    std::multimap<void *, time_value>::iterator pos_arg_time;
  	//遍历arg键对应的所有元素
    for (pos_arg_time = timer_arg_time.lower_bound(arg);
         pos_arg_time != timer_arg_time.upper_bound(arg); ++pos_arg_time) {
        if (pos_arg_time->second == pos_time_arg->first) { 
          //找到time等于timer_time_arg首元素time的删掉
            timer_arg_time.erase(pos_arg_time);
            show();
            return pos_time_arg->second;
        }
    }
    assert(0);
    show();
    return pos_time_arg->second;
}
/*
1. c++ void*使用：
- void *p指针 只包含了指针位置而不包含指针的类型，因此无法判断出指向对象的长度。(一般的指针包含2个属性指针位置和类型，其中类型就能判断出其长度。) 
- 任何指针都可以赋值给void指针，不需转换，但只获得变量/对象地址而不获得大小。
- void指针赋值给其他类型的指针时都要进行转换。
- void指针不能取值，因为void指针只知道 指向变量/对象的起始地址，而不知道指向变量/对象的大小(占几个字节)所以无法正确引用。
- void指针不能参与指针运算,除非进行转换。
2. 分析
    event->setUserData((void*)10);
    int* data = (int*)event->getUserData();
    CCLOG("data = %d", data);  //注此处不能使用*data
    为什么不能使用*data? 
    (void*)10 是指针本身的值就是10，即把10变成了一个指针地址的值，这个地址是没初始化的(*data是错的，因为地址值为10的这块内存地址内容未知)。
3. 使用void*指针拷贝int数组
原型： void *memcpy(void *dest, const void *src, size_t n);
	int a[2]={1,2};
	int b[2];
	int *c;
	c = (int *)memcpy((void *)b,(void *)a,8 );
	------------------------------------------
	struct mystr st1,st2,*st3;
	st1.id = 1;
	strcpy(st1.name,"test"); //字符串赋值，没有直接使用等号！
	st3 = (struct mystr *)memcpy((void *)(&st2),(void *)(&st1),sizeof(struct mystr)); 
	//对st2，st2要取引用
注意：这里的st3和c指针只是指向了st2和b的位置因为memcpy返回的是一个目的内存起始位置的一个指针。
4. multimap 
	http://c.biancheng.net/view/518.html
	https://blog.csdn.net/hlsdbd1990/article/details/46501391
- equal_range(arg)成员函数： 访问给定键对应的所有元素(相同键可能会对应多个值)，返回一个封装了两个迭代器的 pair 对象，这两个迭代器所确定范围内的元素的键和参数值相等。
	auto pr = people.equal_range("Ann");
	if(pr.first != std::end(people))
	{
    	for (auto iter = pr.first ; iter != pr.second; ++iter)
        	std:cout << iter->first << " is " << iter->second << std::endl;
	}
- lower_bound()成员函数： 返回一个迭代器，指向键不小于k的第一个元素(>=k)
	upper_bound()成员函数： 返回一个迭代器，指向键大于k的第一个元素
	在同一键上调用lower_bound 和upper_bound，将产生一个迭代器范围，指示出该键所关联的所有元素。
- erase()成员函数有3个版本： 
	1. 以待删除兀素的迭代器作为参数，这个函数没有返回值；
	2. 以一个键作为参数，它会删除容器中所有含这个键的元素，返回容器中被移除元素的个数；
	3. 接受两个迭代器参数，它们指定了容器中的一段元素，这个范围内的所有元素都会被删除，这个函数返回的迭代器指向最后一个被删除元素的后一个位置。
*/
```

```c++
/*
该函数的作用：

*/
void timer_link::remote_timer(void *arg) {
    std::multimap<void *, time_value>::iterator pos_arg_time;
    for (pos_arg_time = timer_arg_time.lower_bound(arg);
         pos_arg_time != timer_arg_time.upper_bound(arg);) {
        std::multimap<time_value, void *>::iterator pos_time_arg;
        time_value t = pos_arg_time->second;
        for (pos_time_arg = timer_time_arg.lower_bound(t);
             pos_time_arg != timer_time_arg.upper_bound(t);) {
            if (pos_time_arg->second == arg) {
                timer_time_arg.erase(pos_time_arg++);
            } else {
                pos_time_arg++;
            }
        }
        timer_arg_time.erase(pos_arg_time++);
    }
    show();
}
```

```c++
time_value timer_link::get_mintimer() const {
    if (timer_time_arg.size() == 0) {
        return timer_link::mintimer;  //static const int64_t mintimer = 1000 * 1;
    }
    time_value tnow = get_now();
    std::multimap<time_value, void *>::const_iterator pos_time_arg =
        timer_time_arg.begin();
    time_value t = tnow - pos_time_arg->first;
    if (t >= 0) {
        return 0;
    }
    return -t;
}
```

[目录](#目录)

****

##### socket_util.h

```c++
//声明两个socket设置函数

#ifndef SOCKET_UTIL_H
#define SOCKET_UTIL_H
// set socket no block
int set_noblock(int sClient);
// set reused fd
int set_reused(int fd);
#endif
```

[目录](#目录)

****

##### socket_util.cc

```c++
int set_noblock(int sClient) {

    int opts;

    opts = fcntl(sClient, F_GETFL);
    if (opts < 0) {
        perror("fcntl(sock,GETFL)");
        exit(1);
    }
    opts = opts | O_NONBLOCK;
    if (fcntl(sClient, F_SETFL, opts) < 0) {
        perror("fcntl(sock, SETFL, opts)");
        exit(1);
    }
    return 0;
}
/*
fcntl函数的使用：
ref: 1)https://www.cnblogs.com/lonelycatcher/archive/2011/12/22/2297349.html (!!!)
     2)https://blog.csdn.net/fengxinlinux/article/details/51980837
1. 函数原型：
  #include<unistd.h>
  #include<fcntl.h>
  int fcntl(int fd, int cmd);
  int fcntl(int fd, int cmd, long arg);
  int fcntl(int fd, int cmd ,struct flock* lock);
2. fcntl()针对(文件)描述符提供控制。参数fd是被参数cmd操作的描述符。针对cmd的值，fcntl能够接受第三个参数int arg。
3. fcntl()的返回值与命令有关。如果出错，所有命令都返回－1，如果成功则返回某个其他值。
	  F_DUPFD   返回新的文件描述符
    F_GETFD   返回相应标志
    F_GETFL , F_GETOWN   返回一个正的进程ID或负的进程组ID
4. fcntl函数有5种功能：
    1. 复制一个现有的描述符(cmd=F_DUPFD). 
    2. 获得／设置文件描述符标记(cmd=F_GETFD或F_SETFD). 
    3. 获得／设置文件状态标记(cmd=F_GETFL或F_SETFL). 
    4. 获得／设置异步I/O所有权(cmd=F_GETOWN或F_SETOWN). 
    5. 获得／设置记录锁(cmd=F_GETLK , F_SETLK或F_SETLKW).
--------------------------------------------------------------------------

- F_GETFL: 取得fd的文件状态标志，如同下面的描述一样(arg被忽略)，在说明open函数时，已说明
了文件状态标志。
- F_SETFL: 设置给arg描述符状态标志，可以更改的几个标志是：O_APPEND，O_NONBLOCK，O_SYNC 和 O_ASYNC。而fcntl的文件状态标志总共有7个：O_RDONLY , O_WRONLY , O_RDWR , O_APPEND , O_NONBLOCK , O_SYNC和O_ASYNC

   O_NONBLOCK: 非阻塞I/O，如果read(2)调用没有可读取的数据，或者如果write(2)操作将阻塞，则read或write调用将返回-1和EAGAIN错误
*/
```

https://www.cnblogs.com/lonelycatcher/archive/2011/12/22/2297349.html

```c++
int set_reused(int fd) {
    int on = 1;
    int rt = setsockopt(fd, SOL_SOCKET, SO_REUSEADDR, &on, sizeof(on));
    return rt;
}
/*
closesocket（一般不会立即关闭而经历TIME_WAIT的过程）后想继续重用该socket：
BOOL bReuseaddr=TRUE;
setsockopt (s,SOL_SOCKET ,SO_REUSEADDR,(const char*)&bReuseaddr,sizeof(BOOL));

setsockopt()函数使用详解: https://blog.csdn.net/tody_guo/article/details/5972588
*/
```

[目录](#目录)

****

##### listenfd.cc

```c++
extern PFdProcess gFdProcess[MAX_FD]; //Definition in network.cc
extern int sv[MAX_CPU][2]; //Definition in main.cc
extern struct sendfdbuff order_list[MAX_CPU]; //Definition in main.cc
extern int cpu_num; //Definition in main.cc
extern pthread_mutex_t dispatch_mutex; //Definition in main.cc
extern pthread_cond_t dispatch_cond; //Definition in main.cc

extern struct accepted_fd *g_fd;
/*
1. extern修饰的关键字，具有文件外部链接，但是声明extern变量时，编译器并不会给这个变量分配内存，在另外的文件中定义这个文件时才会为其分配内存。
2. 一旦声明了extern关键字，对编译器来意味着：
	- 这个变量声明（即数据类型和变量名，但是编译器并没有分配内存）
	- 这个变量的定义在其他文件中(在定义变量的文件中编译器才会为其分配内存)
将声明与实现分离，便于文件之间的数据共享。
3. extern变量的初始化需要在全局作用域中初始化，所以在局部作用域中不论是声明并初始化，或者声明与初始化分开都会导致编译器报错。
4. ref: https://www.cnblogs.com/yc_sunniwell/archive/2010/07/14/1777431.html（!!!）
				https://www.jianshu.com/p/2f8b049f32e3
*/
```

```c++
// listen fd, 当有可读事件发生时调用
int accept_readfun(int epollfd, int listenfd, timer_link *timerlink) {
    struct sockaddr_in servaddr;
    int len = sizeof(servaddr);
    int newacceptfd = 0;

    char buff[MAXACCEPTSIZE * 4] = {0}; //const int MAXACCEPTSIZE = 1024; (in network.h)
    int acceptindex = 0;
    int *pbuff = (int *)buff; //为什么buff是char型的？

    while (true) {
        memset(&servaddr, 0, sizeof(servaddr));
        newacceptfd =
            accept(listenfd, (struct sockaddr *)&servaddr, (socklen_t *)&len);
        if (newacceptfd > 0) {            // accept new fd
            if (newacceptfd >= MAX_FD) {  // 太多链接 系统无法接受 MAX_FD=2,252,800
                Log << "accept fd  close fd:" << newacceptfd << std::endl;
                close(newacceptfd); //太多链接 系统无法接受,关闭当前不能接受的链接
                continue; //不是break
            }
            *(pbuff + acceptindex) = newacceptfd;
            acceptindex++;
            if (acceptindex < MAXACCEPTSIZE) {//const int MAXACCEPTSIZE = 1024;
                continue; //小于1024，系统可以接受，不用其他的处理
            }

            struct accepted_fd *p = new accepted_fd();
            p->len = acceptindex * 4;
            memcpy(p->buff, buff, p->len);
            //把buff数组内容拷贝到accepted_fd结构的buff数组，拷贝p->len个字节
            p->next = NULL;
            pthread_mutex_lock(&dispatch_mutex);

            if (g_fd) {
                p->next = g_fd;
                g_fd = p;
            } else {
                g_fd = p;
            }
            pthread_cond_broadcast(&dispatch_cond);//唤醒条件变量上阻塞的线程
            pthread_mutex_unlock(&dispatch_mutex);
            acceptindex = 0;  
            //每一个accepted_fd存储1024个socket描述符，不同的accepted_fd使用next连接

        } else if (newacceptfd == -1) {  //accpet出错则返回-1
            if (errno != EAGAIN && errno != ECONNABORTED && errno != EPROTO &&
                errno != EINTR) { //
                Log << "errno ==" << errno << std::endl;
                perror("read error1 errno");
                exit(0);
            } else { //在accept函数调用前，客户端终止链接

                if (acceptindex) {
                    struct accepted_fd *p = new accepted_fd();//什么时候要加accepted_fd
                    p->len = acceptindex * 4;
                    memcpy(p->buff, buff, p->len);
                    p->next = NULL;
                    pthread_mutex_lock(&dispatch_mutex);

                    if (g_fd) {
                        p->next = g_fd;
                        g_fd = p;
                    } else {
                        g_fd = p;
                    }
                    pthread_cond_broadcast(&dispatch_cond);
                    pthread_mutex_unlock(&dispatch_mutex);
                    acceptindex = 0;
                }
                break;
            }
        } else {
            Log << "errno 3==" << errno << std::endl;
            exit(1);
            break;
        }
    }
    return 1;
}
/*
1. accpet函数原型：int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
	 头文件: #include <sys/types.h>  & #include <sys/socket.h>
	 eg: newacceptfd = accept(listenfd, (struct sockaddr *)&servaddr, (socklen_t *)&len);
	 参数描述: sockfd是由socket函数返回的套接字描述符，参数addr和addrlen用来返回已连接的对端进程（客户端）的协议地址。如果我们对客户端的协议地址不感兴趣，可以把arrd和addrlen均置为空指针。
	 返回：若成功则为非负描述符；若出错则为-1，相应地设定全局变量errno。
2. 上述代码中，accept的返回值有3中：
	 1）大于0：连接成功
	 2）等于-1：连接失败
	 3）其他值：（什么时候是该值？）
3. 当newacceptfd > 0时，
	 1）超过设置的最大连接数，将该连接关闭。服务器没有能力处理新的连接。
	 2）当socket描述符小于1024时，是系统可接受的范围，无需另外的处理，continue。
	 3）socket描述符值大于等于1024时，
	 		a)定义结构
	 			struct accepted_fd {
          int len;
          char buff[MAXACCEPTSIZE * 4];
          struct accepted_fd *next;
				};
	 		b)memcpy函数：void *memcpy(void*dest, const void *src, size_t n);
	 			功能：由src指向地址为起始地址的连续n个字节的数据复制到以destin指向地址为起始地址的空间内。
	 			头文件：#include<string.h>
	 			返回值：函数返回一个指向dest的指针。
	 			使用注意：
	 				- source和destin所指内存区域不能重叠，函数返回指向destin的指针。
	 				- 与strcpy相比，memcpy并不是遇到'\0'就结束，而是一定会拷贝完n个字节。
	 				- strcpy只能拷贝字符串，遇到'\0'结束拷贝；memcpy用来做内存拷贝，可以拷贝任何数据类型的对象
	 		c)每当socket描述符的个数为1024时，将这些描述符拷贝给一个accepted_fd结构对象。将不同的accepted_fd用next指针连接。
      d)对加锁的位置有疑惑：什么时候需要加锁呢？与多线程之间的关系？
      e)pthread_cond_broadcast: 唤醒条件变量上阻塞的线程，当解锁后，这些线程竞争该锁
4. 当newacceptfd = -1时，
	 1) ref:
	 		- https://blog.csdn.net/whycold/article/details/48179659(epoll错误码正确处理)
	 		
	 2) accept有两种模式：
	 		a)阻塞模式：如果没有请求过来, 会阻塞进程, 进程会进入睡眠状态.
	 		b)非阻塞模式：
	 			- 如果没有请求过来, 将会返回EAGAIN或EWOULDBLOCK
	 			- 加入事件监听, 触发可读事件, 则表示有新的连接过来.
	 			- 当客户在服务器调用accept之前中止某个连接时, accept调用可以立即返回-1, errno设置为ECONNABORTED或者EPROTO错误忽略这两个错误.
	 -------------------------------------------------------------------------------
	 
	 当有一个已完成的连接准备好被accept时，select将作为可读描述符返回该连接的监听套接字。因此，如果我们使用select在某个监听套接字上等待一个外来连接，那就没有必要把监听套接字设置为非阻塞，这是因为如果select告诉我们该套接字上已有连接就绪，那么随后的accept调用不应该阻塞。
   但这里存在一个可能让我们掉入陷阱的定时问题（BUG）：
   当客户端在跟服务器建立连接之后发送了一个RST包，这个时候accept就会阻塞，直到有下一个已完成的连接准备好被accept为止。
   
   客户建立一个连接并随后终止它：
   A:select向服务器返回监听socket可读,但是服务器要在一段时间之后才能调用accept;
   B:在服务器从select返回和调用accept之前,收到从客户发送过来的RST;
   C:这个已经完成的连接被从队列中删除,我们假设没有其它已完成的连接存在;
   D:服务器调用accept,但是由于没有其它已完成的连接存在,因而服务器被阻塞了;
   注意,服务器会被一直阻塞在accept调用上,直到另外一个客户建立一个连接为止;但是如果一直没有其它客户建立连接,那么服务器将仍然一直被阻塞在accept调用上,不处理任何其他已就绪的socket;
   
   本问题的解决办法如下：
   1）当使用select获悉某个监听套接字上何时有已完成连接准备好被accept时，总是把这个监听套接字设置为非阻塞。
   2）在后续的accept调用中忽略以下错误：EWOULDBLOCK（源自Berkeley的实现，客户终止连接时），ECONNABORTED（POSIX实现，客户终止连接时），EPROTO（SVR4实现，客户终止连接时）和EINTR（如果有信号被捕获)。
	  -------------------------------------------------------------------------------
	  
	  3)accept函数调用前，客户端断开连接：
	  	将buff数组内已连接的socket描述符保存成accepted_fd结构（此时数组的元素可能没有达到1024），为什么要保存呢？直接continue不可以么？
	  	因为收到该错误，说明accept调用失败，没有连接要accept继续处理（代码里有break），退出此程序。

5. 总结：
	有连接过来时，accept_readfun返回描述符（可持续处理多个连接）；当没有连接时，退出该函数，返回1。发生错误时，exit退出程序。
	
	此处的exit(0)是不是不符合服务器一直维护服务的要求？
	
6. accept_readfun函数的第三个参数好像没有用到？
```

1. 为什么使用线程锁<br/>

   在多线程应用程序中，当多个线程共享相同的内存时，如同时访问一个变量时，需要确保每个线程看到一致的数据视图，即保证所有线程对数据的修改是一致的。如：当一个线程在修改变量的值时，其他线程在读取这个变量时可能会得到一个不一致的值。

2. 用程序修改变量值时所经历的三个步骤:<br/>

   1. 从内存单元读入寄存器<br/>
   2. 在寄存器中对变量操作（加/减1）<br/>
   3. 把新值写回到内存单元<br/>

   不能预期以上三步骤在一个总线周期内完成，所以也就不能指望多线程程序如预期那样运行。<br/>

3. 线程锁

   1. 互斥量/互斥锁<br/>

      **同一时间只有一个线程访问数据**。互斥量(mutex)就是一把锁。<br/>

      多个线程只有一把锁一个钥匙，谁上的锁就只有谁能开锁。当一个线程要访问一个共享变量时，先用锁把变量锁住，然后再操作，操作完了之后再释放掉锁，完成。<br/>

      当另一个线程也要访问这个变量时，发现这个变量被锁住了，无法访问，它就会一直等待，直到锁没了，它再给这个变量上个锁，然后使用，使用完了释放锁，以此进行。<br/>

      ****

      互斥变量使用特定的数据类型：`pthread_mutex_t`，使用互斥量前要先初始化，使用的函数如下：

      ```c++
      #include <pthread.h>
      int pthread_mutex_init(pthread_mutex_t *restrict mutex, const pthread_mutexattr_t *restrict attr);
      //或
      pthread_mutex_t mutex=PTHREAD_MUTEX_INITIALIZER;
      
      int pthread_mutex_destroy(pthread_mutex_t *mutex);
      //简单的使用可以使用默认的属性初始化互斥量，函数的后一个参数设置为NULL即可。
      ```

      对互斥量加锁解锁的函数如下：

      ```c++
      #include <pthread.h>
      int pthread_mutex_tlock(pthread_mutex_t *mutex);
      int pthread_mutex_trylock(pthread_mutex_t *mutex);
      int pthread_mutex_unlock(pthreadd_mutex_t *mutex);
      //函数pthread_mutex_trylock会尝试对互斥量加锁，如果该互斥量已经被锁住，函数调用失败，返回EBUSY，否则加锁成功返回0，线程不会被阻塞。
      ```

      Ref: https://blog.csdn.net/guotianqing/article/details/80559865

   2. 条件变量

      **初始化** <br/>

      ```c++
      #include <pthread.h>
      int pthread_cond_init(pthread_cond_t *cv, const pthread_condattr_t *cattr);
      //或
      pthread_cond_t cv = PTHREAD_COND_INITIALIZER;
      //返回值：函数成功返回0；任何其他返回值都表示错误
      ```

      初始化一个条件变量。当参数cattr为空指针时，函数创建的是一个缺省的条件变量。否则条件变量的属性将由cattr中的属性值来决定。<br/>

      不能由多个线程同时初始化一个条件变量。当需要重新初始化或释放一个条件变量时，应用程序必须保证这个条件变量未被使用。<br/>

      ****

      **pthread_cond_wait**   ?<br/>

      ```c++
      //阻塞在条件变量上
      #include <pthread.h>
      int pthread_cond_wait(pthread_cond_t *cv, pthread_mutex_t *mutex);
      //返回值：函数成功返回0；任何其他返回值都表示错误
      ```

      1. 函数将解锁mutex参数指向的互斥锁，并使当前线程阻塞在cv参数指向的条件变量上。

      2. 被阻塞的线程可以被pthread_cond_signal函数，pthread_cond_broadcast函数唤醒，也可能在被信号中断后被唤醒。

      3. pthread_cond_wait函数的返回并不意味着条件的值一定发生了变化，必须重新检查条件的值。

      4. pthread_cond_wait函数返回时，相应的互斥锁将被当前线程锁定，即使是函数出错返回。

      5. 一般一个条件表达式都是在一个互斥锁的保护下被检查。当条件表达式未被满足时，线程将仍然阻塞在这个条件变量上。当另一个线程改变了条件的值并向条件变量发出信号时，等待在这个条件变量上的一个线程或所有线程被唤醒，接着都试图再次占有相应的互斥锁。

      6. 阻塞在条件变量上的线程被唤醒以后，直到pthread_cond_wait()函数返回之前条件的值都有可能发生变化。所以函数返回以后，在锁定相应的互斥锁之前，必须重新测试条件值。最好的测试方法是循环调用pthread_cond_wait函数，并把满足条件的表达式置为循环的终止条件。如：

         ```c++
         pthread_mutex_lock();
         while (condition_is_false)
         	pthread_cond_wait();
         pthread_mutex_unlock();
         //阻塞在同一个条件变量上的不同线程被释放的次序是不一定的。
         ```

      ****

      **pthread_cond_signal** <br/>

      ```c++
      //解除在条件变量上的阻塞
      #include <pthread.h>
      int pthread_cond_signal(pthread_cond_t *cv);
      //返回值：函数成功返回0；任何其他返回值都表示错误
      ```

      1. 函数被用来释放被阻塞在指定条件变量上的一个线程。
      2. 必须在互斥锁的保护下使用相应的条件变量。否则对条件变量的解锁有可能发生在锁定条件变量之前，从而造成死锁。
      3. 唤醒阻塞在条件变量上的所有线程的顺序由调度策略决定，如果线程的调度策略是SCHED_OTHER类型的，系统将根据线程的优先级唤醒线程。
      4. 如果没有线程被阻塞在条件变量上，那么调用pthread_cond_signal()将没有作用。

      ****

      **pthread_cond_timedwait** <br/>

      ```c++
      //阻塞直到指定时间
      #include <pthread.h>
      #include <time.h>
      int pthread_cond_timedwait(pthread_cond_t *cv, pthread_mutex_t *mp, const structtimespec * abstime);
      //返回值：函数成功返回0；任何其他返回值都表示错误
      ```

      1. 函数到了一定的时间，即使条件未发生也会解除阻塞。这个时间由参数abstime指定。函数返回时，相应的互斥锁往往是锁定的，即使是函数出错返回。

      2. pthread_cond_timedwait函数也是退出点。

      3. 超时时间参数是指一天中的某个时刻。使用举例：

         ```c++
         pthread_timestruc_t to;
         to.tv_sec = time(NULL) + TIMEOUT;
         to.tv_nsec = 0;
         //超时返回的错误码是ETIMEDOUT。
         ```

      ****

      **pthread_cond_broadcast** <br/>

      ```c++
      //释放阻塞的所有线程
      #include <pthread.h>
      int pthread_cond_broadcast(pthread_cond_t *cv);
      //返回值：函数成功返回0；任何其他返回值都表示错误
      ```

      1. 函数唤醒所有被pthread_cond_wait函数阻塞在某个条件变量上的线程，参数cv被用来指定这个条件变量。当没有线程阻塞在这个条件变量上时，pthread_cond_broadcast函数无效。
      2. 由于pthread_cond_broadcast函数唤醒所有阻塞在某个条件变量上的线程，这些线程被唤醒后将再次竞争相应的互斥锁，所以必须小心使用pthread_cond_broadcast函数。

      ****

      **pthread_cond_destroy**  <br/>

      ```c++
      //释放条件变量
      #include <pthread.h>
      int pthread_cond_destroy(pthread_cond_t *cv);
      //返回值：函数成功返回0；任何其他返回值都表示错误
      ```

      1. 释放条件变量。
      2. 注意：条件变量占用的空间并未被释放。

      ****

      **唤醒丢失问题** <br/>

      在线程未获得相应的互斥锁时调用pthread_cond_signal或pthread_cond_broadcast函数可能会引起唤醒丢失问题。<br/>

      唤醒丢失往往会在下面的情况下发生：<br/>

      1. 一个线程调用pthread_cond_signal或pthread_cond_broadcast函数；
      2. 另一个线程正处在测试条件变量和调用pthread_cond_wait函数之间；
      3. 没有线程正在处在阻塞等待的状态下。

      ****

      Ref: https://blog.csdn.net/ithomer/article/details/6031723

```c++
int accept_write(int fd) { return 1; }
```

```c++
int fdsend_writefun(int epollfd, int sendfd, timer_link *timerlink) {
    Log << "call fdsend_writefun" << std::endl;
    return sendbuff(sendfd);
}
/*
参数1与参数3没有使用
*/
```

```c++
int sendbuff(int fd) {
    int needsend = order_list[fd].len; //要发送的长度
    int sended = 0;  //已经发送的长度
    int sendlen = 0; //每次发送的长度
    char *buff = order_list[fd].buff;
    while (true) {
        if (needsend - sended > 0) {
            sendlen = send(fd, buff + sended, needsend - sended, 0);
            if (sendlen > 0) {
                sended += sendlen;
            } else {
                break;
            }
        } else {
            break;
        }
    }
    if (needsend == sended) {
    } else {
        memmove(buff, buff + sended, needsend - sended); //数据没有发送完，将发送过的数据清掉
    }
    order_list[fd].len = needsend - sended; 
    return sended;  //返回发送了多少数据
}
/*
1. 结构
  struct sendfdbuff {
      int len;
      char buff[4 * 30000];
  };
  struct sendfdbuff order_list[MAX_CPU];
  #define MAX_CPU 128
2. memmove函数
	 头文件：#include <string.h>
	 函数声明：void *memmove(void *dest, const void *src, size_t n);
	 功能：memmove内存拷贝函数，功能是拷贝n个字节到目标地址，目标内存和源地址内存可以重叠。
	 区别：另外一个内存拷贝函数memcpy，不可以内存重叠。
   一个长度n的数组，使用memmove向后移一位，最多能拷贝n-2个字节。为什么不是n-1？
3. 疑问：为什么 MAX_CPU 为128，如果发送的socket连接超过128个怎么办？哪里有控制么?  
   
*/
```

```c++
int movefddata(int fd, char *buff, int len) {
    memmove(order_list[fd].buff + order_list[fd].len, buff, len);
    order_list[fd].len += len;
    return order_list[fd].len;
}
/*
向fd的缓存区添加要发送的数据
*/
```

```c++
void *dispatch_conn(void *arg) {
    static long sum = 0;
    struct accepted_fd *paccept = NULL;
    for (;;) {
        pthread_mutex_lock(&dispatch_mutex);
        if (g_fd) {
            paccept = g_fd;
            g_fd = NULL;

        } else {
            pthread_cond_wait(&dispatch_cond, &dispatch_mutex);
            paccept = g_fd;
            g_fd = NULL;
        }
        pthread_mutex_unlock(&dispatch_mutex);
        while (paccept) {
            int index = sum % (cpu_num - 1) + 1;
            int sendfd = sv[index][0];
            movefddata(sendfd, paccept->buff, paccept->len);
            sendbuff(sendfd);
            ++sum;
            struct accepted_fd *next = paccept->next;
            delete paccept;
            paccept = next;
        }
    }
    return NULL;
}
/*

*/
```

[目录](#目录)

****

##### client.cc

```c++
int client_readfun(int epoll, int fd, timer_link* timer) {
    char buff[1024];

    while (true) {
        int len = read(fd, buff, sizeof(buff) - 1);
        if (len == -1) { //出错,退出循环，client_readfun返回1
            buff[0] = 0;
            break;

        } else if (len == 0) { //表示到达文件末尾，client_readfun返回-1
            buff[len] = 0; //等价于buff[0] = 0;
            gFdProcess[fd]->m_activeclose = true;
            return -1;
        }
        if (len > 0) {  //继续下一次循环，接着读取数据
            Log << "'" << buff << "'" << std::endl;
        }
    }
    return 1;
}
//---------------------------------------------------------------------------------------
1. read函数
	1）头文件 #include <unistd.h> 
	2）函数声明 ssize_t read(int fd, void *buf, size_t count); 
	3）功能：read函数从打开的设备或文件中读取数据，成功返回读取的字节数，出错返回-1并设置errno，如果在调read之前已到达文件末尾，则这次read返回0。
	4）使用注意：
		a) 参数count是请求读取的字节数，读上来的数据保存在缓冲区buf中，同时文件的当前读写位置向后移。
			 注意这个读写位置和使用C标准I/O库时的读写位置有可能不同，这个读写位置是记在内核中的，而使用C标准I/O库时的读写位置是用户空间I/O缓冲区中的位置。
			 比如用fgetc读一个字节，fgetc有可能从内核中预读1024个字节到I/O缓冲区中，再返回第一个字节，这时该文件在内核中记录的读写位置是1024，而在FILE结构体中记录的读写位置是1。注意返回值类型是ssize_t，表示有符号的size_t，这样既可以返回正的字节数、0（表示到达文件末尾）也可以返回负值-1（表示出错）。
		b) read函数返回时，返回值说明了buf中前多少个字节是刚读上来的。有些情况下，实际读到的字节数（返回值）会小于请求读的字节数count，例如：读常规文件时，在读到count个字节之前已到达文件末尾。例如，距文件末尾还有30个字节而请求读100个字节，则read返回30，下次read将返回0。
		c) 从终端设备读，通常以行为单位，读到换行符就返回了。
		d) 从网络读，根据不同的传输层协议和内核缓存机制，返回值可能小于请求的字节数，后面socket编程部分会详细讲解。
2. 当len > 0时，把读取到的数据输出，下一次循环接着使用read读取数据，内核会记录上一次的读取位置，无需自己记录读取位置
3. 疑问：
	1）读取数据不成功时，为什么设置buff[0] = 0; 我认为buff所有元素都清零比较好？
	2）返回1与返回-1？
4. 注意区别
/* Read "n" bytes from a descriptor. */
ssize_t readn(int fd, void *vptr, size_t n)
{
    size_t    nleft;
    ssize_t    nread;
    char    *ptr;

    ptr = vptr;
    nleft = n;
    while (nleft > 0) {
        if ( (nread = read(fd, ptr, nleft)) < 0) {
            if (errno == EINTR)
                nread = 0;        /* and call read() again */
            else
                return(-1);
        } else if (nread == 0)
            break;                /* EOF */

        nleft -= nread;
        ptr   += nread;
    }
    return(n - nleft);        /* return >= 0 */
}
此段代码有指针，是因为要将fd读到的数据全部写到ptr中
而client_readfun函数是每一次读取都直接输出到中断，buff中不用保留数据，因此每次读取数据直接覆盖就可以了
```

```c++
char senddata[2048];
const int presend = 500; //500字节，2000bytes。与senddata定义的长度是匹配的。
```

```c++
int client_writefun(int epoll, int fd, timer_link* timer) {

    int len = presend - gFdProcess[fd]->m_sended;  //const int presend = 500;
    len = write(fd, &(senddata[0]) + gFdProcess[fd]->m_sended, len);
    if (len == 0) {
        gFdProcess[fd]->m_activeclose = true;
    } else if (len > 0) {
        gFdProcess[fd]->m_sended += len;
        if (gFdProcess[fd]->m_sended == presend) {
            gFdProcess[fd]->m_sended = 0;
        }
    }
    return 1;
}
//---------------------------------------------------------------------------------------
ref: https://www.cnblogs.com/xiehongfeng100/p/4619451.html
1. write函数
	1）头文件：#include <unistd>
	2）函数原型：ssize_t write(int fd, void *buf, size_t nbytes)
  3) 读常规文件是不会阻塞的，不管读多少字节，read一定会在有限的时间内返回。从终端设备或网络读则不一定，如果从终端输入的数据没有换行符，调用read读终端设备就会阻塞，如果网络上没有接收到数据包，调用read从网络读就会阻塞，至于会阻塞多长时间也是不确定的，如果一直没有数据到达就一直阻塞在那里。同样，写常规文件是不会阻塞的，而向终端设备或网络写则不一定。
2. gFdProcess[fd]->m_sended
	 记录上一次数据发送的位置，以便下一次接着该位置发送数据。
  
  
  
  
  
/* Write "n" bytes to a descriptor. */
ssize_t writen(int fd, const void *vptr, size_t n)
{
    size_t nleft;
    ssize_t nwritten;
    const char *ptr;

    ptr = vptr;
    nleft = n;
    while (nleft > 0) {
        if ( (nwritten = write(fd, ptr, nleft)) <= 0) {
            if (nwritten < 0 && errno == EINTR)
                nwritten = 0;        /* and call write() again */
            else
                return(-1);            /* error */
        }

        nleft -= nwritten;
        ptr   += nwritten;
    }
    return(n);
}
/* end writen */

void
Writen(int fd, void *ptr, size_t nbytes)
{
    if (writen(fd, ptr, nbytes) != nbytes)
        err_sys("writen error");
}
```

##### read/write的语义：为什么会阻塞？<br/>

- write

  首先，write成功返回，**只是buf中的数据被复制到了kernel中的TCP发送缓冲区。**至于数据什么时候被发往网络，什么时候被对方主机接收，什么时候被对方进程读取，系统调用层面不会给予任何保证和通知。<br/>

  write在什么情况下会阻塞？当kernel的该socket的发送缓冲区已满时。对于每个socket，拥有自己的send buffer和receive buffer。从Linux 2.6开始，两个缓冲区大小都由系统来自动调节（autotuning），但一般在default和max之间浮动。<br/>

  ```shell
  # 获取socket的发送/接受缓冲区的大小：（后面的值是在Linux 2.6.38 x86_64上测试的结果）
  sysctl net.core.wmem_default       #126976
  sysctl net.core.wmem_max　　　　    #131071
  ```

  已经发送到网络的数据依然需要暂存在send buffer中，只有收到对方的ack后，kernel才从buffer中清除这一部分数据，为后续发送数据腾出空间。接收端将收到的数据暂存在receive buffer中，自动进行确认。但如果socket所在的进程不及时将数据从receive buffer中取出，最终导致receive buffer填满，由于**TCP的滑动窗口和拥塞控制**，接收端会阻止发送端向其发送数据。这些控制皆发生在TCP/IP栈中，对应用程序是透明的，应用程序继续发送数据，最终导致send buffer填满，write调用阻塞。<br/>

  🔥一般来说，由于**接收端进程从socket读数据的速度**跟不上**发送端进程向socket写数据的速度**，最终导致**发送端write调用阻塞。**

- read

  从socket的receive buffer中拷贝数据到应用程序的buffer中。read调用阻塞，通常是发送端的数据没有到达。

****

##### blocking（默认）和nonblock模式下read/write行为的区别<br/>

将socket fd设置为nonblock（非阻塞）是在服务器编程中常见的做法，采用blocking IO并为每一个client创建一个线程的模式开销巨大且可扩展性不佳（带来大量的切换开销），更为通用的做法是采用线程池+Nonblock I/O+Multiplexing（select/poll，以及Linux上特有的epoll）。<br/>

```c++
// 设置一个文件描述符为nonblock
int set_nonblocking(int fd)
{
    int flags;
    if ((flags = fcntl(fd, F_GETFL, 0)) == -1)
        flags = 0;
    return fcntl(fd, F_SETFL, flags | O_NONBLOCK);
}
```

几个重要的结论：<br/>

**1. read总是在接收缓冲区有数据时立即返回，而不是等到给定的read buffer填满时返回。** <br/>**只有当receive buffer为空时，blocking模式才会等待**，而nonblock模式下会立即返回-1（errno = EAGAIN或EWOULDBLOCK） <br/>**注：**阻塞模式下，当对方socket关闭时，read会返回0。 <br/>

**2. blocking的write只有在缓冲区足以放下整个buffer时才返回**（与blocking read并不相同） <br/>
**nonblock write则是返回能够放下的字节数**，之后调用则返回-1（errno = EAGAIN或EWOULDBLOCK）<br/>对于blocking的write有个特例：当write正阻塞等待时对面关闭了socket，则write则会立即将剩余缓冲区填满并返回所写的字节数，再次调用则write失败（connection reset by peer）。

****

##### read/write对连接异常的反馈行为

对应用程序来说，与另一进程的TCP通信其实是**完全异步**的过程：<br/>

1. 我并不知道对面什么时候、能否收到我的数据
2. 我不知道什么时候能够收到对面的数据
3. 我不知道什么时候通信结束（主动退出或是异常退出、机器故障、网络故障等等）

解决：<br/>

对于1和2，采用write() -> read() -> write() -> read() ->...的序列，通过blocking read或者nonblock read+轮询的方式，应用程序基于可以保证正确的处理流程。<br/>

对于3，**kernel将这些事件的“通知”通过read/write的结果返回给应用层。**<br/>

🔥一个例子

**假设A机器上的一个进程a正在和B机器上的进程b通信：某一时刻a正阻塞在socket的read调用上（或者在nonblock下轮询socket）**<br/>

　　当b进程终止时，无论应用程序是否显式关闭了socket（OS会负责在进程结束时关闭所有的文件描述符，对于socket，则会发送一个FIN包到对面）。<br/>

　　”同步通知“：进程a对已经收到FIN的socket调用read，如果已经读完了receive buffer的剩余字节，则会返回EOF:0 <br/>

　　”异步通知“：如果进程a正阻塞在read调用上（前面已经提到，此时receive buffer一定为空，因为read在receive buffer有内容时就会返回），则read调用立即返回EOF，进程a被唤醒。<br/>

　　socket在收到FIN后，虽然调用read会返回EOF，但**进程a依然可以其调用write，因为根据TCP协议，收到对方的FIN包只意味着对方不会再发送任何消息**。 在一个双方正常关闭的流程中，收到FIN包的一端将剩余数据发送给对面（通过一次或多次write），然后关闭socket。<br/>

　　**但是事情远远没有想象中简单。**优雅地（gracefully)关闭一个TCP连接，不仅仅需要双方的应用程序遵守约定，中间还不能出任何差错。<br/>

　　假如b进程是异常终止的，发送FIN包是OS代劳的，b进程已经不复存在，**当机器再次收到该socket的消息时，会回应RST（因为拥有该socket的进程已经终止）**。a进程对收到RST的socket调用write时，操作系统会给a进程发送SIGPIPE，默认处理动作是终止进程。<br/>

通过以上的叙述，**内核通过socket的read/write将双方的连接异常通知到应用层**，虽然**很不直观**，似乎也够用。<br/>

🔥总结

感慨：在写TCP/IP通信时，似乎没怎么考虑连接的终止或错误，只是在read/write错误返回时关闭socket，程序似乎也能正常运行，但某些情况下总是会出奇怪的问题。想完美处理各种错误，却发现怎么也做不对。<br/>

原因之一是：**socket（或者说TCP/IP栈本身）对错误的反馈能力是有限的。**<br/>

考虑这样的错误情况：<br/>

不同于b进程退出（此时OS会负责为所有打开的socket发送FIN包），当B机器的**OS崩溃**（注意不同于**人为关机**，因为关机时所有进程的退出动作依然能够得到保证）/**主机断电**/**网络不可达时**，a进程根本不会收到FIN包作为连接终止的提示。<br/>

如果a进程阻塞在read上，那么结果只能是永远的等待。<br/>

如果a进程先write然后阻塞在read，由于收不到B机器TCP/IP栈的ack，TCP会持续重传12次（时间跨度大约为9分钟），然后在阻塞的read调用上返回错误：ETIMEDOUT/EHOSTUNREACH/ENETUNREACH <br/>

假如B机器恰好在某个时候恢复和A机器的通路，并收到a某个重传的pack，因为**不能识别所以会返回一个RST**，此时a进程上阻塞的read调用会返回错误ECONNREST <br/>

socket对这些错误还是有一定的反馈能力的，前提是在**对面不可达时你依然做了一次write调用，而不是轮询或是阻塞在read上**，那么总是会在重传的周期内检测出错误。如果没有那次write调用，应用层永远不会收到连接错误的通知。<br/>

write的错误最终通过read来通知应用层。<br/>

****

##### 仅仅通过read/write来检测异常情况是不靠谱的，还需要一些额外的工作：

1. 使用TCP的KEEPALIVE功能？<br/>

   ```shell
   cat /proc/sys/net/ipv4/tcp_keepalive_time
   cat /proc/sys/net/ipv4/tcp_keepalive_intvl
   cat /proc/sys/net/ipv4/tcp_keepalive_probes
   ```

   以上参数的大致意思是：keepalive routine每2小时（7200秒）启动一次，发送第一个probe（探测包），如果在75秒内没有收到对方应答则重发probe，当连续9个probe没有被应答时，认为连接已断。（此时read调用应该能够返回错误，待测试）<br/>

   但在我印象中keepalive不太好用，默认的时间间隔太长，又是整个TCP/IP栈的全局参数：修改会影响其他进程，Linux的下似乎可以修改per socket的keepalive参数？（希望有使用经验的人能够指点一下），但是这些方法不是portable的。<br/>

2. 进行应用层的心跳<br/>

   严格的网络程序中，应用层的心跳协议是必不可少的。虽然比TCP自带的keep alive要麻烦不少，但有其最大的优点：可控。<br/>

3. 也可以简单一点，针对连接做timeout，关闭一段时间没有通信的”空闲“连接。

   [Muduo 网络编程示例之八：Timing wheel 踢掉空闲连接](http://www.cnblogs.com/Solstice/archive/2011/05/04/2036983.html) by 陈硕

整段文字转载于：https://www.cnblogs.com/xiehongfeng100/p/4619451.html

****



```c++
int client_closefun(int epoll, int fd, timer_link* timer) {
    timer->remote_timer(gFdProcess[fd]);

    gFdProcess[fd]->init();

    ::close(fd);
    return 1;
}
```



```c++
int client_timeoutfun(int epoll, int fd, timer_link* timers, time_value tnow) {

    int len = presend - gFdProcess[fd]->m_sended;
    len = write(fd, &(senddata[0]) + gFdProcess[fd]->m_sended, len);
    if (len == 0) {
        gFdProcess[fd]->m_activeclose = true;
    } else if (len > 0) {
        gFdProcess[fd]->m_sended += len;
        if (gFdProcess[fd]->m_sended == presend) {
            gFdProcess[fd]->m_sended = 0;
        }
    }
    timers->add_timer(gFdProcess[fd], tnow + 10 * 1000);
    return 0;
}
```









[目录](#目录)

****

##### worker.cc

```c++
int process_order(int epollfd, char *buff, int &buffindex,
                  timer_link *timerlink) {
    unsigned int processed = 0;
    struct epoll_event ev = {0};

    while (true) {

        time_value tnow = get_now();
        if (int(processed + sizeof(int)) <= buffindex) {
            int *p = (int *)(buff + processed);
            processed += sizeof(int);
            ev.events = EPOLLIN | EPOLLET;
            ev.data.fd = *p;
            set_noblock(*p);

            gFdProcess[*p]->m_readfun = client_readfun;
            gFdProcess[*p]->m_writefun = client_writefun;
            gFdProcess[*p]->m_closefun = client_closefun;
            gFdProcess[*p]->m_timeoutfun = client_timeoutfun;
            if (epoll_ctl(epollfd, EPOLL_CTL_ADD, *p, &ev) == -1) {
                perror("epoll_ctl: listen_sock");
                exit(-1);
            } else {
                timerlink->add_timer(gFdProcess[*p], tnow + 10 * 1000);
            }
        } else {
            break;
        }
    }

    if (int(processed) == buffindex) {
        buffindex = 0;
    } else {
        memmove(buff + processed, buff, buffindex - processed);
        buffindex = processed;
    }
    return 0;
}

```



```c++
int order_readfun(int epollfd, int orderfd, timer_link *timerlink) {
    assert(epollfd != 0);
    assert(orderfd != 0);
    char buff[10 * 1024] = {0};
    int buffindex = 0;
    int canread = 0;
    while (true) {
        canread = sizeof(buff) - buffindex;

        int len = read(orderfd, buff + buffindex, canread);
        if (len > 0 && ((unsigned int)(buffindex + len) >= sizeof(int))) {
            buffindex += len;
            process_order(epollfd, buff, buffindex, timerlink);
        } else if (len == -1 && errno == EAGAIN) {
            break;
        } else {
        }
    }
    return 1;
}
```



```c++
void *worker_thread(void *arg) {

    struct worker_thread_arg *pagr = (struct worker_thread_arg *)arg;

    struct epoll_event *m_events;

    int epollfd;
    assert(pagr);
    struct epoll_event ev = {0};

    ev.events = EPOLLIN | EPOLLET;
    ev.data.fd = pagr->orderfd;
    set_noblock(pagr->orderfd);

    m_events = (struct epoll_event *)malloc(MAXEPOLLEVENT *
                                            sizeof(struct epoll_event));
    epollfd = epoll_create(MAXEPOLLEVENT);
    gFdProcess[pagr->orderfd]->m_readfun = order_readfun;
    if (epoll_ctl(epollfd, EPOLL_CTL_ADD, pagr->orderfd, &ev) == -1) {
        perror("epoll_ctl: listen_sock");
        exit(-1);
    } else {
    }

    timer_link global_timer;
    time_value outtime = 1000;
    while (true) {
        process_event(epollfd, m_events, outtime, &global_timer);
        if (global_timer.get_arg_time_size() > 0) {
            outtime = global_timer.get_mintimer();
        } else {
            outtime = 1000;
        }
    }
}

```





[目录](#目录)

****

##### main_event.h

```c++
void process_one_event(int epollfd, struct epoll_event *m_events, int timeout,
                       timer_link *timer);
```

[目录](#目录)

****

##### main_event.cc

```c++
void process_one_event(int epollfd, struct epoll_event *m_events,
                       timer_link *timerlink) {
    int fd = m_events->data.fd;

    if ((!gFdProcess[fd]->m_activeclose) && (m_events->events & EPOLLIN)) {
        if (gFdProcess[fd]->m_readfun) {
            if (0 > gFdProcess[fd]->m_readfun(epollfd, fd, timerlink)) {
                gFdProcess[fd]->m_activeclose = true;
            }
        }
    }

    if ((!gFdProcess[fd]->m_activeclose) && (m_events->events & EPOLLOUT)) {
        if (gFdProcess[fd]->m_writefun) {
            if (0 > gFdProcess[fd]->m_writefun(epollfd, fd, timerlink)) {
                gFdProcess[fd]->m_activeclose = true;
            }
        }
    }

    if ((!gFdProcess[fd]->m_activeclose) && (m_events->events & EPOLLRDHUP)) {
        printf("events EPOLLRDHUP \n");
        gFdProcess[fd]->m_activeclose = true;
    }

    if (gFdProcess[fd]->m_activeclose) {
        timerlink->remote_timer(&(gFdProcess[fd]));
        gFdProcess[fd]->m_closefun(epollfd, fd, timerlink);
    }
}
```



```c++
void process_event(int epollfd, struct epoll_event *m_events,
                   time_value timeout, timer_link *timers) {

    int m_eventsize = epoll_wait(epollfd, m_events, MAXEPOLLEVENT, timeout);
    if (m_eventsize > 0) {
        for (int i = 0; i < m_eventsize; ++i) {
            process_one_event(epollfd, m_events + i, timers);
        }
    } else {
    }

    time_value tnow = get_now();
    // timer
    while (true) {

        void *point = timers->get_timer(tnow);
        if (!point) {
            break;
        }
        FdProcess *p = (FdProcess *)point;
        p->m_timeoutfun(epollfd, p->fd, timers, tnow);
        //		timers->add
    }
}

```







[目录](#目录)

****

##### main.cc























