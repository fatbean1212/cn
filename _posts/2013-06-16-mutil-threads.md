---
layout: post
title: "linux下多线程编程"
description: ""
category: 程序语言
tags: [pthread, mutex, semophore, 多线程]
---

最近研究mysql源码，各种锁，各种互斥，好在我去年认真学了《unix环境高级编程》，
虽然已经忘得差不多了，但是学过始终是学过，拿起来也快。写这篇文章的目的就是总结linux
下多线程编程，作为日后的参考资料。

本文将介绍linux系统下多线程编程中，线程同步的各种方法。包括：

1. <a href="#1">互斥量(mutex)</a>
2. <a href="#2">读写锁</a>
3. <a href="#3">条件变量</a>
4. <a href="#4">信号量</a>
5. <a href="#5">文件互斥</a>

在介绍不同的线程同步的方法之前，先简单的介绍一下进程和线程的概念，
它们的优缺点，线程相关的API，读者——写者问题和哲学家就餐问题。

#基础知识

###1. 进程和线程

进程（process）是指在系统中正在运行的一个应用程序，是系统资源分配的基本单位，在内存
中有其完备的数据空间和代码空间，拥有完整的虚拟空间地址。一个进程所拥有的数据和变量只
属于它自己。

线程（thread）是进程内相对独立的可执行单元，所以也被称为轻量进程（lightweight processes
）；是操作系统进行任务调度的基本单元。它与父进程的其它线程共享该进程所拥有的全部代码
空间和全局变量，但拥有独立的堆栈（即局部变量对于线程来说是私有的）。

####1.1 线程和进程的区别：

1. 线程是进程的一部分，所以线程有的时候被称为是轻量级进程。
2. 一个没有线程的进程是可以被看作单线程的，如果一个进程内拥有多个线程，
进程的执行过程不是一条线（线程）的，而是多条线（线程）共同完成的。
3. 系统在运行的时候会为每个进程分配不同的内存区域，但是不会为线程分配内存（线程所使
用的资源是它所属的进程的资源），线程组只能共享资源。那就是说，除了CPU之外（线程在运
行的时候要占用CPU资源），计算机内部的软硬件资源的分配与线程无关，线程只能共享它所属
进程的资源。
4. 与进程的控制表PCB相似，线程也有自己的控制表TCB，但是TCB中所保存的线程状态比PCB表
中少很多
5. 进程是系统所有资源分配时候的一个基本单位，拥有一个完整的虚拟空间地址，并不依赖线
程而独立存在。

更加详细的介绍，请参考[这里][1]。

####1.2 线程的优点

由于以下原因，行业内广泛地在编程库和操作系统中实现线程：

* 减少内存占用量。创建另一个线程的内存消耗量被限制为线程所需要的堆栈加上线程管理器需
要的一些簿记内存。
* 不需要采用先进的技术来访问服务器全局数据。如果数据有可能由另一个同时运行的线程修改，
则要做的一切就是使用互斥体
保护相关段。
* 创建线程所需要的时间大大小于创建进程所需要的时间，原因是不必复制堆部分（它可能很大）。
* 在线程之间进行环境切换时，内核在调度器中花的时间比在过程之间进行切换花的时间少。
这给负担很重的服务器留下了更多的cpu时间处理工作。

####1.3 线程的缺点

尽管线程在现代计算机中极具重要性，它们却有很多缺点：

* 编程错误代价惨重。如果一个线程崩溃，会使得整个服务器停止。一个坏线程可能会毁坏全局
数据，导致其他线程无法工作
* 容易产生编程错误。程序员必须不断考虑其他一些线程可能正在做引起麻烦的事情，以及如何
避免这种麻烦。需要采用额外的防范方法编制程序。
* 线程服务器在同步漏洞方面声名狼藉，这些漏洞几乎无法在测试中进行复制，却在生产期间很
不合时宜地出现。这类漏洞发生几率之所以如此高，是由于使用共享地址空间，这会产生更高程
度的线程交互。
* 有时候互斥体争用难以控制。如果在同一时间有太多的线程想得到相同的互斥体，就会导致过
度的环境切换，大量的CPU时间就会花在内核调度器上，因而几乎没有时间执行工作。
* 32位系统限制为每个线程使用4G地址空间。由于所有的线程都共享相同的地址空间，理论上整
个服务器就被限制为使用4G RAM,即便实际上有更多的物理RAM也不例外。实际使用时，地址空间
会在一个小得多的限值下开始变得非常拥挤。
* 拥挤的32位地址空间会带来另一个问题。每个线程都需要一些堆栈空间，在分配了堆栈后，
即便不使用所分配的大部分空间，服务器也会为其保留地址空间。每个新堆栈都会减少用于堆的
潜在空间。因此，即使有足够的物理内存，也不可能同时使用大型缓冲区，同时使用大量并发线
程，以及同时为每个线程提供足够的堆栈空间。

