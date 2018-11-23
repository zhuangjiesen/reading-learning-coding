# HashMap误区
### 1.初始化(jdk1.8)
#### HashMap调用构造函数时，只会初始化参数
```
如：
HashMap map = new HashMap();
//这时候默认创建的table(数组)是空的
所以HashMap严谨的说默认是空的，只有第一次put时，才会初始化数组为16的table

```


### 2.初始化时给定 initialCapacity
#### 并不是给多少容量是多少
#### 原因是因为计算hash取模时，用的是 (数组length - 1) & hash 即位运算，而这个算法是基于length是2的整数次幂才能实现的
#### 所以初始化给定容量的时候取2的整数次幂
```
    /**
     * Returns a power of two size for the given target capacity.
     * 该方法通过计算传入的 initialCapacity 算出符合的 2次幂的容量
     */
    static final int tableSizeFor(int cap) {
        int n = cap - 1;
        n |= n >>> 1;
        n |= n >>> 2;
        n |= n >>> 4;
        n |= n >>> 8;
        n |= n >>> 16;
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    }
```


