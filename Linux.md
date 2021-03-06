## Linux基础



### Linux五种I/O模型

Linux I/O分为两个阶段：等待数据准备完毕，把数据从内核态拷贝到用户态。

Unix环境下有五种I/O模型：

* 阻塞式 I/O
* 非阻塞式 I/O
* I/O 复用（select 和 poll）
* 信号驱动式 I/O（SIGIO）
* 异步 I/O（AIO）

#### 阻塞式I/O

进程系统调用直到数据到达应用的缓冲区或者发生错误才返回。

<div align="center"><img src="images/1492928416812_4.png">
</div>

#### 非阻塞式I/O

如果系统调用操作不能立即完成，不要让进程睡眠，而是返回一个错误码。应用进程可以继续执行，但是需要循环执行系统调用确定系统调用是否完成，这个操作又称为轮询。

进程持续轮询内核，执行大量的系统调用，CPU利用率较低。

<div align="center"><img src="images/1492929000361_5.png">
</div>

#### I/O多路复用

如果一个进程需要处理多个文件描述符，可以调用select或者poll，进程阻塞于这两个系统调用之上，而不是阻塞于真正的I/O系统调用上。当某一个描述符可用时返回，并把数据从内核拷贝到用户缓冲区中。

使用select/poll的优势在于可以等待多个描述符就绪，又被称为 Event Driven I/O，即事件驱动 I/O。

相较于每个描述符都需要打开一个线程去处理的方式，I/O多路复用不需要创建大量的线程，较少了线程创建和切换的开销。

<div align="center"><img src="images/1492929444818_6.png">
</div>

#### 信号驱动I/O

使用信号，应用进程使用sigaction系统调用注册一个信号处理函数，系统调用立即返回，进程继续工作，内核在数据到达时向应用进程发送SIGIO信号，收到信号后，进程可以在信号处理函数中数据从内核复制到应用进程中。

这种方式的优势在于等待数据到达的过程期间不被阻塞，主循环可以继续执行。

<div align="center"><img src="images/1492929553651_7.png">
</div>

#### 异步I/O

告知内核某个操作，并且让内核在整个操作（包括将数据从内核复制到进程的缓冲区）完成后应用进程发送信号。

和信号驱动I/O的区别：信号驱动I/O通过信号通知进程什么时候可以开始I/O操作，异步I/O通过信号通知I/O操作已完成。

<div align="center"><img src="images/1492930243286_8.png">
</div>

#### I/O模型比较

* 同步I/O：将数据从内核复制到应用缓冲区（第二阶段），进程会阻塞。
* 异步I/O：第二阶段进程不会阻塞。

同步 I/O 包括阻塞式 I/O、非阻塞式 I/O、I/O 复用和信号驱动 I/O ，它们的主要区别在第一个阶段。

非阻塞式 I/O 、信号驱动 I/O 和异步 I/O 在第一阶段不会阻塞。

<div align="center"><img src="images/1492928105791_3.png">
</div>

### 文件系统的理解（EXT4，XFS，BTRFS）



### 文件处理命令

g/re/p（globally search a regular expression and print)：使用正则表示式进行全局查找并打印。

```shell
$ grep [-acinv] [--color=auto] 搜寻字符串 filename
-c ： 统计匹配到行的个数
-i ： 忽略大小写
-n ： 输出行号
-v ： 反向选择，也就是显示出没有 搜寻字符串 内容的那一行
--color=auto ：找到的关键字加颜色显示

```

awk 每次处理一行，处理的最小单位是字段，每个字段的命名方式为：\$n，n 为字段号，从 1 开始，$0 表示一整行。

```shell
$ awk '条件类型 1 {动作 1} 条件类型 2 {动作 2} ...' filename
```

可以根据字段的某些条件进行匹配。

awk变量：

| 变量名称 |           代表意义           |
| :------: | :--------------------------: |
|    NF    |     每一行拥有的字段总数     |
|    NR    |   目前所处理的是第几行数据   |
|    FS    | 目前的分隔字符，默认是空格键 |

### IO复用的三种方法（select,poll,epoll）深入理解，包括三者区别，内部原理实现？

看博客：

#### select

```C
int select(int n, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout);
```

select 允许应用程序监视一组文件描述符，等待一个或者多个描述符成为就绪状态，从而完成 I/O 操作。

- fd_set 使用数组实现，数组大小使用 FD_SETSIZE 定义，所以只能监听少于 FD_SETSIZE 数量的描述符。有三种类型的描述符类型：readset、writeset、exceptset，分别对应读、写、异常条件的描述符集合。
- timeout 为超时参数，调用 select 会一直阻塞直到有描述符的事件到达或者等待的时间超过 timeout。
- 成功调用返回结果大于 0，出错返回结果为 -1，超时返回结果为 0。

select存在以下的问题：