####1.4 进程的优点

线程的缺点与使用多进程的优点相对应：

* 编程错误并不致命。尽管有可能发生，但一个坏分支服务器进程并不容易中断整个服务器。
* 编程错误发生的可能性小得多。在大多数时候，程序员只需要考虑一个线程的执行，而不用受
可能的并发侵入者的打扰。
* 飘忽不定的漏洞少得多。如果漏洞出现一次，通常非常容易复制。由于各个分支进程有自己的
地址空间，它们之间并没有太多的交互。
* 在32位系统中，地址空间用完的问题并不严重。

####1.5 进程的缺点

* 内存利用不够好。当子进程发生分支时，会不必要地复制大型内存程序段。
* 需要采用特殊技术实现进程数据共享。（IPC）
* 创建进程比创建线程需要更多的内核系统开销。对性能的一个很大的打击是需要 复制父进程的
数据段。不过，Linux 在这方面的手段是执行所谓的copy-on-write 。除非子进程或父进程修改了
父进程页，否则并不真正复制父进程页。在此之前，父子进程使用相同的页。
* 进程之间的环境切换比线程之间的环境切换更为耗时，因为内核需要切换页，文件描述符表
及其他额外的内容信息。留给服务器执行实际工作的时间减少。

以上关于进程和线程的优缺点，都来自《[深入理解mysql核心技术][2]》

###2. 线程API

进程原语和线程原语的比较：

<table>
<tr>
<td>进程原语</td>
<td>线程原语</td>
<td>描述</td>
</tr>
<tr>
<td>fork</td>
<td>pthread_create</td>
<td>创建新的控制流</td>
</tr>
<tr>
<td>exit</td>
<td>pthread_exit</td>
<td>从现有的控制流退出</td>
</tr>
<tr>
<td>waitpid</td>
<td>pthread_join</td>
<td>从控制流中得到退出状态</td>
</tr>
<tr>
<td>atexit</td>
<td>pthread_cancle_push</td>
<td>注册在退出控制流时调用的函数</td>
</tr>
<tr>
<td>getpid</td>
<td>pthread_self</td>
<td>获取控制流的ID</td>
</tr>
<tr>
<td>abort</td>
<td>pthread_cancle</td>
<td>请求控制流的非正常退出</td>
</tr>
</table>


###3. 读者－写者问题

读者————写者问题是一个用信号量实现的经典进程同步问题。在系统中，一个数据集(如文件或
记录) 被几个并发进程共享，这些线程分两类，一部分只要求进行读操作，称之为“读者”；
另一类要求写或修改操作，我们称之为“写者“。一般而言，对一个数据集，为了保证数据的完整
性、正确性，允许多个读者进程同时访问，但是不允许一个写者进程同其它任何一个进程（读者
或者写者）同时访问，而这类问题就称之为"读者-写者"问题。

读者优先的算法在操作系统相关的书籍中都有介绍，这是一种最简单的解决办法： 当没有写进
程正在访问共享数据集时，读进程可以进入访问，否则必须等待。而读者优先的算法存在"饿死
写者"线程的问题：只要有读者不断到来，写者就要持久地等待，直到所有的读者都读完且没有
新的读者到来时写者才能写数据集。而在很多情况下我们需要避免"饿死写者"，故而采用写者优
先算法：

在写者优先算法中，我们要实现的目标是：

1.要让读者与写者之间、以及写者与写者之问要互斥地访同数据集；
2.在无写进程到来时各读者可同时访问数据集；
3.在读者和写者都等待时访问时写者优先．

在实现写者优先时，增加一个互斥量，用于写者优先。当有写者来时，就不在允许读者去读取数据，
等待正在读数据的读者完成以后开始写数据，以此实现写者优先。


###4. 哲学家就餐问题

[哲学家就餐问题][3]可以这样表述，假设有五位哲学家围坐在一张圆形餐桌旁，做以下两件事
情之一：吃饭，或者思考。吃东西的时候，他们就停止思考，思考的时候也停止吃东西。餐桌中
间有一大碗意大利面，每两个哲学家之间有一只餐叉。因为用一只餐叉很难吃到意大利面，所以
假设哲学家必须用两只餐叉吃东西。他们只能使用自己左右手边的那两只餐叉。哲学家就餐问题
有时也用米饭和筷子而不是意大利面和餐叉来描述，因为很明显，吃米饭必须用两根筷子。


