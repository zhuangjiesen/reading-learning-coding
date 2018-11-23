# spring MVC获取请求参数方法，及java 中获取方法中参数的名字

#### 在看spring mvc 中一直纳闷spring mvc 中如何根据方法参数去映射，从HttpServletRequest 获取请求参数的

```
	/** 
		Description: 跳转首页  <p>
	    @author : zhuangjiesen@qq.com 庄杰森 2016年4月19日
		 */
	@RequestMapping("/common/index.do")
	public String toIndex(HttpServletRequest request,HttpServletResponse response ,
						  String name , String content,
						  ModelAndView model){
						  
    /* 如  request 、 response  、name 、content 这些参数源码中是具体如何获取的 */
}
```


#### spring mvc 获取参数的源码位置

##### InvocableHandlerMethod.class 的 getMethodArgumentValues() 方法
```
package org.springframework.web.method.support;


public class InvocableHandlerMethod extends HandlerMethod {
    ...
        private ParameterNameDiscoverer parameterNameDiscoverer = new LocalVariableTableParameterNameDiscoverer();

    
    private Object[] getMethodArgumentValues(NativeWebRequest request, ModelAndViewContainer mavContainer, Object... providedArgs) throws Exception {
        MethodParameter[] parameters = this.getMethodParameters();
        Object[] args = new Object[parameters.length];

        for(int i = 0; i < parameters.length; ++i) {
            MethodParameter parameter = parameters[i];
            parameter.initParameterNameDiscovery(this.parameterNameDiscoverer);
            GenericTypeResolver.resolveParameterType(parameter, this.getBean().getClass());
            args[i] = this.resolveProvidedArgument(parameter, providedArgs);
            if(args[i] == null) {
                if(this.argumentResolvers.supportsParameter(parameter)) {
                    try {
                        args[i] = this.argumentResolvers.resolveArgument(parameter, mavContainer, request, this.dataBinderFactory);
                    } catch (Exception var9) {
                        if(this.logger.isTraceEnabled()) {
                            this.logger.trace(this.getArgumentResolutionErrorMessage("Error resolving argument", i), var9);
                        }

                        throw var9;
                    }
                } else if(args[i] == null) {
                    String msg = this.getArgumentResolutionErrorMessage("No suitable resolver for argument", i);
                    throw new IllegalStateException(msg);
                }
            }
        }

        return args;
    }

}
```

##### 从中可以看出 initParameterNameDiscovery() 方法初始化了参数名称获取的接口类；而重点就是这个 LocalVariableTableParameterNameDiscoverer.class类


### LocalVariableTableParameterNameDiscoverer.class 解析

```
有人说jdk8 中已经可以获取参数名称

public class HelloWorld {
	
	public static void main(String[] args) throws NoSuchMethodException, SecurityException {
		// TODO Auto-generated method stub
		Method method = Man.class.getMethod("doMethod", new Class[]{String.class , String.class });
		Parameter[] parameters = method.getParameters();
		if (parameters != null) {
			for (Parameter parameter : parameters) {
				System.out.println("parameter : "+parameter.getName());
			}
			
		}
	}
	
}

我试了一下，方法获取到的参数名输出：
parameter : arg0
parameter : arg1

// Man 类
public class Man {
	public void doMethod(String arg1My , String arg2My ) {
	}
}
所以反射方法得出的方法参数是不行的

```

##### LocalVariableTableParameterNameDiscoverer 类中是用 ASM 字节码操纵框架读取字节码文件获取真实在 java 代码中定义的真实参数名

##### ASM是一个java字节码操纵框架，它能被用来动态生成类或者增强既有类的功能。ASM 可以直接产生二进制 class 文件，也可以在类被加载入 Java 虚拟机之前动态改变类行为。Java class 被存储在严格格式定义的 .class文件里，这些类文件拥有足够的元数据来解析类中的所有元素：类名称、方法、属性以及 Java 字节码（指令）。ASM从类文件中读入信息后，能够改变类行为，分析类信息，甚至能够根据用户要求生成新类。

    
Spring 中通过 ASM字节码操纵框架获取类方法参数名 LocalVariableTableParameterNameDiscoverer

### 内容不复杂，主要在于帮助自己理解字节码组成，理解 静态变量表和常量池，<clinit> (静态方法块，与静态属性), <init> (构造方法)  这些在 class 文件中的展示

### LocalVariableTableParameterNameDiscoverer类中定义了两个静态内部类用于读取 class 文件中的 method 与 其方法下的 LocalVariableTable(本地变量表) 的数据

