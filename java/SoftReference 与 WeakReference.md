# SoftReference 与 WeakReference

### 软引用和弱引用

#### 准备动作：

```
设置虚拟机参数
设置堆内存，还有年轻代大小
gc信息详情打印
-Xmx32m -Xms16m -Xmn10m -XX:+PrintGCDetails -XX:+PrintGCDateStamps
```


### SoftReference

代码示例：

```

        byte[] buf = new byte[1024 * 1024 * 5];
        SoftReference<byte[]> w1 = new SoftReference<byte[]>(buf);
        System.out.println("1. w1 : "+w1.get());
        //当以下语句注释掉时，不管gc触发没触发，SoftReference 都不会为空
        buf = null;
        //这个值还是有值的
        System.out.println("2. w1 : "+w1.get());
        //触发了gc
        System.gc();
        //这个值还是存在
        System.out.println("3. w1 : "+w1.get());
        // 模拟大内存
        byte[] buf2 = new byte[1024 * 1024 * 10];
//        byte[] buf3 = new byte[1024 * 1024 * 10];
        byte[] buf4 = new byte[1024 * 1024 * 5];
        //触发了gc ， 侧面证明了内存不足时才会优先的回收 软引用
        System.gc();
        //这时候null
        System.out.println("4. w1 : "+w1.get());


```

### WeakReference

代码示例：

```

        byte[] buf = new byte[1024 * 1024 * 5];
        WeakReference<byte[]> w1 = new WeakReference<byte[]>(buf);
        System.out.println("1. w1 : "+w1.get());
        buf = null;
        //这个值还是有值的
        System.out.println("2. w1 : "+w1.get());
        System.gc();
        //这个值是null , 证明了每次gc WeakReference 弱引用都是被回收掉的
        System.out.println("3. w1 : "+w1.get());

```


### 总结

#### 首先 System.gc(); 调用不一定触发gc ，但是以上代码碰巧都触发的gc 所以达到验证的前提条件
#### 被引用对象 buf 如果还带有强引用，证明 buf 这个对象还存活，则两种引用都不能回收这个对象 （buf = null; 这段代码注释掉，软引用和弱引用都不会被gc 回收）

#### 软引用 内存不足时才会优先的回收
#### 弱引用 每个gc周期 都会进行回收