#多线程编程示例

<a name="1"></a>
###5 互斥量(mutex)

**互斥锁创建**

有两种方法创建互斥锁，静态方式和动态方式。POSIX定义了一个宏PTHREAD_MUTEX_INITIALIZER
来静态初始化互斥锁，方法如下：

1. `pthread_mutex_t mutex=PTHREAD_MUTEX_INITIALIZER;`在LinuxThreads实现中，
pthread_mutex_t是一个结构，而PTHREAD_MUTEX_INITIALIZER则是一个结构常量。

2. 动态方式是采用pthread_mutex_init()函数来初始化互斥锁，API定义如下：

        int pthread_mutex_init(pthread_mutex_t *mutex, const pthread_mutexattr_t
                *mutexattr)

    其中mutexattr用于指定互斥锁属性（见下），如果为NULL则使用缺省属性。
    pthread_mutex_destroy ()用于注销一个互斥锁，API定义如下：

        int pthread_mutex_destroy(pthread_mutex_t *mutex)

**锁操作**

锁操作主要包括加锁pthread_mutex_lock()、解锁pthread_mutex_unlock()和测试加锁
pthread_mutex_trylock()三个，不论哪种类型的锁，都不可能被两个不同的线程同时得到，
而必须等待解锁。对于普通锁和适应锁类型，解锁者可以是同进程内任何线程；
而检错锁则必须由加锁者解锁才有效，否则返回EPERM；对于嵌套锁，文档和实现要求必须由
加锁者解锁，但实验结果表明并没有这种限制，这个不同目前还没有得到解释。在同一进程中
的线程，如果加锁后没有解锁，则任何其他线程都无法再获得锁。

    int pthread_mutex_lock(pthread_mutex_t *mutex)
    int pthread_mutex_unlock(pthread_mutex_t *mutex)
    int pthread_mutex_trylock(pthread_mutex_t *mutex)

pthread_mutex_trylock() 语义与pthread_mutex_lock()类似，不同的是在锁已经被占据时返回
EBUSY而不是挂起等待。

**互斥量实现读者写者问题**

	#include <pthread.h>
	#include <signal.h>
	#include "apue.h"
	#define N 5 //No. of reader
	#define M 5 //No. of reading and writing

	pthread_mutex_t rd = PTHREAD_MUTEX_INITIALIZER; // it's mean reader can reading
	pthread_mutex_t wr = PTHREAD_MUTEX_INITIALIZER; //it's mean writer can writing

	int readCount = 0;

	void* reader(void *arg)
	{
	    int n = M;
	    int id = (int)arg;
	    while (n--)
	    {
	        sleep( rand() % 3);
	        pthread_mutex_lock(&rd);
	        readCount++;
	        if( readCount == 1)
	        {
	            pthread_mutex_lock(&wr);
	        }
	        pthread_mutex_unlock(&rd);

	        printf("reader %d is reading\n", id);
	        sleep( rand() % 3);

	        pthread_mutex_lock(&rd);
	        readCount--;
	        if (readCount == 0)
	        {
	            pthread_mutex_unlock(&wr);
	        }
	        pthread_mutex_unlock(&rd);
	        printf("reader %d is leaving\n", id);
	    }
	    printf("----reader %d has done----\n", (int)arg);
	}

	void* writer(void *arg)
	{
	    int n = M;
	    while (n--)
	    {
	        sleep( rand() % 3);
	        pthread_mutex_lock(&wr);
	        printf("\twriter is writing\n");
	        sleep( rand() % 3);
	        pthread_mutex_unlock(&wr);
	        printf("\twriter is leaving\n");
	    }
	    printf("----writer has done----\n");

	}


	int main(int argc, const char *argv[])
	{
	    int err;
	    pthread_t tid[N], writerTid;
	    int i;


	    err = pthread_create(&writerTid, NULL, writer, (void *)NULL);
	    if (err != 0)
	    {
	        err_quit("can't create process for writer");
	    }

	    for (i = 0; i < N; i++)
	    {
	        err = pthread_create(&tid[i], NULL, reader, (void *)(i+1));
	        if (err != 0)
	        {
	            err_quit("can't create process for reader");
	        }
	    }
	    pause();
	    return 0;
	}

对于使用了多线程的程序，编译方法如下：

	g++ reader_writer.c -l pthread -o a.out


