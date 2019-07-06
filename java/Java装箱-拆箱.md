# Java装箱/拆箱

### 代码示例：

```


    public static void main(String[] args) throws Exception {
        Integer a = 180;
        Integer b = 180;
        int c = 180;
        System.out.println(a == b);
        System.out.println(a == c);
    }


```

### 结果：
```
false
true

```

### 原因
```
1.IntegerCache是在Integer中的内部类。默认缓存-127~128 的Integer对象，所以大于的话，对比的是类对象，不相等，所以是false
2.Integer 和 int 进行比较时，jvm编译器会将代码在编译时，进行拆箱，等于 Integer.intValue(a) == c 所以是false

```


### 字节码片段
```

  public static void main(java.lang.String[]) throws java.lang.Exception;
    descriptor: ([Ljava/lang/String;)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=3, locals=4, args_size=1
         0: sipush        180
         3: invokestatic  #2                  // Method java/lang/Integer.valueOf:(I)Ljava/lang/Integer;
         6: astore_1
         7: sipush        180
        10: invokestatic  #2                  // Method java/lang/Integer.valueOf:(I)Ljava/lang/Integer;
        13: astore_2
        14: sipush        180
        17: istore_3
        18: getstatic     #3                  // Field java/lang/System.out:Ljava/io/PrintStream;
        21: aload_1
        22: aload_2
        23: if_acmpne     30
        26: iconst_1
        27: goto          31
        30: iconst_0
        31: invokevirtual #4                  // Method java/io/PrintStream.println:(Z)V
        34: getstatic     #3                  // Field java/lang/System.out:Ljava/io/PrintStream;
        37: aload_1
        38: invokevirtual #5                  // Method java/lang/Integer.intValue:()I
        41: iload_3
        42: if_icmpne     49
        45: iconst_1
        46: goto          50
        49: iconst_0
        50: invokevirtual #4                  // Method java/io/PrintStream.println:(Z)V
        53: return
      LineNumberTable:
        line 56: 0
        line 57: 7
        line 58: 14
        line 59: 18
        line 60: 34
        line 61: 53
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      54     0  args   [Ljava/lang/String;
            7      47     1     a   Ljava/lang/Integer;
           14      40     2     b   Ljava/lang/Integer;
           18      36     3     c   I
      StackMapTable: number_of_entries = 4
        frame_type = 255 /* full_frame */
          offset_delta = 30
          locals = [ class "[Ljava/lang/String;", class java/lang/Integer, class java/lang/Integer, int ]
          stack = [ class java/io/PrintStream ]
        frame_type = 255 /* full_frame */
          offset_delta = 0
          locals = [ class "[Ljava/lang/String;", class java/lang/Integer, class java/lang/Integer, int ]
          stack = [ class java/io/PrintStream, int ]
        frame_type = 81 /* same_locals_1_stack_item */
          stack = [ class java/io/PrintStream ]
        frame_type = 255 /* full_frame */
          offset_delta = 0
          locals = [ class "[Ljava/lang/String;", class java/lang/Integer, class java/lang/Integer, int ]
          stack = [ class java/io/PrintStream, int ]
    Exceptions:
      throws java.lang.Exception


```

### 总结
```
1. Integer a = 180; 编译的时候会变成 Integer a = Integer.valueOf(180)，底层源码是new Integer()

2.a == c 的时候，编译器会自动拆箱，等于 Integer.intValue(a) == c 所以相等
```

