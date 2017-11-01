### libevent笔记
感觉到现在还没有认认真真看过几个源代码，所以从网上找了一些源代码准备阅读一下，选择这个libevent是主要因为是c写的，而且是一个有关I/O复用的网络库。看了网上其他人的解析之后，觉得这个代码还是听不错的，遂准备细读一下，边读边写笔记。我选择了libevent-release-1.4.15-stable (http://libevent.org/) 的版本，当然也可以选新的，但是新的版本加入了不少新的功能，代码量大，阅读起来可能费力一点，我觉得还是从简单的开始比较好。关于这个源代码的解析，写得比较好的就是**[《libevent源码深度剖析——张亮》](http://blog.csdn.net/sparkliang/article/category/660506)**，本文的基本的内容和图片都引用了这篇博客

我觉得要读源代码首先你得会用，如果连项目提供的一些常用的API都不会用，那么读源代码的过程肯定很痛苦。加入对某个项目不是很熟悉，那么可以通过一些简单的例子入手，了解常用的API接口和功能，然后在阅读的过程中多思考，遇到了这个问题要你你会怎么实现，渐渐的你就会在代码中看到作者和你的想法相同与不同之处，相同那没得说，那肯定实现的比较好的方法就这一种，不同啊，应该是你需要提高的地方。再者阅读代码的过程中，不必太过追求细节，主要看实现思路。其实我还有一个方法，就是阅读时如果觉得函数调用的层次过多，可以写一些伪代码，代码里面写上函数的调用关系就行，能够尽快帮助你掌握真个程序的架构


安装过程就略过，一般的套路都是执行某个config脚本进行一系列的配置，然后执行make && make install就行


在源代码的根目录有个sample文件夹，里面有附带的样例代码很不错。下面的代码是time-test.c中的一部分，这个例子展示了一个定时事件的用法
```
static void
timeout_cb(int fd, short event, void *arg)
{
	struct timeval tv;
	struct event *timeout = arg;
	int newtime = time(NULL);

	printf("%s: called at %d: %d\n", __func__, newtime,
	    newtime - lasttime);
	lasttime = newtime;

	evutil_timerclear(&tv);
	tv.tv_sec = 2;
	event_add(timeout, &tv);
}

int
main (int argc, char **argv)
{
	struct event timeout;
	struct timeval tv;
 
	/* Initalize the event library */
	event_init();

	/* Initalize one event */
	evtimer_set(&timeout, timeout_cb, &timeout);

	evutil_timerclear(&tv);
	tv.tv_sec = 2;
	event_add(&timeout, &tv);    /* 添加一个timeout事件 */

	lasttime = time(NULL);
	
	event_dispatch();    /* 事件分发 */

	return (0);
}

```
程序运行结果是每2s在ternimal输出一行信息
```
tan@tan:~/Documents/code/libevent/libevent-release-1.4.15-stable/sample$ ./time-test
timeout_cb: called at 1507931275: 2
timeout_cb: called at 1507931277: 2
timeout_cb: called at 1507931279: 2
```
###Reactor模式
可以参考Reactor模式一文，理解了Reactor模式看libevent代码就会轻松很多

###重要的数据结构
event_base就是Reactor模式中的Initiation Dispatcher + Synchronous Event Demultiplexer
```
/*
*在libevent中，每个I/O复用函数都必须实现这个5个接口用来对事件的管理
*/
struct eventop {
	const char *name;
	void *(*init)(struct event_base *);    /* 初始化 */
	int (*add)(void *, struct event *);    /* 注册事件 */
	int (*del)(void *, struct event *);    /* 删除事件 */
	int (*dispatch)(struct event_base *, void *, struct timeval *);    /* 事件分发 */
	void (*dealloc)(struct event_base *, void *);    /* 注销，释放资源 */
	/* set if we need to reinitialize the event base */
	int need_reinit;
};

struct event_base {
	/*
	*指向全局变量static const struct eventop *eventops[]中的一个
	*而eventops是libevent将系统提供的I/O复用函数进行封装，包括了select，poll，kqueue，epoll等若干实例
	*/
	const struct eventop *evsel; 
	void *evbase;    /* 这个是上面evsel操作的实例对象，e.g. 注册事件的调用evsel->add(evbase, ev)，事件ev被注册到evbase这个对象上 */
	int event_count;		/* counts number of total events */
	int event_count_active;	/* counts number of active events */

	int event_gotterm;		/* Set to terminate loop */
	int event_break;		/* Set to terminate loop immediately */

	/* active event management */
	struct event_list **activequeues;    /*一个二级指针，activequeues[priority]指向一个事件链表，这个链表中每个节点的优先级都是priority*/
	int nactivequeues;

	/* signal handling info */
	struct evsignal_info sig;

	struct event_list eventqueue;    /* 所有注册事件的双向链表 */
	struct timeval event_tv;    /* 上次等待事件发生的时间，用于时间管理，针对用户将系统时间提前的情况，需要相应地减少min_heap里的key */

	struct min_heap timeheap;

	struct timeval tv_cache;
};

```
event就是Reactor中的Event Handler
```
struct event {
	TAILQ_ENTRY (event) ev_next; /* 事件在双向链表I/O事件(已注册事件链表)链表中的位置，也不好说是位置，准确的说是使用这个字段event之间能够构成双向链表 */

	TAILQ_ENTRY (event) ev_active_next; /* 事件在已激活事件链表中的位置 */
	TAILQ_ENTRY (event) ev_signal_next; /* 时间在信号链表中的位置 */
	//用来管理定时事件，在最小堆中的索引
	unsigned int min_heap_idx;	/* for managing timeouts */

	struct event_base *ev_base; /* libevent的是采用reactor模式，这个指针指向的是reactor实例 */

	int ev_fd;  /* 对于I/O事件，是绑定的文件描述符， 对于signal事件是绑定信号 */
	/*
	*event关注的事件类型，可以是以下三种类型
	*I/O事件：EV_WRITE和EV_READ
	*定时事件：EV_TIMEOUT
	*信号：EV_SIGNAL
	*辅助选项：EV_PERSIST，表明是一个永久事件，这个为了统一I/O事件和信号事件的处理
	*/
	short ev_events;
	short ev_ncalls; /* ev_callback的执行次数*/
	short *ev_pncalls;	/* Allows deletes in callback */

	struct timeval ev_timeout; 

	int ev_pri;		/* smaller numbers are higher priority */

	void (*ev_callback)(int, short, void *arg);   /* 回调函数，被ev_base调用，执行事件处理程序*/
	void *ev_arg;   /* ev_callback函数的参数 */

	int ev_res;		/* result passed to event callback */
	int ev_flags;   /* 用来标记event的状态，取值是EVLIST_TIMEOUT，等一系列的宏*/
};
```
###对信号的处理
在对信号的处理方式上libevent做得很巧妙，它采用的办法是创建一对套接字，两者用tcp协议连接起来，然后一个套接字作为读套接字，一个套接字作为写套接字。为读套接字在event_base实例上注册一个persist的读事件。所有信号都使用同一个信号处理函数，这个信号处理函数做的工作就是：将信号发生标记记为1，记录将信号发生次数增加1，然后通过写套接字写一个字节的数据。这样读套接字上就有I/O事件发生，等待的I/O demultiplexer就会返回，通过检查信号标记发现signal事件发生，将signal注册的事件添加到激活事件链表

下面的结构体就是负责管理signal事件
```
struct evsignal_info {
	struct event ev_signal;    /* 为socket pair的读socket向event_base注册读事件使用的event结构 */
	int ev_signal_pair[2];     /* socket pair对 */
	int ev_signal_added;       /* ev_signal事件是否已经注册 */
	volatile sig_atomic_t evsignal_caught;        /* 是否有信号发生 */
	struct event_list evsigevents[NSIG];          /* evsigevents[signo]表示注册到信号signo的事件链表 */
	sig_atomic_t evsigcaught[NSIG];               /* evsigcaught[signo], 信号signo的触发次数 */
#ifdef HAVE_SIGACTION                             /* 保存原来的信号处理函数，某个信号对应的事件被清空时需要恢复原来的信号处理函数 */
	struct sigaction **sh_old;
#else
	ev_sighandler_t **sh_old;
#endif
	int sh_old_max;
};

```

###事件循环
时间循环的流程可以看代码，代码有注释很详细。对时间事件的做法是，根据所有Timer事件的最小超时时间来设置系统I/O的timeout时间；当系统I/O返回时，再激活所有就绪的Timer事件就可以了，这样就能将Timer事件完美的融合到系统的I/O机制中了。

```
int
event_base_loop(struct event_base *base, int flags)
{
	const struct eventop *evsel = base->evsel;
	void *evbase = base->evbase;
	struct timeval tv;
	struct timeval *tv_p;
	int res, done;

	/* clear time cache */
	base->tv_cache.tv_sec = 0;

	//evsignal_base是全局变量，在处理signal时，用于指明所属的event_base实例
	if (base->sig.ev_signal_added)
		evsignal_base = base;
	done = 0;
	while (!done) {
		/* Terminate the loop if we have been asked to */
		if (base->event_gotterm) {
			base->event_gotterm = 0;
			break;
		}
		//event_break和event_gotterm变量分别可以通过event_base_loopbreak和event_loopexit_cb函数来设置用来终止事件循环
		if (base->event_break) {
			base->event_break = 0;
			break;
		}

		/* You cannot use this interface for multi-threaded apps */
		while (event_gotsig) {
			event_gotsig = 0;
			if (event_sigcb) {
				res = (*event_sigcb)();
				if (res == -1) {
					errno = EINTR;
					return (-1);
				}
			}
		}

		timeout_correct(base, &tv);

		//根据timeheap中最小的超时时间，计算I/O demultiplexer的最大等待时间
		tv_p = &tv;
		if (!base->event_count_active && !(flags & EVLOOP_NONBLOCK)) {
			timeout_next(base, &tv_p);
		} else {
			/* 
			 * if we have active events, we just poll new events
			 * without waiting.
			 */
			evutil_timerclear(&tv);
		}
		/* If we have no events, we just exit */
		if (!event_haveevents(base)) {
			event_debug(("%s: no events registered.", __func__));
			return (1);
		}

		/* update last old time */
		gettime(base, &base->event_tv);

		/* clear time cache */
		base->tv_cache.tv_sec = 0;

		res = evsel->dispatch(base, evbase, tv_p);

		if (res == -1)
			return (-1);
		gettime(base, &base->tv_cache);

		//检查timeheap中的timeout event，将就绪的timeout event从heap上删除，插入到激活链表中
		timeout_process(base);

		if (base->event_count_active) {
			//处理激活链表中的就绪事件，按照优先级进行处理
			event_process_active(base);
			if (!base->event_count_active && (flags & EVLOOP_ONCE))
				done = 1;
		} else if (flags & EVLOOP_NONBLOCK)
			done = 1;
	}

	/* clear time cache */
	base->tv_cache.tv_sec = 0;

	event_debug(("%s: asked to terminate loop.", __func__));
	return (0);
}

```

###时间管理

关于时间管理这块，先看一个被频繁调用的函数——gettime，这个函数的作用是获取系统时间，写入tp指向的结构体。monotonic时间是某些系统支持的系统上电之后的时间，这个时间是一直自增，不会因为用户更改系统时间改变而改变。
```
static int
gettime(struct event_base *base, struct timeval *tp)
{
	if (base->tv_cache.tv_sec) { //检查时间缓存是否有效，有效直接使用缓存
		*tp = base->tv_cache;
		return (0);
	}

#if defined(HAVE_CLOCK_GETTIME) && defined(CLOCK_MONOTONIC)
	if (use_monotonic) { //如果支持monotonic时间，使用monotonic时间
		struct timespec	ts;

		if (clock_gettime(CLOCK_MONOTONIC, &ts) == -1)
			return (-1);

		tp->tv_sec = ts.tv_sec;
		tp->tv_usec = ts.tv_nsec / 1000;
		return (0);
	}
#endif

	return (evutil_gettimeofday(tp, NULL)); //通过系统调用获取时间
}

```
接下来看事件循环中有关时间的操作，下面省去了一些无关代码。这里用了一个时间缓存，为了减少获取时间系统调用的次数。下面的代码中从时间点2向下运行，到重新while判断再到时间点1这段时间内gettime获取的都是tv_cache的时间。event_tv是开始一次事件的时间(包括等待事件就绪和事件处理)，在timeout_correct函数中可以看出来判断时间调整的依据就是当前时间是否小于event_tv(因为在正常情况下当前的时间now与event_tv的关系是：now-event_tv = 等待事件就绪时间 + 事件处理时间，所以要是event_tv < now时间肯定是被调整了，但是如果时间是调整过的，但是向前调整的幅度小于两者的差，那是没有办法进行判断是否调整过的，如果要做到这么精细的判断，可以按照指定的精度更新event_tv)

```
base->tv_cache->tv_sec = 0;//开始清空时间缓存
while(!done)
{
    timeout_correct(base, &tv); //时间校正
    //第一次执行此处，event_tv获取的是系统当前时间，其他情况获取的都是tv_cache里的时间
    gettime(base, &base->event_tv);
    //清空时间缓存，时间点1
    base->tv_cache->tv_sec = 0;
    //等待事件就绪
    res = evsel->dispatch(base, evbase, tv_p);
    //将tv_cache赋值为当前系统时间，时间点2
    gettime(base, &base->tv_cache);
    timeout_process(base);
}
base->tv_cache->tv_sec = 0;
```

下面的代码是time_correct函数的代码

```
static void timeout_correct(struct event_base *base, struct timeval *tv)  
{
    struct event **pev;
    unsigned int size;
    struct timeval off;
    if (use_monotonic) // monotonic时间就直接返回，无需调整
        return;
    gettime(base, tv); // tv <---tv_cache
    // 如果now < event_tv表明用户向前调整时间了，需要校正时间
    if (evutil_timercmp(tv, &base->event_tv, >=)) {
        base->event_tv = *tv;
        return
    }
    // 计算时间差值
    evutil_timersub(&base->event_tv, tv, &off);
    // 调整定时事件小根堆
    pev = base->timeheap.p
    size = base->timeheap.n;
    for (; size-- > 0; ++pev) {
        struct timeval *ev_tv = &(**pev).ev_timeout;
        evutil_timersub(ev_tv, &off, ev_tv);
    }
    base->event_tv = *tv; // 更新event_tv为tv_cache
}
```