**互斥量实现读者写者问题（写者优先）**

	#include <pthread.h>
	#include <signal.h>
	#include "apue.h"
	#define N 5 //No. of reader
	#define M 5 //No. of reading and writing

	pthread_mutex_t rd = PTHREAD_MUTEX_INITIALIZER;       // it's mean reader can reading
	pthread_mutex_t wr = PTHREAD_MUTEX_INITIALIZER;       // it's mean writer can writing
	pthread_mutex_t priority = PTHREAD_MUTEX_INITIALIZER; // it's mean writer can writing

	int readCount = 0;

	void* reader(void *arg)
	{
		int n = M;
		int id = (int)arg;
		while (n--)
		{
			sleep( rand() % 3);

			pthread_mutex_lock(&priority);
			pthread_mutex_lock(&rd);
			readCount++;
			if( readCount == 1)
			{  //first reader
				pthread_mutex_lock(&wr);
			}
			pthread_mutex_unlock(&rd);
			pthread_mutex_unlock(&priority);

			printf("reader %d is reading\n", id);
			sleep( rand() % 3);

			pthread_mutex_lock(&rd);
			readCount--;
			if (readCount == 0)
			{  //last reader
				pthread_mutex_unlock(&wr);
			}
			pthread_mutex_unlock(&rd);
			printf("reader %d is leaving\n", id);
		}
		printf("----reader %d has done----\n", (int)arg);
	}

	void* writer(void *arg)
	{
		int n = M;
		while (n--)
		{
			sleep( rand() % 4);
			pthread_mutex_lock(&priority);
			pthread_mutex_lock(&wr);
			printf("\twriter is writing\n");
			sleep( rand() % 4);
			pthread_mutex_unlock(&wr);
			pthread_mutex_unlock(&priority);
			printf("\twriter is leaving\n");
		}
		printf("----writer has done----\n");

	}

	int main(int argc, const char *argv[])
	{
		int err;
		pthread_t tid[N], writer_tid;
		int i;

		for (i = 0; i < N; i++)
		{
			err = pthread_create(&tid[i], NULL, reader, (void *)(i+1));
			if (err != 0)
			{
				err_quit("can't create process for reader");
			}
		}

		err = pthread_create(&writer_tid, NULL, writer, (void *)NULL);
		if (err != 0)
		{
			err_quit("can't create process for writer");
		}
		pause();
		return 0;
	}


<a name="2"></a>
###6 读写锁

读写锁适合于对数据结构的读次数比写次数多得多的情况.因为,读模式锁定时可以共享,以写
模式锁住时意味着独占,所以读写锁又叫共享-独占锁.

**初始化和销毁:**

    #include <pthread.h>
    int pthread_rwlock_init(pthread_rwlock_t *restrict rwlock, const
        pthread_rwlockattr_t *restrict attr);
    int pthread_rwlock_destroy(pthread_rwlock_t *rwlock);

成功则返回0,出错则返回错误编号. 同互斥量以上,在释放读写锁占用的内存之前,需要先通过
pthread_rwlock_destroy对读写锁进行清理工作, 释放由init分配的资源.

**读和写:**

    #include <pthread.h>
    int pthread_rwlock_rdlock(pthread_rwlock_t *rwlock);
    int pthread_rwlock_wrlock(pthread_rwlock_t *rwlock);
    int pthread_rwlock_unlock(pthread_rwlock_t *rwlock);

成功则返回0,出错则返回错误编号.这3个函数分别实现获取读锁,获取写锁和释放锁的操作.获
取锁的两个函数是阻塞操作,同样,非阻塞的函数为:

    #include <pthread.h>
    int pthread_rwlock_tryrdlock(pthread_rwlock_t *rwlock);
    int pthread_rwlock_trywrlock(pthread_rwlock_t *rwlock);

