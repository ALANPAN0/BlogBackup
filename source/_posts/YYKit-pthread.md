---
title: YYKit源码分析---pthread
date: 2016-06-21 17:43:18
categories: iOS
tags: [YYKit, pthread]
---
<!--mored-->

> 大家都知道ibireme的[YYKit](https://github.com/ibireme/YYKit)很强大，个人也特别佩服ibireme。大神常常教导我们这样的小白说：多读源码能够大幅度的提高功力。 <p>每当项目上线后，需求还没有下来时，都会有一段闲暇时间。这段时间学习是极佳的。YYKit这个框架刚开始看的时候就遇到pthread这个玩意，之前很少接触。在此，记录自己的所学所得，并分享给大家。

## 先来看下YY定义的宏
```
static inline void pthread_mutex_init_recursive(pthread_mutex_t *mutex, bool recursive) {
#define YYMUTEX_ASSERT_ON_ERROR(x_) do { \
__unused volatile int res = (x_); \
assert(res == 0); \
} while (0)
    assert(mutex != NULL);
    if (!recursive) {
        YYMUTEX_ASSERT_ON_ERROR(pthread_mutex_init(mutex, NULL));
    } else {
        pthread_mutexattr_t attr;
        
        YYMUTEX_ASSERT_ON_ERROR(pthread_mutexattr_init (&attr));
        YYMUTEX_ASSERT_ON_ERROR(pthread_mutexattr_settype (&attr, PTHREAD_MUTEX_RECURSIVE));
        YYMUTEX_ASSERT_ON_ERROR(pthread_mutex_init (mutex, &attr));
        YYMUTEX_ASSERT_ON_ERROR(pthread_mutexattr_destroy (&attr));
    }
#undef YYMUTEX_ASSERT_ON_ERROR
}   
```
大神的代码都是晦涩难懂的，看到这段代码后劳资突然产生了好几个问题：

- 这个方法是用来干嘛的呢？
- pthread_mutex_t是什么鬼？
- pthread_mutexattr_t是用来配置pthread_mutex_t的吗？

## 解读
#### 功能
其实就是创建个互斥线程，并没有想象中的可怕
#### pthread_mutex_t
`int pthread_mutex_init(pthread_mutex_t * __restrict,
		const pthread_mutexattr_t * __restrict);` 是用这个函数创建出来的。函数是以动态的方式创建互斥锁的，参数attr指定了新建互斥锁的属性。<br>`recursive`这个`bool`值为false时，attr为空，则使用默认的互斥锁属性，默认属性为快速互斥锁。<br>`recursive`这个`bool`值为true时，配置互斥锁属性创建相应的互斥锁。
#### YYMUTEX_ASSERT_ON_ERROR
断言来进行检查错误，所有操作返回非0时，表示有异常错误发生
#### Mutex type attributes
**PTHREAD_MUTEX_NORMAL**：不进行deadlock detection（死锁检测）。当进行relock时，这个mutex就导致deadlock。对一个没有进行lock或者已经unlock的对象进行unlock操作，结果也是未知的。<br>**PTHREAD_MUTEX_ERRORCHECK**：和PTHREAD_MUTEX_NORMAL相比，PTHREAD_MUTEX_ERRORCHECK会进行错误检测，以上错误行为都会返回一个错误。<br>**PTHREAD_MUTEX_RECURSIVE**：和semaphore（信号量）有个类似的东西，mutex会有个锁住次数的概念，第一次锁住mutex的时候，锁住次数设置为1，每一次一个线程unlock这个mutex时，锁住次数就会减1。当锁住次数为0时，其他线程就可以获得该mutex锁了。同样，对一个没有进行lock或者已经unlock的对象进行unlock操作，将返回一个错误。<br>**PTHREAD_MUTEX_DEFAULT**：默认PTHREAD_MUTEX_NORMAL。

## 再看看YY如何使用该宏
```
- (YYImageFrame *)frameAtIndex:(NSUInteger)index decodeForDisplay:(BOOL)decodeForDisplay {
    YYImageFrame *result = nil;
    pthread_mutex_lock(&_lock);
    result = [self _frameAtIndex:index decodeForDisplay:decodeForDisplay];
    pthread_mutex_unlock(&_lock);
    return result;
}
```
**这边为了防止多线程资源抢夺的问题，先进行lock下，等数据操作完毕后释放unlock，有没有一种豁然开朗的感觉呢<br>平时我们在多线程操作的时候也可以使用NSLock、synchronized来进行加锁，yy使用了更加偏向底层的pthread**

## pthread_t和NSThread
两者都是用来操作线程的对象，平时我们使用上层的NSThread比较多，像[NSThread mainThread]获取主线程，[NSThread currentThread] 获取当前线程。pthread_t和NSThread是一一对应的，同样可以通过pthread_main_thread_np() 、pthread_self()来获取。NSThread只是对pthread_t的一层封装而已。

## 实战
- 声明函数

```
void *func(void *argu) {
    char *m = (char *)argu;

    pthread_mutex_lock(&mutex);
    while (*m != '\0') {
        printf("%c", *m);
        fflush(stdout);
        sleep(3);
        m++;
    }
    printf("\n");
    pthread_mutex_unlock(&mutex);
    return 0;
}
```

- mutex使用 

```
    int rc1, rc2;
    
    char *str1 = "Hi";
    char *str2 = "Boy!";
    
    pthread_t thread1, thread2;
    pthread_mutex_init(&mutex, NULL);

    if ((rc1 = pthread_create(&thread1, NULL, func, str1))) {
        fprintf(stdout, "thread1 creat fail : %d \n!", rc1);
    }
    if ((rc2 = pthread_create(&thread2, NULL, func, str2))) {
        fprintf(stdout, "thread2 creat fail : %d \n!", rc2);
    }

    // https://developer.apple.com/legacy/library/documentation/Darwin/Reference/ManPages/man3/pthread_join.3.html#//apple_ref/c/func/pthread_join
    // 等待一个线程的结束，当函数返回时，被等待的线程资源被收回。若线程已经被收回，那么该函数会立即返回
    pthread_join(thread1, NULL);
    pthread_join(thread2, NULL);

	printf("这边只有线程被回收后才会执行！");
```
- 可以帮pthread_mutex_lock(&mutex)和pthread_mutex_unlock(&mutex)注释掉看下打印<p>

> 感谢大家花费时间来查看这篇blog，需要下载demo的同学请猛戳[Git](https://github.com/PanXianyue/BlogDemo)。