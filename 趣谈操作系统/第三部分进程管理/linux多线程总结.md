# linux多线程总结

#### 线程创建与资源回收

#include <pthread.h>

int pthread_create(pthread_t *thread, const pthread_attr_t *attr,
                   void *(*start_routine) (void *), void *arg); 创建线程

 int pthread_join(pthread_t thread, void **retval);回收子线程资源

pthread_t thread_id；定义线程

pthread有两种状态joinable状态和unjoinable状态，如果线程是joinable状态，当线程函数自己返回退出时或pthread_exit时都不会释放线程所占用堆栈
和线程描述符（总计8K多）。只有当你调用了pthread_join之后这些资源才会被释放。若是unjoinable状态的线程，这些资源在线程函数退出时或pthread_exit时

pthread_detach函数 用于分离线程，在子线程设置之后子线程和主线程脱离联系，主线程不用等待子线程结束来回收资源。子线程结束时会自动回收系统资源。



#### 4种线程同步方式

1、互斥锁

解锁的互斥锁可以由某个线程获得，一旦获得，这个互斥锁会锁上变成lock状态，只能由这个线程打开该锁，其他线程想要该锁，必须得等到该线程打开锁之后。

基本API（获取细节，在linux下，man + 函数名字）

 int pthread_mutex_init(pthread_mutex_t *restrict mutex,
        const pthread_mutexattr_t *restrict attr);
 pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER; 互斥量初始化（动态、静态）

#include <pthread.h> 头文件

int pthread_mutex_lock(pthread_mutex_t *mutex); 加锁
int pthread_mutex_trylock(pthread_mutex_t *mutex);
int pthread_mutex_unlock(pthread_mutex_t *mutex);解锁

 int pthread_mutex_destroy(pthread_mutex_t *mutex); 销毁

缺点：性能太差。

示例：

两个线程同时打印i++；

```
#include<pthread.h>
#include<stdio.h>
int another_shared = 0;
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
void * thread_run(void *arg) {
    int *calculator = (int *) arg;
    printf("thread id is: %d \n", pthread_self());
    pthread_mutex_lock(&mutex);
    for (int i = 0; i < 10000; i++) {
        *calculator += 1;
        another_shared += 1;
    }
    pthread_mutex_unlock(&mutex);
}
int main(int c, char **v) {
    int calculator;
    pthread_t tid1;
    pthread_t tid2;
    pthread_create(&tid1, NULL, thread_run, &calculator);
    pthread_create(&tid2, NULL, thread_run, &calculator);
    pthread_join(tid1, NULL);
    pthread_join(tid2, NULL);
    printf("calculator is %d \n", calculator);
    printf("another_shared is %d \n", another_shared);
}
```

2、条件变量

 当线程在等待满足某些条件时使线程进入睡眠状态，一旦条件满足，就唤醒因等待满足特定条件而睡眠的线程 。

结合互斥锁使用，弥补了互斥锁的不足。

API

pthread_cond_t cond = PTHREAD_COND_INITIALIZER; //创建条件变量（静态）

int pthread_cond_init(pthread_cond_t *restrict cond, const pthread_condattr_t *restrict attr);//创建信号量（动态）



int pthread_cond_wait(pthread_cond_t *restrict cond, pthread_mutex_t *restrict mutex);//等待条件触发

 int pthread_cond_timedwait(pthread_cond_t *restrict cond, pthread_mutex_t *restrict mutex, const struct timespec *restrict abstime);//等待条件触发（可设置超时时间）

pthread_cond_wait主要包含3个操作：

1、解锁互斥锁

2、等待条件

3、条件被触发、加互斥锁

int pthread_cond_broadcast(pthread_cond_t *cond);// 广播信号
int pthread_cond_signal(pthread_cond_t *cond);//发送信号给其中一个线程

int pthread_cond_destroy(pthread_cond_t *cond); //销毁信号量



3、读写锁

1.读写锁比互斥锁更加具有适用性和并行性

2.读写锁最适用于对数据结构的读操作读操作次数多余写操作次数的场合。

3.锁处于读模式时可以线程共享，而锁处于写模式时只能独占，所以读写锁又叫做共享-独占锁。

pthread_rwlock_t mutex = PTHREAD_RWLOCK_INITIALIZER;

int pthread_rwlock_init(pthread_rwlock_t *restrict rwlock,
              const pthread_rwlockattr_t *restrict attr);  初始化（静态、动态）

 int pthread_rwlock_rdlock(pthread_rwlock_t *rwlock); 加读锁

 int pthread_rwlock_wrlock(pthread_rwlock_t *rwlock);加写锁