成功则返回0,出错则返回错误编号.非阻塞的获取锁操作,如果可以获取则返回0,否则返回
错误的EBUSY.


	#include <pthread.h>
	#include <signal.h>
	#include "apue.h"
	#define N 5 //No. of reader
	#define M 5 //No. of reading and writing

	pthread_rwlock_t lock; //it's mean writer can writing

	int readCount = 0;

	void* reader(void *arg)
	{
	    int n = M;
	    int id = (int)arg;
	    while (n--)
	    {
	        sleep( rand() % 3);
	        pthread_rwlock_rdlock(&lock);
	        printf("reader %d is reading\n", id);
	        sleep( rand() % 3);

	        pthread_rwlock_unlock(&lock);
	        printf("reader %d is leaving\n", id);
	    }
	    printf("----reader %d has done----\n", (int)arg);
	}

	void* writer(void *arg)
	{
	    int n = M;
	    while (n--)
	    {
	        sleep( rand() % 3);
	        pthread_rwlock_wrlock(&lock);
	        printf("\twriter is writing\n");
	        sleep( rand() % 3);
	        pthread_rwlock_unlock(&lock);
	        printf("\twriter is leaving\n");
	    }
	    printf("----writer has done----\n");
	}

	int main(int argc, const char *argv[])
	{
	    int err;
	    pthread_t tid[N], writerTid;
	    int i;

	    err = pthread_create(&writerTid, NULL, writer, (void *)NULL);
	    if (err != 0)
	    {
	        err_quit("can't create process for writer");

	    }

	    pthread_rwlock_init(&lock, NULL);
	    for (i = 0; i < N; i++)
	    {
	        err = pthread_create(&tid[i], NULL, reader, (void *)(i+1));
	        if (err != 0)
	        {
	            err_quit("can't create process for reader");
	        }
	    }
	    pause();
	    pthread_rwlock_destroy(&lock);
	    return 0;
	}



<a name="3"></a>
###7 条件变量
条件变量参考[这里][6]。

与互斥锁不同，条件变量是用来等待而不是用来上锁的。条件变量用来自动阻塞一个线程，直
到某特殊情况发生为止。通常条件变量和互斥锁同时使用。

条件变量使我们可以睡眠等待某种条件出现。条件变量是利用线程间共享的全局变量进行同步
的一种机制，主要包括两个动作：一个线程等待"条件变量的条件成立"而挂起；另一个线程使
"条件成立"（给出条件成立信号）。

条件的检测是在互斥锁的保护下进行的。如果一个条件为假，一个线程自动阻塞，并释放等待
状态改变的互斥锁。如果另一个线程改变了条件，它发信号给关联的条件变量，唤醒一个或多
个等待它的线程，重新获得互斥锁，重新评价条件。如果两进程共享可读写的内存，条件变量
可以被用来实现这两进程间的线程同步。

使用条件变量之前要先进行初始化。可以在单个语句中生成和初始化一个条件变量如：
`pthread_cond_t my_condition=PTHREAD_COND_INITIALIZER;`（用于进程间线程的通信）。
可以利用函数pthread_cond_init动态初始化。

条件变量分为两部分:条件和变量.条件本身是由互斥量保护的.线程在改变条件状态前先要锁住
互斥量.它利用线程间共享的全局变量进行同步的一种机制。

相关的函数如下：

    int pthread_cond_init(pthread_cond_t *cond,pthread_condattr_t *cond_attr);
    int pthread_cond_wait(pthread_cond_t *cond,pthread_mutex_t *mutex);
    int pthread_cond_timewait(pthread_cond_t *cond,pthread_mutex *mutex,const
          timespec *abstime);
    int pthread_cond_destroy(pthread_cond_t *cond);
    int pthread_cond_signal(pthread_cond_t *cond);
    int pthread_cond_broadcast(pthread_cond_t *cond);  //解除所有线程的阻塞

**简要说明：**

1. 初始化.init()或者pthread_cond_t cond=PTHREAD_COND_INITIALIER;属性置为NULL
2. 等待条件成立.pthread_wait,pthread_timewait.wait()释放锁,并阻塞等待条件变量为真
timewait()设置等待时间,仍未signal,返回ETIMEOUT(加锁保证只有一个线程wait)
3. 激活条件变量:pthread_cond_signal,pthread_cond_broadcast(激活所有等待线程)
4. 清除条件变量:destroy;无线程等待,否则返回EBUSY

详细说明

**初始化:**

条件变量采用的数据类型是pthread_cond_t, 在使用之前必须要进行初始化, 这包括两种方式:

静态: 可以把常量PTHREAD_COND_INITIALIZER给静态分配的条件变量.
动态: pthread_cond_init函数, 是释放动态条件变量的内存空间之前, 要用
pthread_cond_destroy对其进行清理.

    #include <pthread.h>
    int pthread_cond_init(pthread_cond_t *restrict cond, pthread_condattr_t *restrict attr);
    int pthread_cond_destroy(pthread_cond_t *cond);

成功则返回0, 出错则返回错误编号.

当pthread_cond_init的attr参数为NULL时,会创建一个默认属性的条件变量;非默认情况以后讨论.

