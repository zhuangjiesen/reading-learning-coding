# 分布式锁实现 Java + redis (一)

分布式环境下很多系统需要使用分布式锁
目的不多说了


### redis实现分布式锁

```
官网描述:
https://redis.io/topics/benchmarks
在 Pitfalls and misconceptions 小标题下
截取

Redis is, mostly, a single-threaded server from the POV of commands execution (actually modern versions of Redis use threads for different things)
意思是redis 在执行命令的时候是单线程执行的，所以能保证命令执行的原子性
```
### 所以可以使用redis 实现分布式锁

### 另外:
```
看了网上许多的基于 redis 的分布式锁实现 ， 大多都是用
1.setNx加锁 + DEL命令解锁 
2. 使线程休眠一段时间(轮训判断状态) 
3. expire 键过期(貌似要解决setnx 命令后，线程出问题使得系统死锁)

```

优化解决方法
1. setNx 命令 进行加锁 
```
不多说，就是 set if not exist 
如果不存在就set 成功，返回值1
如果已经存在，set 失败 ， 返回0 
官网： https://redis.io/commands/setnx
```
2. 无需使用线程休眠 + 轮训 key 的状态来判断
```
通过 redis 的 Redis Keyspace Notifications
redis 通过设置键控件状态的监听通知客户端，也就是可以订阅(subscribe channels)
由于是通过 DEL mykey 进行释放锁操作，就可以监听 redis 的键DEL 事件

redis 配置
redis 配置文件下在 EVENT NOTIFICATION 区域
配置 notify-keyspace-events "AKE" 可以监听所有键事件(All Keys Events)
当然也可以具体订阅某个事件 

redis 命令
>subscribe __keyevent@0__:del
实现订阅删除事件

可以通过执行：
> PUBLISH __keyevent@0__:del distributelocks
触发订阅事件
或者执行：
>DEL distributelocks （这里删除返回1 即删除成功才会触发）

官网：
https://redis.io/topics/notifications
https://redis.io/commands/del
----------
所以：
可以使用 redis 订阅键空间状态事件，当redis 调用DEL 释放锁， 实现通知其他线程获取锁

当然这个键空间通知跟zk的watch功能不是一样吗？

```

3.解决死锁
```
个人认为：
加个expire 过期时间是解决不了死锁问题的
我的原因是：
当你操作的是数据库数据时，你如果使用了分布式锁，又加了时间
1.如果结束前执行完成，是没有问题的
2.执行一半的情况下，键过期，导致其他(进程下的)线程，获取锁进入
当然这种情况可以拿锁后进行数据查询来判断是否一致

我的解决方式是，可以通过开启定时任务查看分布式锁的键，然后执行对应键(任务)的数据同步操作，或者是解锁操作
```

上代码





### 执行资源竞争操作

```

Thread t1 = new Thread(new Runnable() {
    @Override
    public void run() {
        DistributedLocksService distributedLocksService = applicationContext.getBean(DistributedLocksService.class);
        distributedLocksService.robResource("distributedLocksService" , Thread.currentThread());
    }
});
t1.start();
ThreadHelper.sleep(400);
Thread t2 = new Thread(new Runnable() {
    @Override
    public void run() {
        DistributedLocksService distributedLocksService = applicationContext.getBean(DistributedLocksService.class);
        distributedLocksService.robResource("distributedLocksService" , Thread.currentThread());
    }
});
t2.start();
ThreadHelper.sleep(400);
Thread t3 = new Thread(new Runnable() {
    @Override
    public void run() {
        DistributedLocksService distributedLocksService = applicationContext.getBean(DistributedLocksService.class);
        distributedLocksService.robResource("distributedLocksService" , Thread.currentThread());
    }
});
t3.start();



```

### DistributedLocksService.java

```

/**
 * Created by zhuangjiesen on 2017/12/8.
 */
public class DistributedLocksService {
    private DistributedLock distributedLock;
    /*
    *
    * 抢占资源
    * 涉及竞争
    *
    * */
    public void robResource(String key , Object params ) {
        Thread thread = (Thread)params;
        try {
            System.out.println("1. ------准备获取锁-------- : " + thread.getId());
            distributedLock.lock(key);
            // dosomething ... 
            System.out.println("2. ------成功获取锁-------- : " + thread.getId());
        } finally {
            distributedLock.unLock();
            System.out.println("3. ------成功释放锁-------- : " + thread.getId());
        }

    }
    public DistributedLock getDistributedLock() {
        return distributedLock;
    }
    public void setDistributedLock(DistributedLock distributedLock) {
        this.distributedLock = distributedLock;
    }
}


```


### DistributedLock.java 抽象分布式锁接口

```

/**
 *
 * 分布式锁接口
 * Created by zhuangjiesen on 2017/12/8.
 */
public interface DistributedLock {
    public final static String DEFAULT_LOCK_KEY = "DEFAULT_LOCK_KEY_OF_DISTRIBUTEDLOCK";
    public void lock(String key) ;
    public boolean tryLock(String key) ;
    public void unLock() ;
    public void unLock(String key) ;
    /*
    * 用来执行分布式锁的一些问题，比如lock 之后，服务挂了，重启之后其他线程都拿不到锁了
    * 为了解决这个情况，可以使用一个定时任务，检测锁状态
    * */
    public void scheduleTask();
}

```


