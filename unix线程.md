##unix线程----apue读书笔记
###多线程编程优点
+ 对每种事件类型分配单独的处理线程，简化处理异步事件的代码。每个线程在处理时可以采用同步编程模式，同步编程要比异步编程简单得多
+ 多个进程必须使用操作系统提供的复杂机制才能实现内存和文件描述符的共享。而线程之间可以直接访问相同的内存和文件描述符
+ 有些问题可以分解从而提高整个系统的吞吐量。多个线程情况下互相独立的任务处理就能够并行执行
+ 交互的程序可以通过多线程来改善响应时间，可以把处理用户的输入部分和其他部分分开来

这里书上说处理器的结构不影响程序的结构，不管处理器的个数多少，都能得到多线程模型的好处，即使在单处理器上。我觉得这可能是因为操作系统调度的原因。多个线程不管操作系统怎么调度，得到的cpu时间一般是要比单个线程多的，这样增加cpu使用时间，程序运行的速度就有提升


###常用的API
+ 比较两个线程的id是否相同(因为不是所有的实现表示线程id的pthread_t都是用整数表示)
```
/*
* 相等返回非0,不相等返回0
*/
#include<pthread.h>
int pthread_equal(pthread_t tid1,pthread_t tid2);
```
+ 获得调用线程的线程id
```
#include<pthread.h>
pthread_t pthread_self(void);
```
+ 创建线程
```
/*
* 成功返回0,错误返回错误编号
* tidp是指向子线程的id的指针
* attr是要创建线程的属性
* start_rtn是函数指针,线程从这个函数开始执行
* arg是start_rtn的参数
*/
#include<pthread.h>
int pthread_create(pthread_t *restrict tidp,
				const pthread_attr_t * restrict attr,
				void *(* start_rtn)(void *), void *restrict arg);
```
+ 线程退出（线程退出的方式有很多种可以从启动函数中return;线程可以被同一进程中的其他线程取消;或者线程调用pthread_exit退出）
```
#include<pthread.h>
/*
* rval_ptr是退出的指针，可以通过下面的pthread_join获得线程退出的这个指针
*/
void pthread_exit(void *rval_ptr);

```
+ 等待线程退出
```
/*
* 成功返回0,否则返回错误编号
* thread 指定线程的id
* rval_ptr指向存储线程推出值的指针变量
*/
int pthread_join(pthread_t thread, void **rval_ptr);
```
+ 取消同一进程中的其他进程
```
#include<pthread.h>
/*
* 成功返回0,失败返回错误代码
* tid被指定线程的id
*/
int pthread_cancel(pthread_t tid);//只是发送请求，并不是等待终止
```
+ 分离线程,在线程终止之后线程的资源被立即释放
```
#include<pthread.h>
int pthread_detach(pthread_t tid);
```
+ 注册线程清理函数
```
#include<pthread.h>
void pthread_cleanup_push(void (*rtn)(void *),void *arg)
void pthread_cleanup_pop(int execute);
```
**这些api都能够类比进程的原语**

进程原语 |  线程原语 
-----------|-------------
fork        |   pthread_create
exit        |  pthread_exit
waitpid  |  pthread_join
atexit     |  pthread_cleanup_push,pthread_cleanup_pop
getpid   | pthread_self
abort    |  pthread_cancel

### 线程同步
为什么要线程同步这个都知道，有个很经典的例子，count++，这个操作分三个步骤

1. 从内存中读入寄存器
2. 寄存器对变量加1
3. 将新值写入内存
这三个步骤不是原子的操作那么多个线程执行对共享变量count的自加操作就会出现竞争条件。



**书上有句话很重要-----只有将所有的线程都设计成遵守相同数据访问规则的，互斥机制才能正常的工作，如果允许某个线程在没有得到锁的情况下也能够访问共享资源，那么即使其他线程在使用共享资源前都申请锁也还是会出现数据不一致的情况**
难道共享数据的访问只能够用锁，然后对该数据拷贝，然后释放锁，最后返回数据这种模式了？

书上讨论了5种基本的同步机制，互斥量(mutex)，读写锁(reader-writer lock)，条件变量(Condition Variable)，自旋锁(spin lock)，屏障(barrier)

类型  |   描述 
-------|-----------
互斥量(mutex) | 本质上就是一把锁，确保同一时间只有一个线程能够访问数据。对互斥量进行加锁之后，其他任何想要对该互斥量进行加锁的线程都会被阻塞，直到当前线程释放锁，如果释放互斥量时有一个以上线程阻塞，那么该锁上所有阻塞的线程都会变成可运行的状态。第一个变成可运行的线程就可以对互斥量进行枷锁，其他线程看到互斥量的状态又是锁着的 
读写锁(reader-writer lock) OR 共享互斥锁(shared-exclusive lock)| 和互斥量类似，不过能够允许更高的并行性。互斥量只有两种状态，要么就是加锁状态，要么就是不加锁的状态。而读写锁有三种状态，读模式加锁状态，写模式加锁状态，不加锁状态。当读写锁是写状态加锁，在这个锁被解锁之前，所有对这个锁加锁的线程都会被阻塞。当读写锁是读状态加锁，所有以读状态加锁的线程都能得到该锁，但是任何以写模式加锁的线程都会被阻塞。为了防止读模式长期占用，写请求得不到满足，一般的操作系统实现都会来到一个写请求后阻塞随后的读锁请求。从上面可以看出，一次只有一个线程可以占有写模式的读写锁，但是多个线程可以同时占有读模式的读写锁。读写锁非常适合对数据结构的读操作次数远大于写的情景
条件变量(Condition Variable) | 条件变量(Condtion Variable)是在多线程程序中用来实现“等待->唤醒”逻辑常用的方法。举个简单的例子，应用程序A中包含两个线程t1和t2。t1需要在bool变量test_cond为true时才能继续执行，而test_cond的值是由t2来改变的。这种情况下，有两种实现方法，可以采用一直论询的方式，不过这样太消耗cpu时间。另一种是用条件变量，不满足条件的话就在条件变量的等待列表里等待。满足条件之后再被唤醒。
自旋锁(spin lock) | 自旋锁和互斥量类似不过它不是通过休眠使进程阻塞，而是在获取锁之前一直处于忙等的状态，自旋锁用于锁被持有的时间短，线程并不希望在重新调度上花太多时间
屏障(barrier) | 是协调多个线程并行工作的同步机制，屏障要求每个线程等待，知道所有合作线程都到达某一点，然后从该点继续执行,之前的pthread_join就是一个屏障的例子


