# Handler面试相关问题汇总

## 子线程到主线程通信有哪些方式?
以下不管哪种方式，底层原理都是Handler
- RxJava
- EventBus
- Broadcast
- View.post(Runnable)/View.postDelayed(Runnable,long)
- Activity.RunOnUiThread(Runnable)
- AsyncTask
- ...

## Handler原理
- 子线程（生产者） handler.sendMessage(msg)/handler.post(Runnable) -> messageQueue.enqueueMessage() 消息入队列过程
- 主线程（消费者） AMS(ActivityManagerService) -----发送创建进程的请求---> Zygote.fork() --反射reflect--> ActivityThread.main()-> Looper.loop() -> messageQueue.next() -> handler.dispatchMessage() -> handle.handleMessage()

此方案只是一个浅层的实现逻辑，深层得益于线程间内存共享
### 线程间内存共享
### 跨进程也可以共享内存

## Handler内存泄露的原因是什么？
内存泄露：
- 错误引用，无法回收
- 生命周期短的对象引用了生命周期长的对象
- 匿名内部类Handler自动持有了外部类Activity的引用

这几个回答都比较片面，实际与JVM息息相关
### JVM（引用计数，可达性分析，垃圾回收算法）
GCRoot 直接或者间接持有了对象的引用，这个对象就不能被回收
- 持有关系 static threadLocal -> looper -> messageQueue -> msg -> handler -> activity
#### 解决方案：
本质是找到引用链，然后打断它:
1. 所以handler必须是static的，静态的匿名内部类不会持有外部类的引用(详见《Java编程思想》)
2. remove message,调用MessageQueue.removeMessages()方法
#### static关键字
《Java编程思想》
### 什么类型是GCRoot
常量，静态变量
### handler源码中
```
msg.target = this;
mQueue = mLooper.mQueue;
```
### Looper源码中
```
mQueue = new MessageQueue(quitAllowed);
sThreadLocal.set(new Looper(quitAllowed));
static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
```

## MessageQueue有上限吗？
## MessageQueue
- 单链表，按执行时间先后排序，每次先执行第一个消息，只有前一个执行完成了，才执行下一个
- Activty（AMS来添加的msg）和fragment的生命周期都是handler来控制的，甚至屏幕的刷新(60HZ 1000/60=14ms)都是通过handler发送msg控制，所以不能设置上限
- AMS->Binder(跨进程)->ApplicationThread->ActivityThread

## MessageQueue能不能用阻塞队列？
不能。
### 阻塞队列（概念，线程池管理，并发）
1. 队列有大小
2. 队列空的时候，取消息会等待


## Handler如何处理发送延时消息
- next->等待时间->nativePollOnce(JNI zygote注册)->epoll_wait
- 正常是 nativeWake->epoll
### epoll
监听 红黑树 （hashmap），减少遍历时间