### redis 实现分布式锁 JedisDistributedLock.java

```


/**
 *
 *
 * redis 实现分布式锁
 * Created by zhuangjiesen on 2017/12/8.
 */
public class JedisDistributedLock implements DistributedLock , ApplicationContextAware, ApplicationListener<ContextRefreshedEvent> {
    private ApplicationContext applicationContext;
    private static ThreadLocal<String> lockKeys = new ThreadLocal<>();
    private static ConcurrentHashMap<String , Object> disLocks = new ConcurrentHashMap<>();
    private JedisPool jedisPool;

    @Override
    public void lock(String key) {
        Jedis jedis = jedisPool.getResource();
        Long count = jedis.setnx( key , "1");
        if (count != null && count.longValue() > 0) {
            System.out.println("--lock--1. 获取到分布式锁.......");
            lockKeys.set(key);
        } else {
            System.out.println("--lock--2. not .......未获取到分布式锁.......");
            Object lock = null;
            if ((lock = disLocks.get(key)) == null) {
                // double check
                synchronized (disLocks) {
                    if ((lock = disLocks.get(key)) == null) {
                        lock = new Object();
                        disLocks.put(key , lock);
                    }
                }
            }
            synchronized (lock) {
                try {
                    //线程休眠
                    lock.wait();
                    System.out.println("-------------------------");
                    lock(key);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }

    @Override
    public boolean tryLock(String key) {
        Jedis jedis = jedisPool.getResource();
        Long count = jedis.setnx( key , "1");
        if (count != null && count.longValue() > 0) {
            return true;
        }
        return false;
    }

    @Override
    public void unLock() {
        try {
            String key = lockKeys.get();
            unLock(key);
        } finally {
            lockKeys.remove();
        }
    }

    @Override
    public void unLock(String key) {
        Jedis jedis = jedisPool.getResource();
        Long del = jedis.del(key);
        Object lock = null;
        if (del != null && del.longValue() > 0 && (lock = disLocks.remove(key)) != null) {
            //解锁
            synchronized (lock) {
                lock.notifyAll();
            }
        }
        System.out.println("--unLock--3. key : " + key + " del  : " + del);
    }

    @Override
    public void scheduleTask() {
        //执行轮训死锁问题
    }

    @Override
    public void setApplicationContext(ApplicationContext context) throws BeansException {
        this.applicationContext = context;
    }


    /*
    * spring 容器加载完毕调用
    *
    * */
    @Override
    public void onApplicationEvent(ContextRefreshedEvent event) {
        if (event.getApplicationContext().getParent() == null) {
            jedisPool = applicationContext.getBean(JedisPool.class);
            Jedis jedis = jedisPool.getResource();
            Thread t1 = new Thread(new Runnable() {
                @Override
                public void run() {
                    jedis.subscribe(new JedisPubSub() {
                        @Override
                        public void onMessage(String channel, String message) {
                            super.onMessage(channel, message);
                            Object lock = null;
                            System.out.println(" 1. channel : " + channel);
                            System.out.println(" 2. message : " + message);
                            if ("__keyevent@0__:del".equals(channel) && (lock = disLocks.remove(message)) != null) {
                                //解锁
                                synchronized (lock) {
                                    lock.notifyAll();
                                }
                            }
                        }
                    } , "__keyevent@0__:del");
                }
            });
            t1.start();
        }
    }
}

```





### 线程模拟执行时间 ThreadHelper.java

```

public class ThreadHelper {
    /**
     * 休眠模拟 rpc执行时间
     * @param timeßß
     */
    public static void sleep(long timeßß){
        try {

            Thread.currentThread().sleep(timeßß);
        } catch (Exception e) {
            // TODO: handle exception
            e.printStackTrace();
        }
    }
}


```



### jedis 配置

#### spring-jedis.xml
```
    <context:property-placeholder location="classpath:jedis.properties" />


    <bean id="jedisPoolConfig" class="redis.clients.jedis.JedisPoolConfig" >
        <property name="maxTotal" value="50" />
        <property name="minIdle" value="50" />
    </bean>

    <bean id="jedisPool" class="redis.clients.jedis.JedisPool" >
        <constructor-arg index="0" ref="jedisPoolConfig" />
        <constructor-arg index="1"  value="${standardalone.host}" />
        <constructor-arg index="2" value="${standardalone.port}" />
        <constructor-arg index="3" value="${standardalone.timeout}" />
        <constructor-arg index="4" value="${standardalone.password}" />
    </bean>


```


#### jedis.properties

```
standardalone.host=192.168.130.130
standardalone.port=6379
standardalone.timeout=500
standardalone.password=redis

```

#### applicationContext.xml 中配置jedis 和 分布式锁的service 

```

<bean id="distributedLocksService" class="com.java.service.DistributedLocksService" >
    <property name="distributedLock" ref="distributedLock"></property>
</bean>
<bean id="distributedLock" class="com.java.core.lock.impl.JedisDistributedLock" ></bean>


```



附上项目源码:
https://github.com/zhuangjiesen/CustomerAndProvider/tree/master/JavaProCustomer

目录:
```
处理类:
com.java.service.DistributedLocksService
分布式锁实现目录：
com.java.core.lock

main方法 DistributedLocksApp.java下
com.java.main.DistributedLocksApp.java
```

