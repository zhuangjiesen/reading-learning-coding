1.新的api 支持http2.0 / websocket 废弃 HttpUrlConnection 的使用
2. 压缩字符串对象，有些字符串只需要更少的内存空间，和一些字符串对象的复用 修改了编码格式，使得占用的空间减小
3. g1回收器中优化了字符串对象的存储方式 ->原因是堆内存中发现字符串对象占用的空间太大 ， 共享字符串对象
4. 默认G1回收器

官网描述 ：http://openjdk.java.net/projects/jdk9/



Integer IntegerCache 默认是127 可以通过设置虚拟机参数
-XX:AutoBoxCacheMax=20000 增大这个值
对应

```
属性:
java.lang.Integer.IntegerCache.high
```


ThreadLocal 原理
并不是通过Thread的id 然后在ThreadLocal中维护一个 KEY 是WeakReference 的 map
源码中是通过ThreadLocal 类去操作 Thread 类下的一个 ThreadLocalMap(threadLocals) 的实例变量，
即：
在set 方法中，用该ThreadLocal的实例作为key ，放到对应线程的 ThreadLocalMap 属性中，
取也是同样方法，则变量会通过 Thread 对象实例的消亡而被gc 回收
