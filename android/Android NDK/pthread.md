pthread是使用使用C语言编写的多线程的API， 简称Pthreads ，是线程的POSIX标准，可以在Unix / Linux / Windows 等系统跨平台使用。在类Unix操作系统（Unix、Linux、Mac OS X等）中，都使用Pthreads作为操作系统的线程。

## 1.线程创建

```C
//子线程1
void test1(int *a){
    printf("线程test1");
    //修改自己的子线程系统释放，注释打开后，线程不能用pthread_join方法
    //pthread_detach(pthread_self());
}
//子线程2
void test2(int *a){
    printf("线程test2");
}

/*
int  pthread_create(pthread_t  *  thread, //新线程标识符
pthread_attr_t * attr, //新线程的运行属性
void * (*start_routine)(void *), //线程将会执行的函数
void * arg);//执行函数的传入参数，可以为结构体
*/

//创建线程方法一    （手动释放线程）
int a=10;
pthread_t pid;
pthread_create(&pid, NULL, (void *)test1, (void *)&a);


//线程退出或返回时，才执行回调，可以释放线程占用的堆栈资源（有串行的作用）
if(pthread_join(pid, NULL)==0){
    //线程执行完成
    printf("线程执行完成：%d\n",threadIndex);
    if (message!=NULL) {
        printf("线程执行完成了\n");
    }
}


//创建线程方法二  （自动释放线程）
//设置线程属性
pthread_attr_t attr;
pthread_attr_init (&attr);
//线程默认是PTHREAD_CREATE_JOINABLE，需要pthread_join来释放线程的
pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_DETACHED);
//线程并发
int rc=pthread_create(&pid, &attr, (void *)test2, (void *)a);
pthread_attr_destroy (&attr);
if (rc!=0) {
    printf("创建线程失败\n");
    return;
}
```
## 2.线程退出和其他
```C
pthread_exit (tes1) //退出当前线程
pthread_main_np () // 获取主线程

//主线程和子线程
if(pthread_main_np()){
    //main thread
}else{
    //others thread
}

int pthread_cancel(pthread_t thread)；//发送终止信号给thread线程,如果成功则返回0
int pthread_setcancelstate(int state, int *oldstate)；//设置本线程对Cancel信号的反应
int pthread_setcanceltype(int type, int *oldtype)；//设置本线程取消动作的执行时机
void pthread_testcancel(void)；//检查本线程是否处于Canceld状态，如果是，则进行取消动作，否则直接返回
```
## 3 线程互斥锁（量）与条件变量
### 3.1 互斥锁(量)
```C
//静态创建
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
//动态创建
int pthread_mutex_init(pthread_mutex_t *mutex, const pthread_mutexattr_t *mutexattr);
//注销互斥锁
int pthread_mutex_destroy(pthread_mutex_t *mutex);

//lock 和unlock要成对出现，不然会出现死锁
int pthread_mutex_lock(pthread_mutex_t *mutex);
int pthread_mutex_unlock(pthread_mutex_t *mutex);
//判断是否可以加锁，如果可以加锁并返回0，否则返回非0
int pthread_mutex_trylock(pthread_mutex_t *mutex);
```
### 3.2 条件变量
- 条件变量是利用线程间共享的全局变量进行同步的一种机制，
- 一个线程等待”条件变量的条件成立”而挂起；
- 另一个线程使”条件成立”（给出条件成立信号）。
- 为了防止竞争，条件变量的使用总是和一个互斥锁结合在一起。

```C
//静态创建
pthread_cond_t cond=PTHREAD_COND_INITIALIZER;
//动态创建
int pthread_cond_init(pthread_cond_t *cond, pthread_condattr_t *cond_attr);
//注销条件变量
int pthread_cond_destroy(pthread_cond_t *cond);
//条件等待，和超时等待
int pthread_cond_wait(pthread_cond_t *cond, pthread_mutex_t *mutex);
int pthread_cond_timedwait(pthread_cond_t *cond, pthread_mutex_t *mutex, 
                           const struct timespec *abstime);
//开启条件，启动所有等待线程
int pthread_cond_broadcast(pthread_cond_t *cond);
//开启一个等待信号量
int pthread_cond_signal(pthread_cond_t *cond);
```
### 4.线程同步：互斥锁（量）与条件变量(具体封装实现)
```C
/*全局的队列互斥条件*/
extern pthread_cond_t fan_cond;
extern pthread_cond_t fan_cond_wait;
/*全局的队列互斥锁*/
extern pthread_mutex_t fan_mutex;
//extern pthread_mutex_t fan_mutex_wait;

extern int fan_thread_status;//0=等待 1=执行 -1=清空所有
extern int fan_thread_clean_status;//0=默认  1=清空所有

//开启线程等待  return=-2一定要处理
extern int fan_thread_start_wait(void);
//正常的超时后继续打开下一个信号量 return=-2一定要处理
int fan_thread_start_timedwait(int sec);
//启动线程，启动信号量
extern int fan_thread_start_signal(void);
//启动等待信号量
extern int fan_thread_start_signal_wait(void);
//暂停线程
extern int fan_thread_end_signal(void);
//初始化互斥锁
extern int fan_thread_queue_init(void);
//释放互斥锁信号量
extern int fan_thread_free(void);

//让队列里面全部执行完毕，而不是关闭线程；
extern int fan_thread_clean_queue(void);
//每次关闭清空后，等待1-2秒，要恢复状态，不然线程添加
extern int fan_thread_init_queue(void);
//设置线程的优先级，必须在子线程
extern int fan_thread_setpriority(int priority);
```

