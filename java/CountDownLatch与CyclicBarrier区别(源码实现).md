# CountDownLatch与CyclicBarrier区别(源码实现)

### 测试代码

```


/**
 * CountDownLatch
 * CyclicBarrier 区别
 *
 * 网上说 CyclicBarrier 与 CountDownLatch 区别只是一个能重置一个不行
 * 其实看过源码不止区别在此
 *
 * 调用
 * countDownLatch.countDown();
 * 线程会直接执行完，计数器减1
 *
 * countDownLatch.await(); 只会阻塞当前线程
 *
 * 而
 * 调用
 * cyclicBarrier.await();
 * 线程会阻塞，直到最后一个线程也调用 await(); 时，才能全部释放
 * 这时候会有个 惊群 的问题出现
 *
 * 所以
 * 使用 countDownLatch 会比较好
 *
 * Created by zhuangjiesen on 2018/1/24.
 */
public class CountDownLatchAndBarrierTest {


    public static void main(String[] args) {
        //保活
        Thread t4 = new Thread(new Runnable() {
            @Override
            public void run() {

                while(true) {
                    ThreadHelper.sleep(300);
                }
            }
        });
        t4.start();

//        testCountDownLatch();
        testCyclicBarrier();
    }


    public static void testCountDownLatch(){

        final CountDownLatch countDownLatch = new CountDownLatch(4);

        Thread t1 = new Thread(new Runnable() {
            @Override
            public void run() {
                System.out.println(" t1 start ....");
                countDownLatch.countDown();
                System.out.println(" t1 await over ....");
                ThreadHelper.sleep(300);
            }
        });


        Thread t2 = new Thread(new Runnable() {
            @Override
            public void run() {
                System.out.println(" t2 start ....");
                countDownLatch.countDown();
                System.out.println(" t2 await over ....");
                ThreadHelper.sleep(300);
            }
        });
        Thread t3 = new Thread(new Runnable() {
            @Override
            public void run() {
                System.out.println(" t3 start ....");
                countDownLatch.countDown();
                System.out.println(" t3 await over ....");
                ThreadHelper.sleep(300);
            }
        });
        t1.start();
        t2.start();
        t3.start();
    }


    public static void testCyclicBarrier(){

        final CyclicBarrier cyclicBarrier = new CyclicBarrier(4);
        Thread t1 = new Thread(new Runnable() {
            @Override
            public void run() {
                System.out.println(" t1 start ....");
                try {
                    cyclicBarrier.await();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } catch (BrokenBarrierException e) {
                    e.printStackTrace();
                }
                System.out.println(" t1 await over ....");
                ThreadHelper.sleep(300);
            }
        });

        Thread t2 = new Thread(new Runnable() {
            @Override
            public void run() {
                System.out.println(" t2 start ....");
                try {
                    cyclicBarrier.await();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } catch (BrokenBarrierException e) {
                    e.printStackTrace();
                }
                System.out.println(" t2 await over ....");
                ThreadHelper.sleep(300);
            }
        });
        Thread t3 = new Thread(new Runnable() {
            @Override
            public void run() {
                System.out.println(" t3 start ....");
                try {
                    cyclicBarrier.await();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } catch (BrokenBarrierException e) {
                    e.printStackTrace();
                }
                System.out.println(" t3 await over ....");
                ThreadHelper.sleep(300);
            }
        });

        t1.start();
        t2.start();
        t3.start();
    }


}


```



### 运行结果：

#### testCyclicBarrier();

```
 t1 start ....
 t3 start ....
 t2 start ....

```

#### testCountDownLatch();

```
 t1 start ....
 t3 start ....
 t2 start ....
 t3 await over ....
 t1 await over ....
 t2 await over ....

```

### 结论：
```

CyclicBarrier 源码是通过 lock / condition 
signal 和 await 的， 所以它实现的效果是阻塞当前线程

CountDownLatch 源码是设计一个 继承 aqs 的内部类，
设置 state 的值
当 countDown(); 时，是去 releaseShared(); 释放锁，
然后当state 等于 0 时，就唤醒await(); 等待的线程；
所以 CountDownLatch.countDown(); 时，线程是直接继续执行的，所以也不用进行异常处理

```