# 类加载
### 类的生命周期？类加载？初始化阶段中哪几种情况必须对类初始化
- 类的生命周期就是加载、验证、准备、解析、初始化、使用、卸载。不过解析可能是在初始化之前，或之后。在我们使用之前的这几个步骤都是类加载器去干的。类加载就是前面的五步：加载、验证、准备、解析、初始化，由类加载器做的类加载。
    - 首先加载是懒加载方式，用到的时候jvm才会去加载这个字节码文件，做的工作就比如给这个类一个唯一名字、在方法区给这个类一个访问入口、把字节码文件的结构转成运行时的结构
    - 验证就是验证字节码文件格式 
    - 准备就是给静态类变量分配内存且赋默认值。不是赋值，赋值是在初始化赋值的。
    - 解析就是把常量池的符号引用转为直接引用。
    - 初始化主要就是调用类的static代码块、对类的静态变量初始化为指定的值。初始化发生在：（1）如果父类还没初始化，就先调用父类的初始化。（2）new一个类（3）通过类名调用静态方法（4）通过类名拿到静态变量 （5）反射类。
    - 使用没啥好说的，比如对new出来的对象进行各种操作。
    - 卸载就得满足这个类已经没有任何活的实例对象、加载该类的类加载器已经被回收了，Class对象没有被引用且无法通过反射访问这个类。jvm自带的类加载器加载的类是不会被卸载的，但是由我们自定义的类加载器加载的类是可能被卸载的。

