C1000kPracticeGuide
===================

Document:
   https://github.com/xiaojiaqi/C1000kPracticeGuide/blob/master/docs/cn/c1000K.pdf

 

**For details, please refer to the original link. ** <br/>

https://github.com/xiaojiaqi/C1000kPracticeGuide 



## Cplus version

目录

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

****





