这里再补充说明一下条件变量，先看一个例子
```
#include<pthread.h>

struct msg
{
	struct msg *m_next;
	/*... more stuff here ...*/
};

struct msg *workq;
pthread_cond_t qready = PTHREAD_COND_INITIALIZER;
pthread_mutex_t qlock = PTHREAD_MUTEX_INITIALIZER;

void process_msg(void)
{
	struct msg *mp;
	
	for(;;)
	{
		pthread_mutex_lock(&qlock);
		while(workq == NULL)
			pthread_cond_wait(&qready,&qlock);//在条件变量上阻塞
		mp = workq;
		workq = mp->m_next;
		pthread_mutex_unlock(&qlock);
		/*now process the message mp */

	}
}
void enqueue_msg(struct msg *mp)
{
	pthread_mutex_lock(&qlock);
	mp->m_next = workq;
	workq = mp;
	pthread_mutex_unlock(&qlock);
	pthread_cond_signal(&qready);//唤醒该条件变量上至少一个线程
}
```
这里可能会有几个问题和注意点：

+ **为什么要条件变量和互斥量一起使用？**
这是为了防止一个竞争造成死锁，试着想想一下没有互斥量的保护，A线程检查了workq==NULL成立，准备进入条件变量的阻塞列表，这时候B线程执行了更新workq，唤醒阻塞进程的代码，但是没有等待进程，因为A还没执行玩。这个时候明明workq中有任务，但是工作线程却被阻塞了
+ **while循环检查条件是否满足？**
防止被唤醒后有其他线程执行process_msg,将条件workq的状态又更新成NULL了
+ **一定要注意在更改条件状态之后再给线程发信号**
+ **pthread_cond_wait函数中干了什么？**
调用者把锁住的互斥量传递给函数，函数然后自动把调用线程放到等待条件的线程列表上，对互斥量进行解锁。这就关闭了条件检查和线程进入休眠状态等待条件改变这两个操作的时间通道，这样线程就不会错过条件的任何变化。当线程被唤醒，从pthread_cond_wait返回时，信号量被再次锁住


这就是上面的5种同步方式的api
```
//互斥量
int pthread_mutex_lock(pthread_mutex_t *mutex);
int pthread_mutex_trylock(pthread_mutex_t *mutex);
int pthread_mutex_unlock(pthread_mutex_t *mutex);
int pthread_mutex_timedlock(pthread_mutex_t *restrict mutex,
							const struct timespec *restrict tsptr);
//读写锁
int pthread_rwlock_rdlock(pthread_rwlock_t *rwlock);
int pthread_rwlock_wrlock(pthread_rwlock_t *rwlock);
int pthread_rwlock_unlock(pthread_rwlock_t *rwlock);
int pthread_rwlock_timedrdlock(pthread_rwlock_t *restrict rwlock,
							   const struct timespec *restrict tsptr);
int pthread_rwlock_timedwrlock(pthread_rwlock_t *restrict rwlock,
							   const struct timespec *restrict tsptr);
//条件变量
int pthread_cond_wait(pthread_cond_t *restrict cond,
					  pthread_mutex_t *restrict mutex);
int pthread_cond_timedwait(pthread_cond_t *restrict cond,
						   pthread_mutex_t *restrict mutex,
						   const struct timespec *restrict tsptr);
int pthread_cond_signal(pthread_cond_t *cond);
int pthread_cond_broadcast(pthread_cond_t *cond);
//自旋锁
int pthread_spin_lock(pthread_spinlock_t *lock);
int pthread_spin_trylock(pthread_spinlock_t *lock);
int pthread_spin_unlock(pthread_spinlock_t *lock);

//屏障
int pthread_barrier_wait(pthread_barrier_t *barrier);


```

###死锁
书上将了一下关于防止死锁的通用的方法。死锁的产生是因为对资源的申请方式是占有等待的方式(根本原因当然是资源不足)，例如有A，B两把锁。线程t1需要两把锁，t1先申请到了A，然后要申请B，然后线程t2申请到了B，再申请A。这样两个线程都执行不下去。解决办法也很简单：

+ 改变资源的申请方式，如果获取锁失败那就释放所有的锁
+ 大家按照统一的方式申请，如果要同时申请A，B两把锁，那么就都按照先申请A，后申请B的方式。




