#### AsmClassTest.class (用来测试读取方法参数名的测试类)

java源码：

```

package com.java.core.asm;

/**
 * Created by zhuangjiesen on 2017/5/17.
 */
public class AsmClassTest {

    public AsmClassTest(){
    }

    public AsmClassTest(long constructorArgLong , String constructorArg){
        String localConArg1 = "局部变量1";
        String localConArg2 = "局部变量2";
    }

    /*
    * 没有参数，没有返回值的方法
    * */
    public void doMethodFirst(){
    }

    /*
    * 有参数，没有返回值的方法
    * */
    public void doMethodSecond(long methodArgLong , String methodArgStr1 , String methodArgStr2){
    }


    /*
    * 有参数，有返回值的方法
    * */
    public String doMethodThird(long methodArgLong , String methodArgStr1 , String methodArgStr2){
        String loacalRetValue = "本地变量1";
        String loacalRetValue2 = "本地变量2";
        return loacalRetValue;
    }

    /*
    * 有参数，有返回值的方法
    * */
    public String doMethodForth(long methodArgLong , String methodArgStr1 , String methodArgStr2 , long methodArgLong3 ){
        String loacalRetValue = "本地变量1";
        String loacalRetValue2 = "本地变量2";
        String loacalRetValue3 = "本地变量2";
        return loacalRetValue;
    }
    public static String doStaticMethodFifth(long methodArgLong , String methodArgStr1 , String methodArgStr2 , long methodArgLong3 ){
        String loacalRetValue = "本地变量1";
        String loacalRetValue2 = "本地变量2";
        String loacalRetValue3 = "本地变量2";
        return loacalRetValue;
    }
}

```

#### 通过命令查看class文件的字节码内容：

```
javap -verbose xxxxx.class

//测试
javap -verbose AsmClassTest.class
```

#### 输出结果： （部分地方已经用括号给了注释）

