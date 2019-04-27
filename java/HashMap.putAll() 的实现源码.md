# HashMap.putAll() 的实现源码
### 背景
```
写了个netty-websocket框架，用了ConcurrentHashMap 作自定义心跳处理(微信页面上用websocket时，页面关闭不会有close事件)

但是担心某一时刻突然请求量上去 ConcurrentHashMap 的容量直接上升，但是大部分的时候，请求量是不大的，所以决定作个缩容操作!! 
这时候就想看看 HashMap.putAll() 的源码了

```

### 缩容操作
```

/**
 * @param
 * @Author: zhuangjiesen
 * @Description:
 * @Date: Created in 2019/3/19
 */
public class CommonMain {

    private static Map<String, Object> heatMap = new ConcurrentHashMap<>();
    /** 老map **/
    private static Map<String, Object> oldHeatMap = null;

    /** 是否正在缩容**/
    private static volatile AtomicBoolean isResizeING = new AtomicBoolean(false);
    private static final int RESIZE_LIMIT_SIZE = 100;

    public static void main(String[] args) {

        //容量大于缩容临界
        resizeMap();
        //继续操作无影响
        heatMap.get("xxxx");
        heatMap.put("xxx", "xxx");
        
    }

    /**
     * 缩容操作
     *
     * @author zhuangjiesen
     * @date 2019/4/27 9:02 PM
     * @param
     * @return
     */
    private static void resizeMap() {
        if (heatMap.size() > RESIZE_LIMIT_SIZE) {
            try {
                if (isResizeING.compareAndSet(true, false)) {
                    oldHeatMap = heatMap;
                    //直接初始化 -> 源码里也是for 循环set进去的
                    heatMap = new ConcurrentHashMap<>(oldHeatMap);
                }
            } finally {
                //完成缩容
                isResizeING.set(false);
            }
        }
    }
}

```


### Map 中把别的map加入的方法有2个
```
    public HashMap(Map<? extends K, ? extends V> m) {
        this.loadFactor = DEFAULT_LOAD_FACTOR;
        putMapEntries(m, false);
    }
    
    
   /**
     * Copies all of the mappings from the specified map to this map.
     * These mappings will replace any mappings that this map had for
     * any of the keys currently in the specified map.
     *
     * @param m mappings to be stored in this map
     * @throws NullPointerException if the specified map is null
     */
    public void putAll(Map<? extends K, ? extends V> m) {
        putMapEntries(m, true);
    }

```
### 调用的是同一个方法
```


    /**
     * Implements Map.putAll and Map constructor
     *
     * @param m the map
     * @param evict false when initially constructing this map, else
     * true (relayed to method afterNodeInsertion).
     */
    final void putMapEntries(Map<? extends K, ? extends V> m, boolean evict) {
        int s = m.size();
        if (s > 0) {
            /*
                当前map还没初始化，比如是构造函数进入或者是初始化后还没put任何数据的时候
                需要先初始化大小
            */
            if (table == null) { // pre-size
                float ft = ((float)s / loadFactor) + 1.0F;
                int t = ((ft < (float)MAXIMUM_CAPACITY) ?
                         (int)ft : MAXIMUM_CAPACITY);
                if (t > threshold)
                    //初始化数组的大小
                    threshold = tableSizeFor(t);
            }
            /*
                通过putAll() 并且已经初始化后的
                插入的map 直接大于 临界值，就直接扩容了
            */
            else if (s > threshold)
                resize();
            //直接遍历插入，没什么特殊操作
            for (Map.Entry<? extends K, ? extends V> e : m.entrySet()) {
                K key = e.getKey();
                V value = e.getValue();
                putVal(hash(key), key, value, false, evict);
            }
        }
    }

```

