# Redis 并不是完全单线程 -> Redis的Bio 介绍
```

1.UNLINK(4.0.0后才有) 的内存清理是bio线程实现
2.aof的fsync刷盘是bio线程实现

```
### 背景:
```
日常翻看redis官网，发现4.0.0更新了一个命令 unlink 

执行:
UNLINK key1 key2 key3

官网解释
https://redis.io/commands/unlink
摘录：
Time complexity: O(1) for each key removed regardless of its size. Then the command does O(N) work in a different thread in order to reclaim memory, where N is the number of allocations the deleted objects where composed of.
This command is very similar to DEL: it removes the specified keys. Just like DEL a key is ignored if it does not exist. However the command performs the actual memory reclaiming in a different thread, so it is not blocking, while DEL is. This is where the command name comes from: the command just unlinks the keys from the keyspace. The actual removal will happen later asynchronously.
翻译:
时间复杂度: 与要删除的对象(键 -> key)的大小无关 O(1)。然后这个命令在另外(different)一个线程执行了 O(N) 复杂度的任务为了释放(reclaim)内存，N代表是命令指定的需要删除的对象(allocations)数量。
这个命令与DEL 非常相似：DEL 命令的执行同时也删除了指定的键。与DEL相同，unlink 在键不存在的时候，不会执行任何操作(前提当然查看一把了键存不存在咯。。。)。然而这个命令(UNLINK)的释放(回收)内存的操作是在另外的线程(bio -> bio.c/bio.h)中做的，所以unlink命令并不会导致阻塞(DEL删除一个大对象的时候，可能导致阻塞)。这也是命令为啥叫做UNLINK的原因：UNLINK只是取消了键和内存之间的引用，真正清除操作可以异步执行。

```

### TIPS（redis源码查看心得）
```
redis是c语言编写，作为Java程序员可能有点吃力(好久没写过c)
但是好在redis 源码简洁，阅读性强，可以直接get到它的逻辑
当然只是为了了解实现的原理！！

1.可以下载 clion 编译器
2.就像在idea一样，比如想查看 unlink 的源码实现可以搜 unlinkCommand(xxxCommand)
3.在server.h和server.c中会看到声明和实现

```

### 通过跟代码会到 lazyfree.c -> dbAsyncDelete()  函数实现了异步删除功能

### 其实注释写得很清楚-> 释放内存过程太慢，真没必要阻塞主线程，源码：
#### 重点在于 bioCreateBackgroundJob(BIO_LAZY_FREE,val,NULL,NULL);
```
#define LAZYFREE_THRESHOLD 64
int dbAsyncDelete(redisDb *db, robj *key) {
    /* Deleting an entry from the expires dict will not free the sds of
     * the key, because it is shared with the main dictionary. */
    if (dictSize(db->expires) > 0) dictDelete(db->expires,key->ptr);

    /* If the value is composed of a few allocations, to free in a lazy way
     * is actually just slower... So under a certain limit we just free
     * the object synchronously. */
    dictEntry *de = dictUnlink(db->dict,key->ptr);
    if (de) {
        robj *val = dictGetVal(de);
        size_t free_effort = lazyfreeGetFreeEffort(val);

        /* If releasing the object is too much work, let's put it into the
         * lazy free list. */
        if (free_effort > LAZYFREE_THRESHOLD) {
            atomicIncr(lazyfree_objects,1);

            /* ！！！重点是发现了这个 ！！！！*/
            bioCreateBackgroundJob(BIO_LAZY_FREE,val,NULL,NULL);
            dictSetVal(db->dict,de,NULL);
        }
    }

    /* Release the key-val pair, or just the key if we set the val
     * field to NULL in order to lazy free it later. */
    if (de) {
        dictFreeUnlinkedEntry(db->dict,de);
        if (server.cluster_enabled) slotToKeyDel(key);
        return 1;
    } else {
        return 0;
    }
}

```

#### 好奇心驱使,全局搜了 BIO_LAZY_FREE 
#### 发现了 bio.h/bio.c 
```
在 bio.c 中发现了pthread语句(创建线程(组)而且不止一个线程)



```
### 源码小段:
```
/* Initialize the background system, spawning the thread. */
void bioInit(void) {
    pthread_attr_t attr;
    pthread_t thread;
    size_t stacksize;
    int j;

    /* Initialization of state vars and objects */
    /* 初始化状态参数 和 对象 -> 锁、和 condition用来wait 和 signal(notify) */
    for (j = 0; j < BIO_NUM_OPS; j++) {
        pthread_mutex_init(&bio_mutex[j],NULL);
        pthread_cond_init(&bio_newjob_cond[j],NULL);
        pthread_cond_init(&bio_step_cond[j],NULL);
        /* 这是个数组， bio的任务队列 */
        bio_jobs[j] = listCreate();
        bio_pending[j] = 0;
    }

    /* Set the stack size as by default it may be small in some system */
    /* 设置栈空间大小 */
    pthread_attr_init(&attr);
    pthread_attr_getstacksize(&attr,&stacksize);
    if (!stacksize) stacksize = 1; /* The world is full of Solaris Fixes */
    while (stacksize < REDIS_THREAD_STACK_SIZE) stacksize *= 2;
    pthread_attr_setstacksize(&attr, stacksize);

    /* Ready to spawn our threads. We use the single argument the thread
     * function accepts in order to pass the job ID the thread is
     * responsible of. */
     /*
     	初始化线程
     */
    for (j = 0; j < BIO_NUM_OPS; j++) {
        void *arg = (void*)(unsigned long) j;
        if (pthread_create(&thread,&attr,bioProcessBackgroundJobs,arg) != 0) {
            serverLog(LL_WARNING,"Fatal: Can't initialize Background Jobs.");
            exit(1);
        }
        bio_threads[j] = thread;
    }
}
```

### 所以
```
通过 bio 源码 bio.h bio.c 查看
1.redis并不是单一线程执行的。
2.bio -> background io，通过后台运行线程，处理background 任务

然后：
bio.h 中定义了3个后台任务：
#define BIO_CLOSE_FILE    0 /* Deferred close(2) syscall. */ 
	-> 这个不清楚
#define BIO_AOF_FSYNC     1 /* Deferred AOF fsync. */ 
	-> aof文件的追加刷盘(fsync函数)同步 -> 延迟刷盘时，调用fsync()函数，直接等待磁盘数据刷好(落盘)才返回
#define BIO_LAZY_FREE     2 /* Deferred objects freeing. */
	-> (4.0后新增,说明之后优化redis后台任务也可能新增 )就是后台释放内存操作，比如 unlink命令， 或者大对象的渐进式释放 dbAsyncDelete()方法

bio.h/bio.c的一些基本实现方法：

bioInit() 
1.初始化线程pthread 和任务队列 bio_jobs
2.在线程中运行 bioProcessBackgroundJobs() 方法(死循环) 
	1.就是类似taskqueue(bio_jobs任务列表) -> 如果没有任务就wait(休眠lock)
	2.如果有任务就唤醒(signal)执行，通过线程锁执行(lock)

3.bioCreateBackgroundJob() 主线程创建后台任务，唤醒后台任务线程执行任务


```
### bytheway
```
aof 的 everysec 的策略，丢命令最多2秒(不是1秒)，当aof的fsync刷盘没成功，第二秒fsync()时会阻塞，这时候redis下线，导致2秒的数据丢失

```