int pthread_rwlock_unlock(pthread_rwlock_t *rwlock);解除读/写锁

4、自旋锁

和互斥锁的区别：

当互斥锁在加锁失败时一般会阻塞，而自旋锁则在一个循环里不断地检测资源是否就绪。

4、信号量

一个计数器，可以用来控制多个线程对共享资源的访问。 

信号量（sem）和互斥锁的区别：互斥锁只允许一个线程进入临界区，而信号量允许多个线程进入临界区 。

基本API

#include <semaphore.h>

int sem_init(sem_t *sem, int pshared, unsigned int value); 初始化信号量，value指定信号量的值。将value设置为1可看成是互斥锁，作用和互斥锁等价。

int sem_wait(sem_t *sem); 当信号量大于0时，获取信号量，执行sem-1

int sem_post(sem_t *sem);  释放信号量，执行sem+1

int sem_destroy(sem_t *sem)  销毁信号量

特点：通用性好。

示例：

//编程实现三个线程ABC，并让它们顺次打印ABC

```
#include<stdio.h>
#include<sys/types.h>
#include<semaphore.h>
#include<pthread.h>
sem_t sem_id1, sem_id2, sem_id3;
*void* *func1() {
  sem_wait(&sem_id1);//sem_id1-1 sem_id1=0
  printf("A\n");
  sem_post(&sem_id2);//sem_id2+1 sem_id2=1
}
*void* *func2() {
  sem_wait(&sem_id2);//被func1唤醒 sem_id2=0
  printf("B\n");
  sem_post(&sem_id3);//sem_id3+1 sem_id3=1
}
*void* *func3() {
  sem_wait(&sem_id3);//被func2唤醒 ,sem_id3=0
  printf("C\n");
  sem_post(&sem_id1); //sem_id1=1
}

*int* main() {
  sem_init(&sem_id1, 0, 1);  //活动
  sem_init(&sem_id2, 0, 0);
  sem_init(&sem_id3, 0, 0);
  pthread_t pthread_id1, pthread_id2, pthread_id3;
  pthread_create(&pthread_id1, NULL, func1, NULL);
  pthread_create(&pthread_id2, NULL, func2, NULL);
  pthread_create(&pthread_id3, NULL, func3, NULL);
  pthread_join(pthread_id1, NULL);
  pthread_join(pthread_id2, NULL);
  pthread_join(pthread_id3, NULL);
  return 0;
}
```



#### 线程同步和互斥

所谓互斥，就是不同线程通过竞争进入临界区（共享的数据和硬件资源），为了防止访问冲突，在有限的时间内只允许其中之一独占性的使用共享资源。如不允许同时写

同步关系则是多个线程彼此合作，通过一定的逻辑关系来共同完成一个任务。一般来说，同步关系中往往包含互斥，同时对临界区的资源会按照某种逻辑顺序进行访问。如先生产后使用。

 互斥是通过竞争对资源的独占使用，彼此之间不需要知道对方的存在，执行顺序是一个乱序。
同步是协调多个相互关联线程合作完成任务，彼此之间知道对方存在，执行顺序往往是有序的。 



#### 线程创建

1、调用的系统调用是clone，task_struct结构引用计数+1

2、线程内存模型如下：

![img](https://upload-images.jianshu.io/upload_images/144787-3bd837e4c115098a.png?imageMogr2/auto-orient/strip|imageView2/2/format/webp)



一个进程内的多个子线程共享该进程的代码段、数据段。

各个子线程有自己的寄存器、栈（从进程的堆分配）等。



#### 进程和线程的关系：

  （1）一个线程只能属于一个进程，而一个进程可以有多个线程，但至少有一个线程。

  （2）资源分配给进程，同一进程的所有线程共享该进程的所有资源。

  （3）处理器分给线程，即真正在处理器上运行的是线程。

  （4）线程在执行过程中，需要协作同步。不同进程的线程间要利用消息通信的办法实现同步。线程是指进程内的一个执行单元,也是进程内的可调度实体.

进程与线程的区别：

  （1）调度：线程作为调度和分配的基本单位，进程作为拥有资源的基本单位

  （2）并发性：不仅进程之间可以并发执行，同一个进程的多个线程之间也可并发执行

  （3）拥有资源：进程是拥有资源的一个独立单位，线程不拥有系统资源，但可以访问隶属于进程的资源.

  （4）系统开销：在创建或撤消进程时，由于系统都要为之分配和回收资源，导致系统的开销明显大于创建或撤消线程时的开销。



