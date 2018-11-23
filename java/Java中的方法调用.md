# Java中的方法调用

### 概念

```
方法在线程中执行
虚拟机在每个线程中创建一个 java虚拟机栈 -> 后入先出的原理

每个方法的开始与结束对应了栈中的栈帧创建与销毁
栈帧：操作数栈，局部变量表，动态链接，返回地址
栈容量可以由虚拟机参数指定，允许动态扩展
栈容量到达最大值时，无法分配给下一个方法执行空间 -> StackOverflowError
内存无法分配栈空间时 -> OOM

方法正常结束 -> 方法返回值压入调用该方法的操作数栈中，并返回执行控制权

方法异常结束 -> 方法执行过程中发生异常，并且没有对方法进行异常处理 （try catch or throws exception ）方法异常返回，没有返回值，返回执行权(如 IndexOutOfBoundException , NullPointerException 等)

方法异常返回时，若调用者同时也并未对异常进行处理，也会将异常继续往它调用者抛，有调用者对异常进行处理或者线程执行结束

动态链接
每个栈帧都包含一个指向当前方法所在类的运行时常量池的引用，以便对当前方法的代码进行动态链接。
类方法中要调用其他方法，或者访问成员变量，则需要通过符号引用，动态链接可以将这些符号引用，转化成直接引用 （类初始化过程中的解析步骤）





```


### 方法代码与字节码


```
 public static void doTest3() {
        System.out.println("doTest3.....before");



        try {
        //构造异常
            byte[] buf = new byte[10];
            buf[11] = 2;
        } catch (RuntimeException e1) {
            System.out.println(" doTest3... i am catching an runtimeexception ...");
            throw  e1;
        } catch (Exception e) {
            System.out.println(" doTest3... i am catching an exceotion ...");
            throw  e;
        }



        System.out.println("doTest3.....after...");

    }


```

```


  public static void doTest3();
    descriptor: ()V
    // 访问标识符，若是同步方法，ACC_SYNCHRONIZED 也是出现在这里
    flags: ACC_PUBLIC, ACC_STATIC
    // 方法的指令内容
    Code:
      stack=3, locals=1, args_size=0
         0: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
         3: ldc           #15                 // String doTest3.....before
         5: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
         8: bipush        10
        10: newarray       byte
        12: astore_0
        13: aload_0
        14: bipush        11
        16: iconst_2
        17: bastore
        18: goto          43
        21: astore_0
        22: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
        25: ldc           #17                 // String  doTest3... i am catching an runtimeexception ...
        27: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
        30: aload_0
        31: athrow
        32: astore_0
        33: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
        36: ldc           #18                 // String  doTest3... i am catching an exceotion ...
        38: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
        41: aload_0
        42: athrow
        43: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
        46: ldc           #19                 // String doTest3.....after...
        48: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
        51: return
        // 异常处理器 try catch 
      Exception table:
      /*from : (try开始) 的指令
        to :  到catch 前一个指令并不包括18
        
        target : catch中要执行的代码
        type : 对应处理的异常类型
        
      */ 
         from    to  target type
             8  18/*  */    21   Class java/lang/RuntimeException
             8    18    32   Class java/lang/Exception
      LineNumberTable:
        line 56: 0
        line 61: 8
        line 62: 13
        line 69: 18
        line 63: 21
        line 64: 22
        line 65: 30
        line 66: 32
        line 67: 33
        line 68: 41
        line 73: 43
        line 75: 51
        
      LocalVariableTable:
      
      /*
      局部表
      start 0的时候是方法参数，实例方法的第一个参数是this 代表该方法的实例
      slot 是槽位long double 占用2个槽位
      name 实际代码中定义的参数名称
      Signature 参数类型
    
      
      */
        Start  Length  Slot  Name   Signature
           13       5     0   buf   [B
           22      10     0    e1   Ljava/lang/RuntimeException;
           33      10     0     e   Ljava/lang/Exception;

```


### 异常处理器

```
Java虚拟机执行的每个方法都会配有 0 或者多个异常处理器 （Exception table ）,描述了方法代码中的有效作用范围(from,to )，能处理的异常类型，以及异常处理的代码位置(target)
发生异常时，虚拟机搜索当前方法包含的异常处理器列表，找到处理该异常的异常处理器，把代码控制权转向异常处理器中描述的处理异常的分支之中(target)
若未找到异常处理器，并且程序执行中发生了异常，当前方法的操作数栈和局部变量表将被丢弃，随后对应的栈帧出栈，并恢复调用者的栈帧中，未被处理的异常将在方法调用者的栈帧中重新被抛出，直到到达方法调用链的顶端还没有异常处理器，整个执行线程将被终止
异常处理器的搜索顺序是按先后定义查找的



```