```

public class com.java.core.asm.AsmClassTest
  minor version: 0
  major version: 49
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Methodref          #7.#39         // java/lang/Object."<init>":()V
   #2 = String             #40            // 局部变量1
   #3 = String             #41            // 局部变量2
   #4 = String             #42            // 本地变量1
   #5 = String             #43            // 本地变量2
   #6 = Class              #44            // com/java/core/asm/AsmClassTest
   #7 = Class              #45            // java/lang/Object
   #8 = Utf8               <init>
   #9 = Utf8               ()V
  #10 = Utf8               Code
  #11 = Utf8               LineNumberTable
  #12 = Utf8               LocalVariableTable
  #13 = Utf8               this
  #14 = Utf8               Lcom/java/core/asm/AsmClassTest;
  #15 = Utf8               (JLjava/lang/String;)V
  #16 = Utf8               constructorArgLong
  #17 = Utf8               J
  #18 = Utf8               constructorArg
  #19 = Utf8               Ljava/lang/String;
  #20 = Utf8               localConArg1
  #21 = Utf8               localConArg2
  #22 = Utf8               doMethodFirst
  #23 = Utf8               doMethodSecond
  #24 = Utf8               (JLjava/lang/String;Ljava/lang/String;)V
  #25 = Utf8               methodArgLong
  #26 = Utf8               methodArgStr1
  #27 = Utf8               methodArgStr2
  #28 = Utf8               doMethodThird
  #29 = Utf8               (JLjava/lang/String;Ljava/lang/String;)Ljava/lang/String;
  #30 = Utf8               loacalRetValue
  #31 = Utf8               loacalRetValue2
  #32 = Utf8               doMethodForth
  #33 = Utf8               (JLjava/lang/String;Ljava/lang/String;J)Ljava/lang/String;
  #34 = Utf8               methodArgLong3
  #35 = Utf8               loacalRetValue3
  #36 = Utf8               doStaticMethodFifth
  #37 = Utf8               SourceFile
  #38 = Utf8               AsmClassTest.java
  #39 = NameAndType        #8:#9          // "<init>":()V
  #40 = Utf8               局部变量1
  #41 = Utf8               局部变量2
  #42 = Utf8               本地变量1
  #43 = Utf8               本地变量2
  #44 = Utf8               com/java/core/asm/AsmClassTest
  #45 = Utf8               java/lang/Object
{
  public com.java.core.asm.AsmClassTest();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 8: 0
        line 9: 4
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       5     0  this   Lcom/java/core/asm/AsmClassTest;

  public com.java.core.asm.AsmClassTest(long, java.lang.String);
    descriptor: (JLjava/lang/String;)V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=6, args_size=3
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: ldc           #2                  // String 局部变量1
         6: astore        4
         8: ldc           #3                  // String 局部变量2
        10: astore        5
        12: return
      LineNumberTable:
        line 11: 0
        line 12: 4
        line 13: 8
        line 14: 12
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      13     0  this   Lcom/java/core/asm/AsmClassTest;
            0      13     1 constructorArgLong   J
            0      13     3 constructorArg   Ljava/lang/String;
            8       5     4 localConArg1   Ljava/lang/String;
           12       1     5 localConArg2   Ljava/lang/String;

  public void doMethodFirst();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=0, locals=1, args_size=1
         0: return
      LineNumberTable:
        line 20: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       1     0  this   Lcom/java/core/asm/AsmClassTest;

  public void doMethodSecond(long, java.lang.String, java.lang.String);
    descriptor: (JLjava/lang/String;Ljava/lang/String;)V
    flags: ACC_PUBLIC
    Code:
      stack=0, locals=5, args_size=4
         0: return
      LineNumberTable:
        line 26: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       1     0  this   Lcom/java/core/asm/AsmClassTest;
            0       1     1 methodArgLong   J
            0       1     3 methodArgStr1   Ljava/lang/String;
            0       1     4 methodArgStr2   Ljava/lang/String;

  public java.lang.String doMethodThird(long, java.lang.String, java.lang.String);
    descriptor: (JLjava/lang/String;Ljava/lang/String;)Ljava/lang/String;
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=7, args_size=4
         0: ldc           #4                  // String 本地变量1
         2: astore        5
         4: ldc           #5                  // String 本地变量2
         6: astore        6
         8: aload         5
        10: areturn
      LineNumberTable:
        line 33: 0
        line 34: 4
        line 35: 8
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      11     0  this   Lcom/java/core/asm/AsmClassTest;
            0      11     1 methodArgLong   J
            0      11     3 methodArgStr1   Ljava/lang/String;
            0      11     4 methodArgStr2   Ljava/lang/String;
            4       7     5 loacalRetValue   Ljava/lang/String;
            8       3     6 loacalRetValue2   Ljava/lang/String;

  public java.lang.String doMethodForth(long, java.lang.String, java.lang.String, long);
    descriptor: (JLjava/lang/String;Ljava/lang/String;J)Ljava/lang/String;
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=10, args_size=5
         0: ldc           #4                  // String 本地变量1
         2: astore        7
         4: ldc           #5                  // String 本地变量2
         6: astore        8
         8: ldc           #5                  // String 本地变量2
        10: astore        9
        12: aload         7
        14: areturn
      LineNumberTable:
        line 42: 0
        line 43: 4
        line 44: 8
        line 45: 12
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      15     0  this   Lcom/java/core/asm/AsmClassTest;
            0      15     1 methodArgLong   J
            0      15     3 methodArgStr1   Ljava/lang/String;
            0      15     4 methodArgStr2   Ljava/lang/String;
            0      15     5 methodArgLong3   J
            4      11     7 loacalRetValue   Ljava/lang/String;
            8       7     8 loacalRetValue2   Ljava/lang/String;
           12       3     9 loacalRetValue3   Ljava/lang/String;

  public static java.lang.String doStaticMethodFifth(long, java.lang.String, java.lang.String, long);
    descriptor: (JLjava/lang/String;Ljava/lang/String;J)Ljava/lang/String;
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=1, locals=9, args_size=4
         0: ldc           #4                  // String 本地变量1
         2: astore        6
         4: ldc           #5                  // String 本地变量2
         6: astore        7
         8: ldc           #5                  // String 本地变量2
        10: astore        8
        12: aload         6
        14: areturn
      LineNumberTable:
        line 50: 0
        line 51: 4
        line 52: 8
        line 53: 12
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      15     0 methodArgLong   J
            0      15     2 methodArgStr1   Ljava/lang/String;
            0      15     3 methodArgStr2   Ljava/lang/String;
            0      15     4 methodArgLong3   J
            4      11     6 loacalRetValue   Ljava/lang/String;
            8       7     7 loacalRetValue2   Ljava/lang/String;
           12       3     8 loacalRetValue3   Ljava/lang/String;
}


```


#### 通过分析字节码文件，观察如果需要获取参数名称，需要从 Constant pool  或者方法下的 LocalVariableTable 

args_size 表示参数数量：如果是实例方法，真正自己定义的参数数量是 args_size - 1  ,静态方法的参数数量就等于 args_size


