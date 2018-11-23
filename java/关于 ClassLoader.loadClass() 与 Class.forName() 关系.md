# 关于 ClassLoader.loadClass() 与 Class.forName() 关系

### 我们都知道类加载器是
#### 通过类的权限定名来获取描述类的二进制字节流，所以loadClass()不能初始化类的，但是可以获取类的Class 对象，再对类进行初始化


### 不理解的问题
1. Class.forName() 加载的类与ClassLoader.loadClass() 加载的类的关系 ， 由于书上说是可以出现两个一样全限定名的类对象的


### 直接看代码

#### 条件：
1.两个 ClassLoaderTest.class 在静态方法以及 <clinit> 方法中定义了不一样的打印，用于区分。
2.自定义一个 自定义的class类加载器 （ myLoader ），加载的是不在classPath(编译器目录下)下的,class类（ClassLoaderTest.class）

3.分别用自定义类加载器初始化类，和Class.forName初始化类；


### ClassLoaderTest.java 代码：
1. 自定义类加载器要加载的类，编译完后将class文件放到电脑上的其他位置：

```
package org.dragsun;

/**
 * Created by zhuangjiesen on 2017/7/25.
 */
public class ClassLoaderTest {
    //这是自定义classLoader要加载的类
    static {

        System.out.println("i am new ClassLoaderTest....");
    }

    public static void showData(){
        System.out.println("jason...hello ...");

    }

}



```

2. Class.forName() 要加载的类：

```

package org.dragsun;

/**
 * Created by zhuangjiesen on 2017/7/25.
 */
public class ClassLoaderTest {

    //区分这是用class.forName加载的
    static {

        System.out.println("i am old.......111111 ClassLoaderTest....");
    }

    public static void showData(){
        System.out.println("jason...hello ...");

    }

}


```



### main方法，测试主体：

1.自定义的外部的类加载

```


    public static void main(String[] args) throws Exception {




        /*
        定义了自定义的类加载器，并打破父类委托机制，调用defineClass 自己加载类,所以加载类的加载器不是AppClassLoader
        *
        * */
        ClassLoader myLoader = new ClassLoader() {
            @Override
            public Class<?> loadClass(String name) throws ClassNotFoundException {
                try {
                    if (name.startsWith("org.dragsun")) {
                        File classLoaderTestClassFile = new File("/Users/zhuangjiesen/develop/java/ClassLoaderTest.class");
                        FileInputStream fins = new FileInputStream(classLoaderTestClassFile);
                        byte[] b = new byte[fins.available()];
                        fins.read(b);
                        return defineClass(name , b , 0 , b.length);
                    }
                } catch (Exception e) {
                    e.printStackTrace();
                }
                return super.loadClass(name);
            }
        };

        Class classLoaderTest = myLoader.loadClass("org.dragsun.ClassLoaderTest");
//通过实例化加载类        classLoaderTest.newInstance(); Class.forName("org.dragsun.ClassLoaderTest");

    }

```

#### 打印结果：
```
i am new ClassLoaderTest....
i am old.......111111 ClassLoaderTest....
```
#### 结论
说明两个类并不是同一个类
但是虚拟机可以存在两个全限定名一样的类，并且都可以进行实例化，各自运行；

#### 这时候我纳闷了，点开了Class.forName方法
```
   
    @CallerSensitive
    public static Class<?> forName(String className)
                throws ClassNotFoundException {
                /*
                发现这里的 forName0 方法中居然传了个classLoader 进来，说明class.forName方法也是需要一个类加载器的
                */
        Class<?> caller = Reflection.getCallerClass();
        return forName0(className, true, ClassLoader.getClassLoader(caller), caller);
    }

```
#### 于是我打个断点

```
发现caller 的类加载器就是Launcher$AppClassLoader 类(应用程序类加载器，系统默认加载器)
```

#### 所以 class.forName 方法是用系统(应用程序)类加载器去加载(并初始化)这个全限定名下的类

#### 接着我就在 Class 类中发现了另一个方法：
```
/*
name 是全限定名
initialize 是是否初始化，如果false 只会对类进行加载 ， 不触发<clinit> 方法

loader 也就是可以自定义类加载器
*/ 
    public static Class<?> forName(String name, boolean initialize,
                                   ClassLoader loader)
```

### 最后试了：


```

    public static void main(String[] args) throws Exception {
        /*
        定义了自定义的类加载器，并打破父类委托机制，调用defineClass 自己加载类,所以加载类的加载器不是AppClassLoader
        *
        * */
        ClassLoader myLoader = new ClassLoader() {
            @Override
            public Class<?> loadClass(String name) throws ClassNotFoundException {
                try {
                    if (name.startsWith("org.dragsun")) {
                        File classLoaderTestClassFile = new File("/Users/zhuangjiesen/develop/java/ClassLoaderTest.class");
                        FileInputStream fins = new FileInputStream(classLoaderTestClassFile);
                        byte[] b = new byte[fins.available()];
                        fins.read(b);
                        return defineClass(name , b , 0 , b.length);
                    }
                } catch (Exception e) {
                    e.printStackTrace();
                }
                return super.loadClass(name);
            }
        };

        Class classLoaderTest = myLoader.loadClass("org.dragsun.ClassLoaderTest");
                Class.forName("org.dragsun.ClassLoaderTest" , true , myLoader);
        Class.forName("org.dragsun.ClassLoaderTest");

    }


```

#### 打印结果：

```
i am new ClassLoaderTest....
i am old.......111111 ClassLoaderTest....

```

### 总结：
1. 不同的类加载器可以加载不同的类，并进行操作
2. Class.forName 默认了系统（应用程序） 类加载器(Laucher$AppClassLoader)
3. 被同一个虚拟机加载，只要他们的加载器不同，必定不会相同

4. 如果没有重写loadClass 且classPath 已经有一样全限定名的类，自定义类加载器只会调用父类加载器（应用程序类加载器） 加载类