**等待条件:**

    #include <pthread.h>
    int pthread_cond_wait(pthread_cond_t *restrict cond, pthread_mutex_t
            *restric mutex);
    int pthread_cond_timedwait(pthread_cond_t *restrict cond, pthread_mutex_t
            *restrict mutex, const struct timespec *restrict timeout);

成功则返回0,出错则返回错误编号.这两个函数分别是阻塞等待和超时等待.

等待条件函数等待条件变为真,传递给pthread_cond_wait的互斥量对条件进行保护,调用者把锁
住的互斥量传递给函数.函数把调用线程放到等待条件的线程列表上,然后对互斥量解锁,这两个
操作是原子的. 这样便关闭了条件检查和线程进入休眠状态等待条件改变这两个操作之间的时
间通道,这样线程就不会错过条件的任何变化.

当pthread_cond_wait返回时, 互斥量再次被锁住.

3. 通知条件:

    #include <pthread.h>
    int pthread_cond_signal(pthread_cond_t *cond);
    int pthread_cond_broadcast(pthread_cond_t *cond);

成功则返回0, 出错则返回错误编号.

这两个函数用于通知线程条件已经满足. 调用这两个函数, 也称向线程或条件发送信号.
必须注意, 一定要在改变条件状态以后再给线程发送信号.

示例程序

    #include <stdio.h>
    #include <pthread.h>
    pthread_mutex_t mutex;
    pthread_cond_t cond;
    void *thread1(void *arg)
    {

        pthread_cleanup_push(pthread_mutex_unlock, &mutex);

        //提供函数回调保护
        while (1) {

            printf("thread1 is running\n");

            pthread_mutex_lock(&mutex);

            pthread_cond_wait(&cond, &mutex);

            printf("thread1 applied the condition\n");

            pthread_mutex_unlock(&mutex);

            sleep(4);

        }

        pthread_cleanup_pop(0);

    }


    void *thread2(void *arg)
    {

        while (1) {

            printf("thread2 is running\n");

            pthread_mutex_lock(&mutex);

            pthread_cond_wait(&cond, &mutex);

            printf("thread2 applied the condition\n");

            pthread_mutex_unlock(&mutex);

            sleep(1);

        }
    }



    int main()
    {

        pthread_t thid1, thid2;

        printf("condition variable study!\n");

        pthread_mutex_init(&mutex, NULL);

        pthread_cond_init(&cond, NULL);

        pthread_create(&thid1, NULL, (void *) thread1, NULL);

        pthread_create(&thid2, NULL, (void *) thread2, NULL);

        do {

            pthread_cond_signal(&cond);

        } while (1);

        sleep(20);

        pthread_exit(0);

        return 0;

    }


**条件变量与互斥锁、信号量的区别**

1. 互斥锁必须总是由给它上锁的线程解锁，信号量的挂出即不必由执行过它的等待操作的同一
进程执行。一个线程可以等待某个给定信号灯，而另一个线程可以挂出该信号灯。
2. 互斥锁要么锁住，要么被解开（二值状态，类型二值信号量）。
3. 由于信号量有一个与之关联的状态（它的计数值），信号量挂出操作总是被记住。然而当向
一个条件变量发送信号时，如果没有线程等待在该条件变量上，那么该信号将丢失。
4. 互斥锁是为了上锁而设计的，条件变量是为了等待而设计的，信号灯即可用于上锁，也可用
于等待，因而可能导致更多的开销和更高的复杂性。




<a name="4"></a>
###8 信号量

信号量(semaphore)简单好理解，不知道它有什么缺点，在《unix
环境高级编程》中居然没有讲。我们大学课堂，上操作系统，各种PV操作，其实就是在操作信号量。

信号量的具体使用方法，请

	man sem_init

相关的几个系统调用：

	sem_init
	sem_wait
	sem_trywait
	sem_post
	sem_getvalue
	sem_destory