线程队列互斥，并且按入队顺序，一个一个按照外部条件，触发信号量，主要是等待队列，
```C
/*全局的队列互斥条件*/
pthread_cond_t fan_cond=PTHREAD_COND_INITIALIZER;
pthread_cond_t fan_cond_wait=PTHREAD_COND_INITIALIZER;

/*全局的队列互斥锁*/
pthread_mutex_t fan_mutex = PTHREAD_MUTEX_INITIALIZER;
//pthread_mutex_t fan_mutex_wait = PTHREAD_MUTEX_INITIALIZER;
int fan_thread_status=1;//0=等待 1=执行
int fan_thread_clean_status;//0=默认  1=清空所有

//开启线程等待
int fan_thread_start_wait(void){
    pthread_mutex_lock(&fan_mutex);
    fan_thread_clean_status=0;
    while (fan_thread_status==0) {
        pthread_cond_wait(&fan_cond, &fan_mutex);
        if (fan_thread_clean_status==1) {
            break;
        }
    }
    if (fan_thread_clean_status==1) {
        pthread_mutex_unlock(&fan_mutex);
        return -2;
    }
    if (fan_thread_status==1) {
        fan_thread_status=0;
        pthread_mutex_unlock(&fan_mutex);
    }else{
        pthread_mutex_unlock(&fan_mutex);
    }
    return 0;
}
//正常的超时后继续打开下一个信号量
int fan_thread_start_timedwait(int sec){
    int rt=0;
    pthread_mutex_lock(&fan_mutex);
    struct timeval now;
    struct timespec outtime;
    gettimeofday(&now, NULL);
    outtime.tv_sec = now.tv_sec + sec;
    outtime.tv_nsec = now.tv_usec * 1000;
    
    int result = pthread_cond_timedwait(&fan_cond_wait, &fan_mutex, &outtime);
    if (result!=0) {
        //线程等待超时
        rt=-1;
    }
    if (fan_thread_clean_status==1) {
        rt = -2;
    }
    pthread_mutex_unlock(&fan_mutex);
    return rt;
}
//启动线程，启动信号量
int fan_thread_start_signal(void){
    int rs=pthread_mutex_trylock(&fan_mutex);
    if(rs!=0){
        pthread_mutex_unlock(&fan_mutex);
    }
    fan_thread_status=1;
    pthread_cond_signal(&fan_cond);
//    pthread_cond_broadcast(&fan_cond);//全部线程
    pthread_mutex_unlock(&fan_mutex);
    return 0;
}
//开启等待时间的互斥信号量
int fan_thread_start_signal_wait(void){
    int rs=pthread_mutex_trylock(&fan_mutex);
    if(rs!=0){
        pthread_mutex_unlock(&fan_mutex);
    }
//    fan_thread_status=1;
    pthread_cond_signal(&fan_cond_wait);
    //    pthread_cond_broadcast(&fan_cond);//全部线程
    pthread_mutex_unlock(&fan_mutex);
    return 0;
}
//暂停下一个线程
int fan_thread_end_signal(void){
    pthread_mutex_lock(&fan_mutex);
    fan_thread_status=0;
    pthread_cond_signal(&fan_cond);
    pthread_mutex_unlock(&fan_mutex);
    return 0;
}
//初始化互斥锁（动态创建）
int fan_thread_queue_init(void){
    pthread_mutex_init(&fan_mutex, NULL);
    pthread_cond_init(&fan_cond, NULL);
    return 0;
}
//释放互斥锁和信号量
int fan_thread_free(void)
{
    pthread_mutex_destroy(&fan_mutex);
    pthread_cond_destroy(&fan_cond);
    return 0;
}

//清空所有的队列
int fan_thread_clean_queue(void){
    pthread_mutex_lock(&fan_mutex);
    fan_thread_clean_status=1;
    pthread_cond_broadcast(&fan_cond);
    pthread_cond_broadcast(&fan_cond_wait);
    pthread_mutex_unlock(&fan_mutex);
    return 0;
}
//恢复队列
int fan_thread_init_queue(void){
    pthread_mutex_lock(&fan_mutex);
    fan_thread_clean_status=0;
    fan_thread_status=1;
    pthread_cond_signal(&fan_cond);
    pthread_mutex_unlock(&fan_mutex);
    return 0;
}
//设置线程的优先级，必须在子线程
int fan_thread_setpriority(int priority){
    struct sched_param sched;
    bzero((void*)&sched, sizeof(sched));
//    const int priority1 = (sched_get_priority_max(SCHED_RR) + sched_get_priority_min(SCHED_RR)) / 2;
    sched.sched_priority=priority;
    //SCHED_OTHER(正常,非实时)SCHED_FIFO(实时，先进先出)SCHED_RR(实时、轮转法)
    pthread_setschedparam(pthread_self(), SCHED_RR, &sched);
    return 0;
}
```
### 5 其他线程方法
```C
//return=0:线程存活。ESRCH：线程不存在。EINVAL：信号不合法。
int kill_ret=pthread_kill(pid, 0);//测试线程是否存在
printf("线程状态：%d\n",kill_ret);
if(kill_ret==0){
    //关闭线程
    pthread_cancel(pid);
}


pthread_equal(pid, pid1);//比较两个线程ID是否相同


//函数执行一次
pthread_once_t once = PTHREAD_ONCE_INIT;
pthread_once(&once, test1);

```

