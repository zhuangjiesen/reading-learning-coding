J.U.C 队列集合解析
lock:
ArrayBlockingQueue
LinkedBlockingQueue
PriorityBlockingQueue
DelayQueue

cas ：
ConcurrentLinkedQueue size() 是遍历队列取出大小然而并发情况下统计数据不精确
LockSupport：
SynchronousQueue
LinkedTransferQueue





用aqs 锁 CountDownLatch
用法： 分执行线程(用来countdown) 等待线程（await）
参数 count 数量下的线程都执行完 ， await方法才能继续执行否则阻塞
内部类 Sync extends AQS 实现共享锁的功能 ， 所以多个线程可以同时await 
三个方法
1.构造函数 ： 设置了等待线程的大小 -> aqs 的 setState 设置状态 （其实就是加了锁）

2.await : 等待方法 -> 调用的 Sync 的 tryAcquireShare 
判断 state == 0 
true -> 1 获取锁继续执行
false -> -1 进入aqs队列，阻塞(waiting)状态

3.countDown : 线程执行完释放 -> 调用 Sync 的 tryReleaseShare 
判断 state == 0 
true -> 释放共享锁
false -> state - 1 return false 不执行释放锁

用lock 锁 CyclicBarrier
可以reset 重置 也有 count 
用的是 lock 与 condition

用法： 全部线程都等待，直到最后一个线程执行，所有线程释放

reset : 
lock.lock 先加锁
condition.signalAll 释放所有节点
状态变更
count 重置

await :
lock.lock 获取锁
count -- 
count == 0 
true : signalAll 
false : await 