下面来看一个信号量解决哲学家就餐问题，在这里semaphore
初始为1,用法和互斥量没有什么区别。

	#include <semaphore.h>
	#include <pthread.h>
	#include "apue.h"

	#define N 5 // No. of philosopher
	#define M 5 // times of eating
	sem_t forks[N];

	void * thr_philosopher( void *arg);
	int main(int argc, char* argv[])
	{
	    int i = 0;
	    int err;
	    pthread_t tid[N];
	    void *tret;
	    //initilize semaphore
	    for (i = 0; i < N; i++)
	    {
	        if(sem_init(&forks[i], 0, 1) != 0)
	        {
	            err_quit("init forks error");
	        }
	    }
	    //create thread
	    for (i = 0; i < N; i++)
	    {
	        err = pthread_create(&tid[i], NULL, thr_philosopher, (void *)i);
	        if (err != 0)
	        {
	            err_quit("can't create thread %d: %s\n", i + 1, strerror(err));
	        }
	    }

	    //get the return value
	    for (i = 0; i < N; i++)
	    {
	        err = pthread_join(tid[i], &tret);
	        if (err != 0)
	        {
	            err_quit("can't join with philosopher %d : %s\n", i + 1,
	                    strerror(err));
	        }
	        printf("-------------------philosopher %d has done-------------------\n", (int)tret);
	    }

	    // delete the source of semaphore
	    for (i = 0; i < N; i++)
	    {
	        err = sem_destroy(&forks[i]);
	        if (err != 0)
	        {
	            err_sys("can't destory semaphore");
	        }
	    }
	    exit(0);
	}

	void * thr_philosopher( void *arg)
	{

	    /*
	     * here cann't judge arg == NULL
	     * because (void *)0 will lead to arg = NULL
	     */
	    int n = M;
	    int i = 0;
	    i = (int)arg;
	    while ( n-- )
	    {
	        sleep(1);
	        if ( i == N - 1)
	        {
	            sem_wait(&forks[0]);
	            sem_wait(&forks[i]);
	        }
	        else
	        {
	            sem_wait(&forks[i]);
	            sem_wait(&forks[i + 1]);
	        }
	        printf("philosopher %d is eating\n", i + 1);
	        if ( i == N - 1)
	        {
	            sem_post(&forks[0]);
	            sem_post(&forks[i]);
	        }
	        else
	        {
	            sem_post(&forks[i]);
	            sem_post(&forks[i + 1]);
	        }

	    }

	    return ((void*)i);
	}


<a name="5"></a>
###9 文件互斥

文件互斥并不是操作系统提供的一组API，而是使用了一点小技巧，我们可以通过linux下文件互
斥地打开，实现线程/进程互斥的访问资源，以此实现多线程编程。

值得注意的是，文件互斥的方式不但适用于多线程编程，还能实现多进程之间的交互。

文件互斥还有一个妙用－－保证一个系统中只有一个实例，请参考[这里][4]。

新建lock.h 文件，内容入下：

	#include    <sys/types.h>
	#include    <sys/stat.h>
	#include    <fcntl.h>
	#include    <unistd.h>
	void initlock(const char *lockfile);
	void lock(const char *lockfile);
	void unlock(const char *lockfile);


新建lock.c 文件，内容入下：

	#include <sys/types.h>
	#include <sys/stat.h>
	#include <fcntl.h>
	#include <unistd.h>

	void initlock(const char *lockfile)
	{
	    int i;
	    unlink(lockfile);
	}

	void lock(const char *lockfile)
	{

	    int fd;
	    while ( (fd = open(lockfile, O_RDONLY | O_CREAT | O_EXCL)) < 0)
	        sleep(1);
	    close(fd);
	}

	void unlock(const char *lockfile)
	{
	    unlink(lockfile);
	}


新建main.c 文件，内容如下：

	#include <sys/wait.h>
	#include <stdio.h>
	#include <stdlib.h>
	#include <pthread.h>
	#include "apue.h"
	#include "lock.h"

	#define N 5 // No. of philosopher
	#define M 3 // No. of eating
	static char* forks[] = {"fork0", "fork1", "fork2", "fork3", "fork4"};

	void takeFork( int i )
	{
	    if ( i == N - 1 )
	    {
	        lock(forks[0]);
	        printf("philosopher %d:  takefork 0\n", i + 1 );
	        lock(forks[i]);
	        printf("philosopher %d:  takefork %d\n", i + 1, i );
	    }
	    else
	    {
	        lock(forks[i]);
	        printf("philosopher %d:  takefork %d\n", i + 1, i );
	        lock(forks[i+1]);
	        printf("philosopher %d:  takefork %d\n", i + 1, i+1 );
	    }
	}

	void eating(int i, int nsecs)
	{
	    printf("\tphilosopher %d:  eat %d s\n", i + 1, nsecs );
	    sleep(nsecs);
	}

	void thinking(int i, int nsecs)
	{
	    printf("philosopher %d:  think %d sec\n", i + 1, nsecs );
	    sleep(nsecs);
	}

	void putFork( int i )
	{
	    if ( i == N - 1 )
	    {
	        unlock(forks[0]);
	        unlock(forks[i]);
	    }
	    else
	    {
	        unlock(forks[i]);
	        unlock(forks[i+1]);
	    }
	}

	void* philosopher(void *arg)
	{
		int i = (int)arg;
	//    printf("philosopher %d : pid = %d is running", i + 1, getpid());
	    int m = M;
	    int nsecs;
	    srand( time(0) );
	    while ( m-- )
	    {
	        nsecs = rand() % 5;
	        thinking(i, nsecs);
	        takeFork(i);
	        nsecs = rand() % 5;
	        eating(i, nsecs);
	        putFork(i);
	    }
	    printf("====================philosopher %d : is\
	 successful===================\n", i + 1 );
	}

	int main(int argc, char* argv[])
	{

	    int i;
		int err;
		pthread_t tid[N];

		for (i = 0; i < sizeof(forks) / sizeof(forks[0]); i++)
		{
			initlock(forks[i]);
		}

		for (i = 0; i < N; i++)
		{
			err = pthread_create(&tid[i], NULL, philosopher, (void *)i);
			if (err != 0)
			{
				err_quit("can't create process for philosopher");
			}
		}
		pause();
	}