### 分析 LineNumberTable
Java源码中与字节码指令的对应
第一个 linx xx: 0 指的是方法命名的地方 


### 分析 LocalVariableTable 



#### 静态方法中没有 this 的局部变量，所以静态方法中没有 this 变量
#### 实例方法的本地变量表中默认第一个添加了 this 变量

#### 这也解决了我对于this 引用的疑惑

Start 为 0 的除了 this 就是方法的参数变量了
slot  则代表参数占局部变量表数组的位置，当参数为基本类型 long 或者 double 时，slot 占用了两个槽位


这样一分析，对于字节码解析方法获取方法参数名就有个大概思路

### LocalVariableTableParameterNameDiscoverer 类解读

入口：

```
public String[] getParameterNames(Method method) 方法
先获取了缓存存储的类信息 ， 获取不到就直接解析class类信息

```

详细代码：

```

	public String[] getParameterNames(Method method) {
		Method originalMethod = BridgeMethodResolver.findBridgedMethod(method);
		Class<?> declaringClass = originalMethod.getDeclaringClass();
		Map<Member, String[]> map = this.parameterNamesCache.get(declaringClass);
		if (map == null) {

			// 使用 asm 框架解析 class 
			map = inspectClass(declaringClass);
			this.parameterNamesCache.put(declaringClass, map);
		}
		if (map != NO_DEBUG_INFO_MAP) {
			return map.get(originalMethod);
		}
		return null;
	}
```



##### inspectClass() 方法代码：

```
	/**
	 * Inspects the target class. Exceptions will be logged and a maker map returned
	 * to indicate the lack of debug information.
	 */
	private Map<Member, String[]> inspectClass(Class<?> clazz) {
		InputStream is = clazz.getResourceAsStream(ClassUtils.getClassFileName(clazz));
		if (is == null) {
			// We couldn't load the class file, which is not fatal as it
			// simply means this method of discovering parameter names won't work.
			if (logger.isDebugEnabled()) {
				logger.debug("Cannot find '.class' file for class [" + clazz
						+ "] - unable to determine constructors/methods parameter names");
			}
			return NO_DEBUG_INFO_MAP;
		}
		try {

			// 重点是 ClassReader 类 accept() 方法解析

			ClassReader classReader = new ClassReader(is);
			Map<Member, String[]> map = new ConcurrentHashMap<Member, String[]>();
			classReader.accept(new ParameterNameDiscoveringVisitor(clazz, map), false);
			return map;
		}
		catch (IOException ex) {
			if (logger.isDebugEnabled()) {
				logger.debug("Exception thrown while reading '.class' file for class [" + clazz
						+ "] - unable to determine constructors/methods parameter names", ex);
			}
		}
		finally {
			try {
				is.close();
			}
			catch (IOException ex) {
				// ignore
			}
		}
		return NO_DEBUG_INFO_MAP;
	}

```

##### 这里就可以知道了，代码中是通过 ClassReader 类的 accept() 方法解析类的而
#### asm 框架，解析类需要两个接口：
* ClassVisitor 
* MethodVisitor 

#### 读取方法及本地变量表的方法是：

```
ClassVisitor 接口的 visitMethod(int access, String name, String desc, String signature, String[] exceptions) 方法 ;

MethodVisitor 接口的 visitLocalVariable(String name, String description, String signature, Label start, Label end,
				int index) 方法;
```

#### LocalVariableTableParameterNameDiscoverer 类的两个静态内部类：
* ParameterNameDiscoveringVisitor 
* LocalVariableTableVisitor 


##### ParameterNameDiscoveringVisitor 代码： 重点 visitMethod() 方法


```

	/**
	 * Helper class that inspects all methods (constructor included) and then
	 * attempts to find the parameter names for that member.
	 */
	private static class ParameterNameDiscoveringVisitor extends EmptyVisitor {


		// 静态属性和静态代码块初始化方法
		private static final String STATIC_CLASS_INIT = "<clinit>";

		private final Class<?> clazz;
		private final Map<Member, String[]> memberMap;

		public ParameterNameDiscoveringVisitor(Class<?> clazz, Map<Member, String[]> memberMap) {
			this.clazz = clazz;
			this.memberMap = memberMap;
		}


		@Override
		public MethodVisitor visitMethod(int access, String name, String desc, String signature, String[] exceptions) {
			// exclude synthetic + bridged && static class initialization
			// 过滤静态属性初始化方法和 过滤 bridge 方法 synthetic 方法（编译器生成的方法，不存在源代码中）
			if (!isSyntheticOrBridged(access) && !STATIC_CLASS_INIT.equals(name)) {
				// 过滤后进入局部变量表读取  LocalVariableTableVisitor 的 MethodVisitor
				return new LocalVariableTableVisitor(clazz, memberMap, name, desc, isStatic(access));
			}
			return null;
		}

		private static boolean isSyntheticOrBridged(int access) {
			return (((access & Opcodes.ACC_SYNTHETIC) | (access & Opcodes.ACC_BRIDGE)) > 0);
		}

		private static boolean isStatic(int access) {
			return ((access & Opcodes.ACC_STATIC) > 0);
		}
	}

```


