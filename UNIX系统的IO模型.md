#UNIX系统的IO模型

目录
*    I/O模型
*    阻塞式I/O模型
*    非阻塞式I/O模型
*    I/O复用模型
*    信号驱动式I/O模型
*    异步I/O模型
*    各种I/O模型的比较
*    同步I/O和异步I/O对比

**1、I/O模型**

I/O模型分：
_阻塞式I/O、非阻塞式I/O、I/O复用、信号驱动式I/O、异步I/O_；

一个输入操作通常包括两个不同的阶段：
*    1)等待数据准备好；
*    2)从内核向进程复制数据；
对于一个套接字上的输入操作，第一步通常涉及等待数据从网络中到达。当所等待分组到达时，它被复制到内核中的某个缓冲区。第二步就是把数据从内核缓冲区复制到应用进程缓冲区。

**阻塞式I/O模型**

最流行的I/O模型是阻塞式I/O（blocking I/O）模型，默认情况下，所有套接字都是阻塞的。以数据报套接字作为例子，我们有如下图所示的情形。

![阻塞式I/O模型](https://raw.githubusercontent.com/hongyuanlei/papersite/master/image/01.jpg)

在上图中，进程调用recvfrom，其系统调用直到数据报去到达且被复制到应用进程的缓冲区中或者发生错误才返回。我们说进程在从调用recvfrom开始到它返回的整段时间内是被阻塞的。recvfrom成功返回后，应用进程开始处理数据报。

**非阻塞式I/O模型**

进程把一个套接字设置成非阻塞是在通知内核：当所请求的I/O操作非得把本进程投入睡眠才能完成时，不要把本进程设入睡眠，而是返回一个错误。非阻塞式I/O模型如下图：

![非阻塞式I/O模型](https://raw.githubusercontent.com/hongyuanlei/papersite/master/image/0002.jpg)

前三次调用recvfrom时没有数据可返回，因此内核转而立即返回一个EWOULBLOCK错误。第四次调用recvfrom时已经有一个数据报准备好，它被复制到应用进程缓冲区，于是recvfrom成功返回。我们接着处理数据。

**I/O复用模型**

有了I/O复用（I/O multiplexing），我们就可以调用select或poll，阻塞在这两个系统调用中的某一个上，而不是阻塞在真正的I/O系统调用上。下图：

![I/O复用模型](https://raw.githubusercontent.com/hongyuanlei/papersite/master/image/03.jpg)

我们阻塞于select调用，等待数据报套接字变为可读。当select返回套接字可读这一条件时，我们调用recvfrom把所读数据报复制到应用进程缓冲区。

和阻塞式I/O模型比较，I/O复用并不显得有什么优势，事实上由于使用select需要两个而不是单个系统调用，I/O复用还稍有劣势。使用select的优势在于我们可以等待多个描述符就绪。

与I/O复用密切相关的另一种I/O模型是在多线程中使用阻塞式I/O（我们经常这么干）。这种模型与上述模型极为相似，但它并没有使用select阻塞在多个文件描述符上，而是使用多个线程（每个文件描述符一个线程），这样每个线程都可以自由的调用recvfrom之类的阻塞式I/O系统调用了。


**信号驱动式I/O模型**

我们也可以用信号，让内核在描述符就绪时发送SIGIO信号通知我们。我们称这种模型为信号驱动式I/O(signal-driven I/O)。 

![信号驱动式I/O模型](https://raw.githubusercontent.com/hongyuanlei/papersite/master/image/04.jpg)

我们首先开启套接字的信号驱动式I/O功能，并通过sigaction系统调用安装一个信号处理函数。改系统调用将立即返回，我们的进程继续工作，也就是说他没有被阻塞。当数据报准备好读取时，内核就为该进程产生一个SIGIO信号。我们随后就可以在信号处理函数中调用recvfrom读取数据报，并通知主循环数据已经准备好待处理，也可以立即通知主循环，让它读取数据报。

无论如何处理SIGIO信号，这种模型的优势在于等待数据报到达期间进程不被阻塞。主循环可以继续执行，只要等到来自信号处理函数的通知：既可以是数据已准备好被处理，也可以是数据报已准备好被读取。

**异步I/O模型**

异步I/O（asynchronous I/O）由POSIX规范定义。演变成当前POSIX规范的各种早起标准所定义的实时函数中存在的差异已经取得一致。一般地说，这些函数的工作机制是：告知内核启动某个操作，并让内核在整个操作（包括将数据从内核复制到我们自己的缓冲区）完成后通知我们。这种模型与前一节介绍的信号驱动模型的主要区别在于：信号驱动式I/O是由内核通知我们何时可以启动一个I/O操作，而异步I/O模型是由内核通知我们I/O操作何时完成。

![异步I/O模型](https://raw.githubusercontent.com/hongyuanlei/papersite/master/image/05.jpg)

我们调用aio_read函数（POSIX异步I/O函数以aio_或lio_开头），给内核传递描述符、缓冲区指针、缓冲区大小（与read相同的三个参数）和文件偏移（与lseek类似），并告诉内核当整个操作完成时如何通知我们。该系统调用立即返回，并且在等待I/O完成期间，我们的进程不被阻塞。本例子中我们假设要求内核在操作完成时产生某个信号。改信号直到数据已复制到应用进程缓冲区才产生，这一点不同于信号驱动I/O模型。

**各种I/O模型的比较**

对比了上述5中不同的I/O模型。可以看出，前4中模型的主要区别在于第一阶段，因为他们的第二阶段是一样的：在数据从内核复制到调用者的缓冲区期间，进程阻塞于recvfrom调用。相反，异步I/O模型在这两个阶段都要处理，从而不同于其他4中模型。

![各种I/O模型的比较](https://raw.githubusercontent.com/hongyuanlei/papersite/master/image/006.jpg)

**同步I/O和异步I/O对比**

POSIX把这两个术语定于如下：

同步I/O操作（sysnchronous I/O opetation）导致请求进程阻塞，直到I/O操作完成；

异步I/O操作（asynchronous I/O opetation）不导致请求进程阻塞。

根据上述定义，我们的前4种模型----阻塞式I/O模型、非阻塞式I/O模型、I/O复用模型和信号驱动I/O模型都是同步I/O模型，因为其中真正的I/O操作（recvfrom）将阻塞进程。只有异步I/O模型与POSIX定义的异步I/O相匹配。

[原文](http://my.oschina.net/shenxueliang/blog/159510)