最后，我们来看一个使用文件互斥实现进程之间的交互的例子，其中，lock.h 和 lock.c
请参考上面的实现。main.c 的实现如下：

	#include <sys/wait.h>
	#include <sys/wait.h>
	#include <stdio.h>
	#include "apue.h"
	#include "lock.h"

	#define N 5 // No. of philosopher
	#define M 3 // No. of eating
	static char* forks[] = {"fork0", "fork1", "fork2", "fork3", "fork4"};

	void takeFork( int i )
	{
	    if ( i == N - 1 )
	    {
	        lock(forks[0]);
	        printf("philosopher %d:  takefork 0\n", i + 1 );
	        lock(forks[i]);
	        printf("philosopher %d:  takefork %d\n", i + 1, i );
	    }
	    else
	    {
	        lock(forks[i]);
	        printf("philosopher %d:  takefork %d\n", i + 1, i );
	        lock(forks[i+1]);
	        printf("philosopher %d:  takefork %d\n", i + 1, i+1 );
	    }
	}

	void eating(int i, int nsecs)
	{
	    printf("philosopher %d:  eat %d sec\n", i + 1, nsecs );
	    sleep(nsecs);
	}

	void thinking(int i, int nsecs)
	{
	    printf("philosopher %d:  think %d sec\n", i + 1, nsecs );
	    sleep(nsecs);
	}

	void putFork( int i )
	{
	    if ( i == N - 1 )
	    {
	        unlock(forks[0]);
	        unlock(forks[i]);
	    }
	    else
	    {
	        unlock(forks[i]);
	        unlock(forks[i+1]);
	    }
	}

	void philosopher(int i)
	{
	    printf("philosopher %d : pid = %d\n is running", i + 1, getpid());
	    int m = M;
	    int nsecs;
	    srand( time(0) );
	    while ( m-- )
	    {
	        nsecs = rand() % 5;
	        thinking(i, nsecs);
	        takeFork(i);
	        nsecs = rand() % 5;
	        eating(i, nsecs);
	        putFork(i);
	    }
	    printf("==============================philosopher %d : is\
	 successful=============================\n", i + 1 );
	}

	int main(int argc, char* argv[])
	{
	    pid_t pid;
	    int i;

	    for ( i = 0; i < N; i++ )
	    {
	        pid = fork();
	        if (pid == 0 || pid == -1)
	        {
	            break;
	        }
	    }

	    if ( pid == 0 )
	    {
	        philosopher(i);
	    }
	    else if ( pid < 0 )
	    {
	        perror("fork error");
	    }
	    else
	    {
	        waitpid(pid, 0, 0);
	    }

	    exit(0);
	}


如果你是新手，可能对下面这几行代码不是很理解，请参考酷壳网的《[一个fork的面试题][5]》。

	    for ( i = 0; i < N; i++ )
	    {
	        pid = fork();
	        if (pid == 0 || pid == -1)
	        {
	            break;
	        }
	    }

完。

[1]: http://289972458.iteye.com/blog/1325189
[2]: http://book.douban.com/subject/4022870/
[3]: http://zh.wikipedia.org/wiki/%E5%93%B2%E5%AD%A6%E5%AE%B6%E5%B0%B1%E9%A4%90%E9%97%AE%E9%A2%98
[4]: http://mingxinglai.com/cn/2013/01/leveldb-lock/
[5]: http://coolshell.cn/articles/7965.html
[6]: http://www.cnblogs.com/feisky/archive/2010/03/08/1680950.html