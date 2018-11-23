Java中的try finally 的原理


### 情况一： doTryFinally1() 就是try 中修改值并且return 

#### Java 代码：
```

    public static String doTryFinally1 (){
        String name = null;
        try {
            name= "zhuang";
            return name;
        } finally {
            name = "zhuang111111";
        }
    }

```
#### 字节码：  (加括号是自己的理解)

```

  public static java.lang.String doTryFinally1();
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=1, locals=3, args_size=0
         0: aconst_null   (初始化null值，进栈)
         1: astore_0      
         2: ldc           #2                  // String zhuang  
                 (int, float或String型常量值从常量池中推送至栈顶)
         4: astore_0      
                (将栈顶引用型数值存入第一个本地变量)
         5: aload_0   
                (将一个局部变量加载到操作数栈的指令)    
         6: astore_1    
                (将栈顶引用型数值存入第二个本地变量 这里等于又存了个要 return 的值到本地变量表)
         7: ldc           #3                  // String zhuang111111
         9: astore_0   
              (给name 赋值，也是 finally 块里的代码)   
        10: aload_1  
              (将第二个局部变量加载到操作数栈的指令 也就是 name = "zhuang" 的值, 所以finally 中对name 的地址进行操作对返回值没有效果)   
        11: areturn       
        12: astore_2 

        13: ldc           #3                  // String zhuang111111
        15: astore_0      
        16: aload_2       
        17: athrow        
      Exception table:
         from    to  target type
             2     7    12   any

         (finally 的另一个作用是当抛异常时，finally 中的代码块还依然执行，所以在exception table 中，又定义了一个 any 的异常处理如果抛异常了，程序也依然执行target 下的代码，最后返回的值也是不变的  )
      LineNumberTable:
        line 26: 0
        line 28: 2
        line 29: 5
        line 31: 7
        line 29: 10
        line 31: 12
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
               2      16     0  name   Ljava/lang/String;
      StackMapTable: number_of_entries = 1
           frame_type = 255 /* full_frame */
          offset_delta = 12
          locals = [ class java/lang/String ]
          stack = [ class java/lang/Throwable ]



```

#### 结论：
finally 的代码块编译后都会接到 try 代码块之后

1.如果 try 代码块中return ,就return 了
2.如果 try 代码块后还有代码继续执行，则会出现 goto 指令，跳转到下段指令
然后在 exception table 中注册了 any 异常
如果在 try 内抛了异常，就会去异常表找到 any 然后，跳转到对应的 target 代码段继续执行


### 测试的Java 代码

```

/**
 * Created by zhuangjiesen on 2017/12/9.
 */
public class TryFinallyTest {

    public static void main(String[] args) {
        try {
            String str = "zhuang";
        } finally {
            String name = "zhuang111111";
        }

        System.out.println("doTryFinally1 : " + doTryFinally1());
        System.out.println("doTryFinally2 : " + doTryFinally2());
        System.out.println("doTryFinally3 : " + doTryFinally3());
        System.out.println("doTryFinally4 : " + doTryFinally4());
    }

    public static String doTryFinally1 (){
        String name = null;
        try {
            name= "zhuang";
            return name;
        } finally {
            name = "zhuang111111";
        }
    }

    public static String doTryFinally2 (){
        String name = null;
        try {
            name= "zhuang";
            return name;
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            name = "zhuang111111";
            return name;
        }
    }

    public static String doTryFinally3 (){
        String name = null;
        try {
            name= "zhuang";
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            name = "zhuang111111";
            return name;
        }
    }

    public static int doTryFinally4 (){
        int a = 0;
        try {
            a = 10;
            return a;
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            a++;
            return a;
        }
    }

    public static String doTryFinally5 (){
        String name = null;
        try {
            name= "zhuang";
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            name = "zhuang111111";
        }
        return name;
    }
    public static String doTryFinally6 (){
        String name = null;
        try {
            name= "zhuang";
        } catch (Exception e) {
            name= "444444";
            return name;
        } finally {
            name = "zhuang111111";
        }
        return name;
    }

}
```

### 字节码 (javap -verbose TryFinallyTest)