![image](https://user-images.githubusercontent.com/27798171/180156987-39ce0fa0-6051-4a0b-bf29-80adf473182c.png)

### 啥是符号引用？啥是直接引用？啥是静态链接？啥是动态链接？
- 比如一个main()方法，符号引用可以理解为字节码文件里面方法的引用，直接引用可以理解为指向内存的指针的或句柄。当在类加载期间，符号引用替换为直接引用是就是静态链接过程。
- 动态链接是在程序运行期间完成的将符号引用替换为直接引用，

### 类加载器和双亲委派机制
- 说下类加载器
    - 类加载器就是干类的加载、验证、准备、解析、初始化这些操作的
    - jdk提供的三个类加载器有引导类加载器(专门加载jre的lib下的核心类库)、它下面有个扩展类加载器(专门加载jre的lib的ext目录下的扩展类库)，扩展类加载器下面又有个应用类加载器(加载classpath下的类，我们自己写的类基本都是它来加载的)。
    - 当然你也可以自定义类加载器，就是自己去写个类去继承抽象类ClassLoader，你可以去重写classLoader方法，也可以不重写保留他原先的双亲委派机制。然后主要是得重写findClass方法来控制自定义类加载器扫描扫描的类路径。另外，除了引导类加载器以外，其他类加载器都得在parent的属性变量里面配置父类加载器，抽象类ClassLoader默认是父类加载器是应用类加载器。

```java
自定义类加载器

public class MyClassLoader extends ClassLoader {

    private String classPath;

    public AgentClassLoader(String classPath) {
        this.classPath = classPath;
    }

    private byte[] loadByte(String name) throws Exception {
        //省略，就是通过类名加载成byte[]
    }

    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        try {
            byte[] data = loadByte(name);
            //defineClass将一个字节数组转为Class对象，这个字节数组是class文件读取后最终的字节数组。
            return defineClass(name, data, 0, data.length);
        } catch (Exception e) {
            e.printStackTrace();
            throw new ClassNotFoundException();
        }
    }
}

    public static void main(String args[]) throws Exception {
        //初始化自定义类加载器，会先初始化父类ClassLoader，父类构造器会把自定义类加载器的父加载器设置为AppClassLoader
        MyClassLoader classLoader = new MyClassLoader ("D:\\test");
        Class clazz = classLoader.loadClass("com.cry.User");
        Object obj = clazz.newInstance();
    }
```

- 调用main方法后，类加载器什么时候触发加载的？
    - 首先java.exe调用底层的jvm.dll文件（C++代码）创建jvm
    - 然后这个c++程序就会创建引导类的实例，然后引导类实例会调用java代码去加载Launcher类的实例，在Launcher的构造方法就会创建扩展类加载器和应用类加载器。
    - JVM默认使用Launcher的getClassLoader()方法返回的类加载器AppClassLoader的实例加载我们的应用程序，所以要加载一个类，就会先调用Launcher的getClassLoader()，再调用返回AppClassLoader的loadClass方法来完成类加载。
    - 然后C++程序那边看加载完了就会调用类的main方法。
    
![image](https://user-images.githubusercontent.com/27798171/180396132-d41b43a9-d468-404d-8924-0bc3cc544c00.png)
      
```java
Launcher源码

public class Launcher {
    private ClassLoader loader;

    //Laucher的构造方法
    public Launcher() {
        Launcher.ExtClassLoader var1;
        try {
            //1. 构造扩展类加载器，在构造的过程中将其父加载器设置为null
            var1 = Launcher.ExtClassLoader.getExtClassLoader();
        } catch (IOException var10) {
            throw new InternalError("Could not create extension class loader", var10);
        }

        try {
            //2. 构造应用类加载器，在构造的过程中将其父加载器设置为ExtClassLoader， 
            this.loader = Launcher.AppClassLoader.getAppClassLoader(var1);
        } catch (IOException var9) {
            throw new InternalError("Could not create application class loader", var9);
        }

        Thread.currentThread().setContextClassLoader(this.loader);
        ......
        }

    }

    public ClassLoader getClassLoader() {
        // 3.Launcher的loader属性值是AppClassLoader，我们一般都是用这个类加载器来加载我们自己写的应用程序
        return this.loader;
    }
}
```

- 双亲委派 
    - 什么是双亲委派？看源码的classLoad类的loadClass方法就知道，它先检查这当前类加载器（默认类加载器是应用程序类加载器）有没有加载过这个类，有就直接返回这个Class对象，没有就先看看当前类加载器有没有父加载器，有就尝试给父加载器加载，没有父加载器就给引导类加载器加载。还是加载不了的话就尝试自己加载。说白了就是先一层一层往上询问有没有加载过啊，没加载过就一层一层往下尝试加载。但是双亲委派机制是可以打破的，因为你可以重写classLoader方法嘛。
    - 为什么要双亲委派机制？（1）可以防止核心类被随意篡改。（2）避免类的重复加载，父加载器加载过的类，子加载器没必要再加载。
    - 什么是全盘委托机制？ 当一个类加载器装载一个类时，除非显示的使用另外一个ClassLoder，该类所依赖及引用的类也由这个类加载器载入。

```java
双亲委派机制源码

public abstract class ClassLoader {
    protected Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException
    {
        synchronized (getClassLoadingLock(name)) {
            // 检查当前类加载器是否已经加载了该类
            Class<?> c = findLoadedClass(name);
            
            //当前类加载器没加载过
            if (c == null) {
                long t0 = System.nanoTime();
                try {
                    if (parent != null) {
                        //如果当前加载器父加载器不为空则委托父加载器加载该类
                        c = parent.loadClass(name, false);
                    } else {
                        //如果当前加载器父加载器为空则委托引导类加载器加载该类
                        c = findBootstrapClassOrNull(name);
                    }
                } catch (ClassNotFoundException e) {
                    // ClassNotFoundException thrown if class not found
                    // from the non-null parent class loader
                }
                
                //父加载器和引导类加载器都加载不了
                if (c == null) {
                    // If still not found, then invoke findClass in order
                    // to find the class.
                    long t1 = System.nanoTime();
                    
                    //自己加载。（都会调用URLClassLoader的findClass方法在加载器的类路径里查找并加载该类）
                    c = findClass(name);

                    // this is the defining class loader; record the stats
                    sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                    sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                    sun.misc.PerfCounter.getFindClasses().increment();
                }
            }
            
            //正常不会执行到，忽略
            if (resolve) {
                resolveClass(c);
            }
            return c;
        }
    }
}    
```
# 内存模型


# GC

# 性能调优

# 编译器优化







# 介绍下 Java 内存区域（运行时数据区）

# 说一下方法区和永久代的关系

# Java对象的内存分配过程是如何保证线程安全的？

# 了解分代理论吗？讲一下 Minor GC、还有 Full GC

# JDK 中有几种引用类型？分别的特点是什么？

# Java 用什么方法确定哪些对象该被清理？讲一下可达性分析算法的流程

# 讲一下Java创建一个对象的过程（五步）

# 如何回收方法区

# 对象的访问定位的两种方式（句柄和直接指针两种方式）

# 新生代垃圾收集器有哪些？老年代垃圾收集器有哪些？哪些是单线程垃圾收集器，哪些是多线程垃圾收集器？各有什么特点？各基于哪一种垃圾收集算法

# 标记清楚、标记复制、标记整理分别是怎样清理垃圾的？各有什么优缺点？

# JVM 中的安全点和安全区各代表什么？写屏障你了解吗？

# 类的生命周期

# 类的加载过程

# 类加载器

# 最重要的 JVM 参数总结

# 并发标记要解决什么问题？并发标记带来了什么问题？如何解决并发扫描时对象消失问题？

# 并发的可达性分析

# 讲一下内存分配策略

# 什么时即时编译器

# 虚拟机是怎么识别出热点代码的

# SafePoint 是什么

