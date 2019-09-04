# Note

Code: https://github.com/xiaojiaqi/C1000kPracticeGuide <br/>

Document: https://github.com/xiaojiaqi/C1000kPracticeGuide/blob/master/docs/cn/c1000K.pdf

## 准备阶段

### 概念

C1000K: 一台服务器为100万连接提供服务

### 目标

在一台不通物理机器上实现100万连接，服务器每10秒向所有的客户端发送500字节长度的消息。要求稳定支持连接，发送数据稳定，客户端接收正常。

### 系统准备

1. 查看服务器信息

   `sudo dmidecode | grep "Product Name"` 

2. 检查cpu规格

   `cat /proc/cpuinfo | grep name | awk -F: '{print $(NF)}' | uniq -c` <br/>

   ```sh
   #output
   40  Intel(R) Xeon(R) CPU E5-2670 v2 @ 2.50GHz
   #一块 40核的E5-2670
   ```

   `cat /proc/cpuinfo | grep physical | grep -v address | uniq -c` <br/>

   ```sh
   #output
   10 physical id	: 0
   10 physical id	: 1
   10 physical id	: 0
   10 physical id	: 1
   ```

   是物理硬核的CPU

3. 检查内存

   `cat /proc/meminfo` <br/>

4. 网卡

   `dmesg | grep -i eth` <br/>

5. 查看操作系统

   安装命令centos7: `sudo yum install redhat-lsb-core-y` <br/>

   `lsb_release -a` <br/>

6. 内核版本

   `uname -a` <br/>

7. c++

   `g++ --version`  4.6.3<br/>

8. go

   `go version` <br/>

### 系统优化

系统优化是非常关键的一步，因为默认的系统设置并不是以高并发高负载的服务器做为目标的，所有配置需要进行修改，这样可以尽可能的使用操作系统和硬件的能力。

##### 根据高并发高负载业务要求对Linux系统设置进行优化

1. 提高文件数目上限

   在 Linux 中 socket 被表示为一个文件描述符，默认的文件数目上限是1024，因此需要提高文件打开上限。<br/>

   `ulimit -a` 查看，其中 `open files ` 的数值为服务器可以提供服务的用户数目。<br/>

   ****

   提高打开文件数目上限<br/>

   - 修改 /etc/security/limits.conf，并添加信息：

     ```sh
     * hard nofile 1025500
     * soft nofile 1025500
     ```

   - 修改/etc/sysctl.conf

     ```sh
     fs.file-max=1025500
     ```

   - 运行 sysctl –p 使修改生效

   - 重新登录并确认修改生效

   ****

2. TCP/IP优化

   TCP/IP 的优化选项非常的多，而且 Linux 提供了非常友好的修改方法，使 用起来非常方便。但是对于 TCP/IP 各个字段的意义，以及可能产生的后 果，需要对 TCP/IP 本身有一个较为深入的理解。<br/>

3. d

##### 针对硬件进行优化



## 编码

介绍：https://www.infoq.cn/article/weixin-bonus-load <br/>

### 思考&需要深入了解的

1. 网络模型都有哪些？优劣情况？常用的模型？

   阻塞的网络I/O，多线程加阻塞I/O，非阻塞轮询，多路复用（epoll:比较主流，主要问题是将逻辑碎片化，成为程序设计的难题），异步I/O，纤程

2. 关注go的并发及协程



### c++实现

使用epoll，pipe，cpu亲缘性

link: 

### Go实现

**关注go的并发及协程** <br/>

```go
package main

import (
	"fmt"
	"net"
	"time"
)

var array []byte = make([]byte, 500)

func checkError(err error, info string) (res bool) {

	if err != nil {
		fmt.Println(info + "  " + err.Error())
		return false
	}
	return true
}

func Handler(conn net.Conn) {
//	fmt.Println("connection is connected from ...", conn.RemoteAddr().String())
	for {
		_, err := conn.Write(array) //服务器每10秒向所有的客户端发送500字节长度的消息
		if err != nil {
			return
		}
		time.Sleep(10 * time.Second)
	}
}

func main() {

	for i := 0; i < 500; i += 1 {
		array[i] = 'a'
	}

	service := ":8888"
	tcpAddr, _ := net.ResolveTCPAddr("tcp4", service)
	l, _ := net.ListenTCP("tcp", tcpAddr)

	for { //一直循环
    //fmt.Println("Listening ...")
		conn, _ := l.Accept()
		//fmt.Println("Accepting ...")
		go Handler(conn) //此处使用go关键字新建线程处理连接，实现并发
	}

}
//上述代码没有关闭连接的过程，因为是模拟，主要在于可以连接百万个用户。正常情况下，连接处理后，要主动关闭，释放资源。
```

语法：<br/>