```

Classfile /C:/Users/zhuangjiesen/IdeaProjects/HelloWorld/target/classes/com/java/TryFinallyTest.class
  Last modified 2017-12-9; size 2387 bytes
  MD5 checksum 1d97c2b783088a05dc7a89a3066f1fcc
  Compiled from "TryFinallyTest.java"
public class com.java.TryFinallyTest
  SourceFile: "TryFinallyTest.java"
  minor version: 0
  major version: 52
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Methodref          #23.#55        //  java/lang/Object."<init>":()V
   #2 = String             #56            //  zhuang
   #3 = String             #57            //  zhuang111111
   #4 = Fieldref           #58.#59        //  java/lang/System.out:Ljava/io/PrintStream;
   #5 = Class              #60            //  java/lang/StringBuilder
   #6 = Methodref          #5.#55         //  java/lang/StringBuilder."<init>":()V
   #7 = String             #61            //  doTryFinally1 : 
   #8 = Methodref          #5.#62         //  java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
   #9 = Methodref          #22.#63        //  com/java/TryFinallyTest.doTryFinally1:()Ljava/lang/String;
  #10 = Methodref          #5.#64         //  java/lang/StringBuilder.toString:()Ljava/lang/String;
  #11 = Methodref          #65.#66        //  java/io/PrintStream.println:(Ljava/lang/String;)V
  #12 = String             #67            //  doTryFinally2 : 
  #13 = Methodref          #22.#68        //  com/java/TryFinallyTest.doTryFinally2:()Ljava/lang/String;
  #14 = String             #69            //  doTryFinally3 : 
  #15 = Methodref          #22.#70        //  com/java/TryFinallyTest.doTryFinally3:()Ljava/lang/String;
  #16 = String             #71            //  doTryFinally4 : 
  #17 = Methodref          #22.#72        //  com/java/TryFinallyTest.doTryFinally4:()I
  #18 = Methodref          #5.#73         //  java/lang/StringBuilder.append:(I)Ljava/lang/StringBuilder;
  #19 = Class              #74            //  java/lang/Exception
  #20 = Methodref          #19.#75        //  java/lang/Exception.printStackTrace:()V
  #21 = String             #76            //  444444
  #22 = Class              #77            //  com/java/TryFinallyTest
  #23 = Class              #78            //  java/lang/Object
  #24 = Utf8               <init>
  #25 = Utf8               ()V
  #26 = Utf8               Code
  #27 = Utf8               LineNumberTable
  #28 = Utf8               LocalVariableTable
  #29 = Utf8               this
  #30 = Utf8               Lcom/java/TryFinallyTest;
  #31 = Utf8               main
  #32 = Utf8               ([Ljava/lang/String;)V
  #33 = Utf8               args
  #34 = Utf8               [Ljava/lang/String;
  #35 = Utf8               StackMapTable
  #36 = Class              #79            //  java/lang/Throwable
  #37 = Utf8               doTryFinally1
  #38 = Utf8               ()Ljava/lang/String;
  #39 = Utf8               name
  #40 = Utf8               Ljava/lang/String;
  #41 = Class              #80            //  java/lang/String
  #42 = Utf8               doTryFinally2
  #43 = Utf8               e
  #44 = Utf8               Ljava/lang/Exception;
  #45 = Class              #74            //  java/lang/Exception
  #46 = Utf8               doTryFinally3
  #47 = Utf8               doTryFinally4
  #48 = Utf8               ()I
  #49 = Utf8               a
  #50 = Utf8               I
  #51 = Utf8               doTryFinally5
  #52 = Utf8               doTryFinally6
  #53 = Utf8               SourceFile
  #54 = Utf8               TryFinallyTest.java
  #55 = NameAndType        #24:#25        //  "<init>":()V
  #56 = Utf8               zhuang
  #57 = Utf8               zhuang111111
  #58 = Class              #81            //  java/lang/System
  #59 = NameAndType        #82:#83        //  out:Ljava/io/PrintStream;
  #60 = Utf8               java/lang/StringBuilder
  #61 = Utf8               doTryFinally1 : 
  #62 = NameAndType        #84:#85        //  append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
  #63 = NameAndType        #37:#38        //  doTryFinally1:()Ljava/lang/String;
  #64 = NameAndType        #86:#38        //  toString:()Ljava/lang/String;
  #65 = Class              #87            //  java/io/PrintStream
  #66 = NameAndType        #88:#89        //  println:(Ljava/lang/String;)V
  #67 = Utf8               doTryFinally2 : 
  #68 = NameAndType        #42:#38        //  doTryFinally2:()Ljava/lang/String;
  #69 = Utf8               doTryFinally3 : 
  #70 = NameAndType        #46:#38        //  doTryFinally3:()Ljava/lang/String;
  #71 = Utf8               doTryFinally4 : 
  #72 = NameAndType        #47:#48        //  doTryFinally4:()I
  #73 = NameAndType        #84:#90        //  append:(I)Ljava/lang/StringBuilder;
  #74 = Utf8               java/lang/Exception
  #75 = NameAndType        #91:#25        //  printStackTrace:()V
  #76 = Utf8               444444
  #77 = Utf8               com/java/TryFinallyTest
  #78 = Utf8               java/lang/Object
  #79 = Utf8               java/lang/Throwable
  #80 = Utf8               java/lang/String
  #81 = Utf8               java/lang/System
  #82 = Utf8               out
  #83 = Utf8               Ljava/io/PrintStream;
  #84 = Utf8               append
  #85 = Utf8               (Ljava/lang/String;)Ljava/lang/StringBuilder;
  #86 = Utf8               toString
  #87 = Utf8               java/io/PrintStream
  #88 = Utf8               println
  #89 = Utf8               (Ljava/lang/String;)V
  #90 = Utf8               (I)Ljava/lang/StringBuilder;
  #91 = Utf8               printStackTrace
{
  public com.java.TryFinallyTest();
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0       
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return        
      LineNumberTable:
        line 6: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
               0       5     0  this   Lcom/java/TryFinallyTest;

  public static void main(java.lang.String[]);
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=3, locals=4, args_size=1
         0: ldc           #2                  // String zhuang
         2: astore_1      
         3: ldc           #3                  // String zhuang111111
         5: astore_1      
         6: goto          15
         9: astore_2      
        10: ldc           #3                  // String zhuang111111
        12: astore_3      
        13: aload_2       
        14: athrow        
        15: getstatic     #4                  // Field java/lang/System.out:Ljava/io/PrintStream;
        18: new           #5                  // class java/lang/StringBuilder
        21: dup           
        22: invokespecial #6                  // Method java/lang/StringBuilder."<init>":()V
        25: ldc           #7                  // String doTryFinally1 : 
        27: invokevirtual #8                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
        30: invokestatic  #9                  // Method doTryFinally1:()Ljava/lang/String;
        33: invokevirtual #8                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
        36: invokevirtual #10                 // Method java/lang/StringBuilder.toString:()Ljava/lang/String;
        39: invokevirtual #11                 // Method java/io/PrintStream.println:(Ljava/lang/String;)V
        42: getstatic     #4                  // Field java/lang/System.out:Ljava/io/PrintStream;
        45: new           #5                  // class java/lang/StringBuilder
        48: dup           
        49: invokespecial #6                  // Method java/lang/StringBuilder."<init>":()V
        52: ldc           #12                 // String doTryFinally2 : 
        54: invokevirtual #8                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
        57: invokestatic  #13                 // Method doTryFinally2:()Ljava/lang/String;
        60: invokevirtual #8                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
        63: invokevirtual #10                 // Method java/lang/StringBuilder.toString:()Ljava/lang/String;
        66: invokevirtual #11                 // Method java/io/PrintStream.println:(Ljava/lang/String;)V
        69: getstatic     #4                  // Field java/lang/System.out:Ljava/io/PrintStream;
        72: new           #5                  // class java/lang/StringBuilder
        75: dup           
        76: invokespecial #6                  // Method java/lang/StringBuilder."<init>":()V
        79: ldc           #14                 // String doTryFinally3 : 
        81: invokevirtual #8                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
        84: invokestatic  #15                 // Method doTryFinally3:()Ljava/lang/String;
        87: invokevirtual #8                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
        90: invokevirtual #10                 // Method java/lang/StringBuilder.toString:()Ljava/lang/String;
        93: invokevirtual #11                 // Method java/io/PrintStream.println:(Ljava/lang/String;)V
        96: getstatic     #4                  // Field java/lang/System.out:Ljava/io/PrintStream;
        99: new           #5                  // class java/lang/StringBuilder
       102: dup           
       103: invokespecial #6                  // Method java/lang/StringBuilder."<init>":()V
       106: ldc           #16                 // String doTryFinally4 : 
       108: invokevirtual #8                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
       111: invokestatic  #17                 // Method doTryFinally4:()I
       114: invokevirtual #18                 // Method java/lang/StringBuilder.append:(I)Ljava/lang/StringBuilder;
       117: invokevirtual #10                 // Method java/lang/StringBuilder.toString:()Ljava/lang/String;
       120: invokevirtual #11                 // Method java/io/PrintStream.println:(Ljava/lang/String;)V
       123: return        
      Exception table:
         from    to  target type
             0     3     9   any
      LineNumberTable:
        line 11: 0
        line 13: 3
        line 14: 6
        line 13: 9
        line 14: 13
        line 17: 15
        line 18: 42
        line 19: 69
        line 20: 96
        line 21: 123
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
               0     124     0  args   [Ljava/lang/String;
      StackMapTable: number_of_entries = 2
           frame_type = 73 /* same_locals_1_stack_item */
          stack = [ class java/lang/Throwable ]
           frame_type = 5 /* same */


  public static java.lang.String doTryFinally1();
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=1, locals=3, args_size=0
         0: aconst_null   
         1: astore_0      
         2: ldc           #2                  // String zhuang
         4: astore_0      
         5: aload_0       
         6: astore_1      
         7: ldc           #3                  // String zhuang111111
         9: astore_0      
        10: aload_1       
        11: areturn       
        12: astore_2      
        13: ldc           #3                  // String zhuang111111
        15: astore_0      
        16: aload_2       
        17: athrow        
      Exception table:
         from    to  target type
             2     7    12   any
      LineNumberTable:
        line 26: 0
        line 28: 2
        line 29: 5
        line 31: 7
        line 29: 10
        line 31: 12
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
               2      16     0  name   Ljava/lang/String;
      StackMapTable: number_of_entries = 1
           frame_type = 255 /* full_frame */
          offset_delta = 12
          locals = [ class java/lang/String ]
          stack = [ class java/lang/Throwable ]


  public static java.lang.String doTryFinally2();
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=1, locals=3, args_size=0
         0: aconst_null   
         1: astore_0      
         2: ldc           #2                  // String zhuang
         4: astore_0      
         5: aload_0       
         6: astore_1      
         7: ldc           #3                  // String zhuang111111
         9: astore_0      
        10: aload_0       
        11: areturn       
        12: astore_1      
        13: aload_1       
        14: invokevirtual #20                 // Method java/lang/Exception.printStackTrace:()V
        17: ldc           #3                  // String zhuang111111
        19: astore_0      
        20: aload_0       
        21: areturn       
        22: astore_2      
        23: ldc           #3                  // String zhuang111111
        25: astore_0      
        26: aload_0       
        27: areturn       
      Exception table:
         from    to  target type
             2     7    12   Class java/lang/Exception
             2     7    22   any
            12    17    22   any
      LineNumberTable:
        line 39: 0
        line 41: 2
        line 42: 5
        line 46: 7
        line 47: 10
        line 43: 12
        line 44: 13
        line 46: 17
        line 47: 20
        line 46: 22
        line 47: 26
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
              13       4     1     e   Ljava/lang/Exception;
               2      26     0  name   Ljava/lang/String;
      StackMapTable: number_of_entries = 2
           frame_type = 255 /* full_frame */
          offset_delta = 12
          locals = [ class java/lang/String ]
          stack = [ class java/lang/Exception ]
           frame_type = 73 /* same_locals_1_stack_item */
          stack = [ class java/lang/Throwable ]


  public static java.lang.String doTryFinally3();
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=1, locals=3, args_size=0
         0: aconst_null   
         1: astore_0      
         2: ldc           #2                  // String zhuang
         4: astore_0      
         5: ldc           #3                  // String zhuang111111
         7: astore_0      
         8: aload_0       
         9: areturn       
        10: astore_1      
        11: aload_1       
        12: invokevirtual #20                 // Method java/lang/Exception.printStackTrace:()V
        15: ldc           #3                  // String zhuang111111
        17: astore_0      
        18: aload_0       
        19: areturn       
        20: astore_2      
        21: ldc           #3                  // String zhuang111111
        23: astore_0      
        24: aload_0       
        25: areturn       
      Exception table:
         from    to  target type
             2     5    10   Class java/lang/Exception
             2     5    20   any
            10    15    20   any
      LineNumberTable:
        line 56: 0
        line 58: 2
        line 62: 5
        line 63: 8
        line 59: 10
        line 60: 11
        line 62: 15
        line 63: 18
        line 62: 20
        line 63: 24
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
              11       4     1     e   Ljava/lang/Exception;
               2      24     0  name   Ljava/lang/String;
      StackMapTable: number_of_entries = 2
           frame_type = 255 /* full_frame */
          offset_delta = 10
          locals = [ class java/lang/String ]
          stack = [ class java/lang/Exception ]
           frame_type = 73 /* same_locals_1_stack_item */
          stack = [ class java/lang/Throwable ]


  public static int doTryFinally4();
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=1, locals=3, args_size=0
         0: iconst_0      
         1: istore_0      
         2: bipush        10
         4: istore_0      
         5: iload_0       
         6: istore_1      
         7: iinc          0, 1
        10: iload_0       
        11: ireturn       
        12: astore_1      
        13: aload_1       
        14: invokevirtual #20                 // Method java/lang/Exception.printStackTrace:()V
        17: iinc          0, 1
        20: iload_0       
        21: ireturn       
        22: astore_2      
        23: iinc          0, 1
        26: iload_0       
        27: ireturn       
      Exception table:
         from    to  target type
             2     7    12   Class java/lang/Exception
             2     7    22   any
            12    17    22   any
      LineNumberTable:
        line 70: 0
        line 72: 2
        line 73: 5
        line 77: 7
        line 78: 10
        line 74: 12
        line 75: 13
        line 77: 17
        line 78: 20
        line 77: 22
        line 78: 26
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
              13       4     1     e   Ljava/lang/Exception;
               2      26     0     a   I
      StackMapTable: number_of_entries = 2
           frame_type = 255 /* full_frame */
          offset_delta = 12
          locals = [ int ]
          stack = [ class java/lang/Exception ]
           frame_type = 73 /* same_locals_1_stack_item */
          stack = [ class java/lang/Throwable ]


  public static java.lang.String doTryFinally5();
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=1, locals=3, args_size=0
         0: aconst_null   
         1: astore_0      
         2: ldc           #2                  // String zhuang
         4: astore_0      
         5: ldc           #3                  // String zhuang111111
         7: astore_0      
         8: goto          28
        11: astore_1      
        12: aload_1       
        13: invokevirtual #20                 // Method java/lang/Exception.printStackTrace:()V
        16: ldc           #3                  // String zhuang111111
        18: astore_0      
        19: goto          28
        22: astore_2      
        23: ldc           #3                  // String zhuang111111
        25: astore_0      
        26: aload_2       
        27: athrow        
        28: aload_0       
        29: areturn       
      Exception table:
         from    to  target type
             2     5    11   Class java/lang/Exception
             2     5    22   any
            11    16    22   any
      LineNumberTable:
        line 85: 0
        line 87: 2
        line 91: 5
        line 92: 8
        line 88: 11
        line 89: 12
        line 91: 16
        line 92: 19
        line 91: 22
        line 93: 28
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
              12       4     1     e   Ljava/lang/Exception;
               2      28     0  name   Ljava/lang/String;
      StackMapTable: number_of_entries = 3
           frame_type = 255 /* full_frame */
          offset_delta = 11
          locals = [ class java/lang/String ]
          stack = [ class java/lang/Exception ]
           frame_type = 74 /* same_locals_1_stack_item */
          stack = [ class java/lang/Throwable ]
           frame_type = 5 /* same */


  public static java.lang.String doTryFinally6();
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=1, locals=4, args_size=0
         0: aconst_null   
         1: astore_0      
         2: ldc           #2                  // String zhuang
         4: astore_0      
         5: ldc           #3                  // String zhuang111111
         7: astore_0      
         8: goto          28
        11: astore_1      
        12: ldc           #21                 // String 444444
        14: astore_0      
        15: aload_0       
        16: astore_2      
        17: ldc           #3                  // String zhuang111111
        19: astore_0      
        20: aload_2       
        21: areturn       
        22: astore_3      
        23: ldc           #3                  // String zhuang111111
        25: astore_0      
        26: aload_3       
        27: athrow        
        28: aload_0       
        29: areturn       
      Exception table:
         from    to  target type
             2     5    11   Class java/lang/Exception
             2     5    22   any
            11    17    22   any
      LineNumberTable:
        line 98: 0
        line 100: 2
        line 105: 5
        line 106: 8
        line 101: 11
        line 102: 12
        line 103: 15
        line 105: 17
        line 103: 20
        line 105: 22
        line 107: 28
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
              12      10     1     e   Ljava/lang/Exception;
               2      28     0  name   Ljava/lang/String;
      StackMapTable: number_of_entries = 3
           frame_type = 255 /* full_frame */
          offset_delta = 11
          locals = [ class java/lang/String ]
          stack = [ class java/lang/Exception ]
           frame_type = 74 /* same_locals_1_stack_item */
          stack = [ class java/lang/Throwable ]
           frame_type = 5 /* same */

}

```