##### LocalVariableTableVisitor 代码：重点 构造函数LocalVariableTableVisitor() 与 visitLocalVariable() 方法 （代码中有注释）


```
	private static class LocalVariableTableVisitor extends EmptyVisitor {

		private static final String CONSTRUCTOR = "<init>";

		private final Class<?> clazz;
		private final Map<Member, String[]> memberMap;
		private final String name;
		private final Type[] args;
		private final boolean isStatic;

		private String[] parameterNames;
		private boolean hasLvtInfo = false;

		/*
		 * The nth entry contains the slot index of the LVT table entry holding the
		 * argument name for the nth parameter.
		 */
		private final int[] lvtSlotIndex;

		public LocalVariableTableVisitor(Class<?> clazz, Map<Member, String[]> map, String name, String desc,
				boolean isStatic) {
			this.clazz = clazz;
			this.memberMap = map;
			this.name = name;
			// determine args
			/*
				通过解析这样的值 (JLjava/lang/String;Ljava/lang/String;J)Ljava/lang/String; 获取参数数量 
				如果是 静态方法 少了个 this 

			*/ 
			args = Type.getArgumentTypes(desc);
			this.parameterNames = new String[args.length];
			this.isStatic = isStatic;
			this.lvtSlotIndex = computeLvtSlotIndices(isStatic, args);
		}


		/*
			对比槽位值判断是否方法参数，并保存
		*/
		@Override
		public void visitLocalVariable(String name, String description, String signature, Label start, Label end,
				int index) {
			this.hasLvtInfo = true;
			for (int i = 0; i < lvtSlotIndex.length; i++) {
				if (lvtSlotIndex[i] == index) {
					this.parameterNames[i] = name;
				}
			}
		}


		/*
			解析完 本地变量表后将参数列表保存并返回
		*/
		@Override
		public void visitEnd() {
			if (this.hasLvtInfo || (this.isStatic && this.parameterNames.length == 0)) {
				// visitLocalVariable will never be called for static no args methods
				// which doesn't use any local variables.
				// This means that hasLvtInfo could be false for that kind of methods
				// even if the class has local variable info.
				memberMap.put(resolveMember(), parameterNames);
			}
		}

		private Member resolveMember() {
			ClassLoader loader = clazz.getClassLoader();
			Class<?>[] classes = new Class<?>[args.length];

			// resolve args
			for (int i = 0; i < args.length; i++) {
				classes[i] = ClassUtils.resolveClassName(args[i].getClassName(), loader);
			}
			try {
				if (CONSTRUCTOR.equals(name)) {
					return clazz.getDeclaredConstructor(classes);
				}

				return clazz.getDeclaredMethod(name, classes);
			} catch (NoSuchMethodException ex) {
				throw new IllegalStateException("Method [" + name
						+ "] was discovered in the .class file but cannot be resolved in the class object", ex);
			}
		}


		/*
			获取参数名称 将参数的 slot 值返回成数组 long double 类型 nextIndex +2 就是slot 槽位占两个位置
		*/
		private static int[] computeLvtSlotIndices(boolean isStatic, Type[] paramTypes) {
			int[] lvtIndex = new int[paramTypes.length];
			int nextIndex = (isStatic ? 0 : 1);
			for (int i = 0; i < paramTypes.length; i++) {
				lvtIndex[i] = nextIndex;
				if (isWideType(paramTypes[i])) {
					nextIndex += 2;
				} else {
					nextIndex++;
				}
			}
			return lvtIndex;
		}

		private static boolean isWideType(Type aType) {
			// float is not a wide type
			/*
			判断是否占用两个slot 的类型 一个是 long 一个是 double  ， 源码中还强调了 float 并没有占用两个槽位(slot)
			*/ 
			return (aType == Type.LONG_TYPE || aType == Type.DOUBLE_TYPE);
		}
	}

```