1. make https://www.cnblogs.com/hustcat/p/4004889.html

   | make                             | new                                    |
   | -------------------------------- | -------------------------------------- |
   | 返回值是一个类型的引用           | 返回值是一个指针                       |
   | 分配空间并初始化                 | 仅分配空间，不初始化内存，只是将其置零 |
   | 只能是三种对象：slice，map，chan | ---                                    |
   | var v []int = make([]int, 10)    | var p *[]int = new([]int)              |
   | 内建函数                         | 内建函数                               |

   注：<br/>

   1. Slice, Map 和 Channel这三种类型实质上是**对在使用前必须进行初始化的数据结构**的**引用**。 例如, Slice是一个 具有三项内容的描述符，包括 指向数据（在一个数组内部）的指针，长度以及容量。**在这三项内容被初始化之前，Slice的值为nil**。使用 make 之后 slice 是一个初始化的 slice，即 slice 的长度、容量、底层指向的 array 都被 make 完成初始化，此时 **slice 内容被类型 int 的零值填充** ，形式是 [0 0 0]。对于Slice，Map和Channel，make()函数初始化了其内部的数据结构，并且准备了将要使用的值。
   2. 在Go语言中，如果一个局部变量在函数返回后仍然被使用，这个变量会从heap，而不是stack中分配内存。因此，**golang 中的函数可以返回局部变量**。
   3. 如果不特殊声明，**go 的函数默认都是按值传参**，即通过函数传递的参数是值的副本，在函数内部对值修改不影响值的本身，但是 make(T, args) 返回的值通过函数传递参数之后可以直接修改，即 map，slice，channel 通过函数穿参之后在函数内部修改将影响函数外部的值。说明 make(T, args) 返回的是**引用类型**。

2. go TCP编程

   | server                                                       |                                                              |
   | ------------------------------------------------------------ | ------------------------------------------------------------ |
   | func **ResolveTCPAddr**(net, addr string) (*TCPAddr, os.Error) | 获取一个`TCPAddr`，参数都是`string`类型，net是个`const string`,包括`tcp4`,`tcp6`,`tcp`一般使用`tcp`，兼容v4和v6，`addr`表示ip地址，包括端口号，如`www.google.com:80`之类 |
   | func **ListenTCP**(net string, laddr *TCPAddr) (l *TCPListener, err os.Error) | 用来监听端口，`net`表示协议类型，`laddr`表示本机地址,是`TCPAddr`类型，注意，此处的laddr包括端口，返回一个`*TCPListener`类型或者错误 |
   | func (l *TCPListener) **Accept**() (c Conn, err os.Error)    | 用来返回一个新的连接，进行后续操作，这是`TCPListener`的方法，一般`TCPListener`从上一个函数返回得来。 |
   | **client**                                                   |                                                              |
   | func **DialTCP**(net string, laddr, raddr *TCPAddr) (c *TCPConn, err os.Error) | 用来连接(connect)到远程服务器上，net表示协议方式，`tcp`,`tcp4`或者`tcp6`，`laddr`表示本机地址，一般为`nil`，`raddr`表示远程地址，这里的`laddr`和`raddr`都是`TCPAddr`类型的，一般是上一个函数的返回值。 |
   | conn.**RemoteAddr**()                                        | 获取连接对象的IP地址                                         |
   | **TCP有很多连接控制函数**                                    |                                                              |
   | func **DialTimeout**(net, addr string, timeout time.Duration) (Conn, error) | 设置建立连接的超时时间，客户端和服务器端都适用，当超过设置时间时，连接自动关闭。 |
   | func (c *TCPConn) **SetReadDeadline**(t time.Time) error<br/>func (c *TCPConn) **SetWriteDeadline**(t time.Time) error | 用来设置写入/读取一个连接的超时时间。当超过设置时间时，连接自动关闭。 |
   | func (c *TCPConn) **SetKeepAlive**(keepalive bool) os.Error  | 设置客户端是否和服务器端保持长连接，可以降低建立TCP连接时的握手开销，对于一些需要频繁交换数据的应用场景比较适用。 |

   ```go
   //Go语言包中处理UDP Socket和TCP Socket不同的地方就是在服务器端处理多个客户端请求数据包的方式不同,UDP缺少了对客户端连接请求的Accept函数。其他基本几乎一模一样，只是TCP换成了UDP而已。UDP的几个主要函数如下所示：
   func ResolveUDPAddr(net, addr string) (*UDPAddr, os.Error)
   func DialUDP(net string, laddr, raddr *UDPAddr) (c *UDPConn, err os.Error)
   func ListenUDP(net string, laddr *UDPAddr) (c *UDPConn, err os.Error)
   func (c *UDPConn) ReadFromUDP(b []byte) (n int, addr *UDPAddr, err os.Error
   func (c *UDPConn) WriteToUDP(b []byte, addr *UDPAddr) (n int, err os.Error)
   //TCP与UDP的conn对象是不同的，前者是net.Conn，后者是net.UDPConn。另外UDP的服务器端的读写是要带着地址的，这个地址是客户端的地址。
   ```



### 两个版本的对比

