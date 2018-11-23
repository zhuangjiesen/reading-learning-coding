# ThreadLocal 的理解

```
测试程序以及注释解释

public class HelloWorld {


    private static ThreadLocal<String> threadLocal = new ThreadLocal<>();


    public static void main(String[] args) throws Exception {

        Thread t1 = new Thread(new Runnable() {
            @Override
            public void run() {
                /*
                设置属性，其实是设置到 Thread 变量中的 threadLocals 属性
                ThreadLocal 类等于是个装饰者模式，没有具体实现的数据结构存储，set get 都是去Thread 下获取数据的
                *
                * */
                threadLocal.set("zhuangjiesen");

                ThreadHelper.sleep(30 * 1000);

            }
        });
        t1.start();
        ThreadHelper.sleep(300);

        Unsafe unsafe = null;
        /*
        * 设置成null 为了测试 ThreadLocal 的 OOM 内存溢出的问题探究
        * 其实 ThreadLocal 源码中的 ThreadLocalMap 是继承于 WeakReference 类型修饰的key ，说明了
        * set get 时，是用ThreadLocal 的内存地址(toString() )去具体的Thread下获取线程下的变量属性
        * 所以 Thread 消亡后，自然对应的线程局部数据相应没了
        * 然而还有另一个情况：
        * 通常系统运用了线程池，所以线程是一直保活的，ThreadLocal 比 Thread 先被回收了，所以变量可能已经不再需要了，Thread中的
        * threadLocals 的对应的 ThreadLocal 的key (WeakReference) 被回收了，但是value 还是存在的，所以不断的使用新的ThreadLocal
        * 或 没有 remove() 调用，可能有OOM内存溢出问题
        *
        * */
        threadLocal = null;
        System.gc();
        Field field = Thread.class.getDeclaredField("threadLocals");
        field.setAccessible(true);
        Object threadLocals = field.get(t1);

        /*
        * 这里可以打断点，会发现 ThreadLocal的对象已经Null 而且被回收了，但是这个Thread 的内部局部变量
        * 的属性 threadLocals 中还存在响应的数据
        *
        * */
        System.out.println("---field : " + threadLocals);

        System.out.println("-----");

    }





}


```