- 每次调用，程序需要拷贝一份包含所有指定的文件描述符的数据结构到内核中。当检查大量文件描述符时，拷贝操作将会占用大量的 CPU 时间。
- 每次调用，内核必须检查所有的文件描述符，是否处于就绪状态。如果文件描述符过多，该操作会消耗大量时间。
- 程序必须检查返回的数据结构中的每个文件描述符，如果文件描述符过多，这个操作会耗费大量的时间。
- select 使用的数据结构 fd_set 对于被检查的文件描述符有一个上限（FD_SETSIZE），在 Linux 下的默认值是 1024。

#### poll

poll 的功能与 select 类似，也是等待一组描述符中的一个成为就绪状态，但是采用链表的方式组织所有的文件描述符，没有最大监听文件描述符的限制。

```C
int poll(struct pollfd *fds, unsigned int nfds, int timeout);
```

select 和 poll 的功能基本相同，不过在一些实现细节上有所不同。

- select 会修改描述符，而 poll 不会；
- select 的描述符类型使用数组实现，FD_SETSIZE 大小默认为 1024，因此默认只能监听少于 1024 个描述符。如果要监听更多描述符的话，需要修改 FD_SETSIZE 之后重新编译；而 poll 没有描述符数量的限制；
- poll 提供了更多的事件类型，并且对描述符的重复利用上比 select 高。
- 如果一个线程对某个描述符调用了 select 或者 poll，另一个线程关闭了该描述符，会导致调用结果不确定。

select 和 poll 速度都比较慢，每次调用都需要将全部描述符从应用进程缓冲区复制到内核缓冲区。select的可移植性较强。

#### epoll

```c
int epoll_create(int size);
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event)；
int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);
```

epoll把所有注册到内核中的文件描述符用一个红黑树维护起来，通过回调函数内核会将 I/O 准备好的描述符加入到一个链表中管理，进程调用 epoll_wait() 便可以得到事件完成的描述符。

epoll仅需要将描述符从进程缓冲区向内核缓冲区拷贝一次，并且进程不需要通过轮询来获得事件完成的描述符，比 select 和 poll 更加灵活而且没有描述符数量限制。

epoll仅适用于Linux系统中。

#### 使用场景

select的timeout参数为微秒，而 poll 和 epoll 为毫秒，因此 select 更加适用于实时性要求比较高的场景。同时，select可移植性强，如果有跨平台的需求可以考虑select。

poll 没有最大描述符数量的限制，如果平台支持并且对实时性要求不高，应该使用 poll 而不是 select。

epoll用于只在Linux上运行，并且并发连接数很高但是活跃连接比例不高的场景，因此更适用于长连接场景。如果连接数量不多，拷贝的文件描述符不多，不能体现出epoll的优势。如果活跃连接数多，变化频繁，并且连接都是短暂的，也不适用于epoll。因为epoll 中的所有描述符都存储在内核中，造成每次需要对描述符的状态改变都需要通过 epoll_ctl 进行系统调用，频繁系统调用降低效率。

### Epoll的ET模式和LT模式（ET的非阻塞）

* ET是边缘模式。在这个模式下，只有当描述符从未就绪变为就绪状态，内核才回通过epoll进行通知，直到下一次描述符就绪，不会再次重复通知。优点就是只通知一次，减少内核资源浪费，效率高。缺点就是如果有些数据来不及读可能就会无法取出。
* LT是水平触发模式。在这个模式下，如果文件描述符就绪，内核就会进行通知，只要描述符一直处于就绪状态，内核就会一直通知。优点在于使用简单，不容易出错。缺点是内核会一直通知，会不停从内核空间切换到用户空间，造成时间的浪费。

源码角度：

如果采用LT模式，当前的epoll_item对象被重新加到eventpoll的就绪列表中，在下一次 epoll_wait调用时，这些epoll_item对象就会被重新处理。如果事件没有被消费，会被再次触发，通知应用程序消费；如果事件已经消费过了，直接从就绪队列移除就可以了。

如果采用ET模式，epoll_item直接被移除，除非监听件描述符再次触发。‘

参考链接（源码分析）：https://www.nowcoder.com/discuss/26226

### 查询进程占用CPU的命令（注意要了解到used，buf，cache代表意义）



### linux的其他常见命令（kill，find，cp等等）

find用于查找文件





### shell脚本用法



### 硬链接和软链接的区别





### 文件权限怎么看（rwx）

用户分为三种：文件拥有者、群组以及其它人，对不同的用户有不同的文件权限。

常见的文件类型及其含义有：

- d：目录
- -：文件
- l：链接文件

9 位的文件权限字段中，每 3 个为一组，共 3 组，每一组分别代表对文件拥有者、所属群组以及其它人的文件权限。一组权限中的 3 位分别为 r、w、x 权限，表示可读、可写、可执行。

文件时间有以下三种：

- modification time (mtime)：文件的内容更新就会更新；
- status time (ctime)：文件的状态（权限、属性）更新就会更新；
- access time (atime)：读取文件时就会更新。

### Linux监控网络带宽的命令，查看特定进程的占用网络资源情况命令



### 怎么修改一个文件的权限

chmod 777  (177 277 477 等，权限组合是 1 2 4，分别代表r x w )

