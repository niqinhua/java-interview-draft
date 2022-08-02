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

### 啥是符号引用？啥是直接引用？啥是静态链接？啥是动态链接？啥是字面量
- 在编译时，并不知道某个类或方法或字段的实际内存地址，因此只能使用符号引用来代替地址。
- 在类加载时，会给类或方法或字段分配实际内存地址，指向内存的指针的或句柄就是直接引用。
- 在类加载期间，符号引用替换为直接引用就是静态链接过程。
- 在程序运行期间，符号引用替换为直接引用就是动态链接过程。
- 字面量就是字符串、final定义的常量值等。

### 类加载器
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

### 双亲委派机制
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
### jvm的内存模型
- 执行一个Java程序，jvm会先向操作系统申请一块虚拟内存给这个程序，这块区域就是运行时数据区。
- jvm会把运行时数据区划分为线程共享的和线程私有的，线程共享的包括方法区和堆。线程私有的包括虚拟机栈、本地方法栈、程序计数器等。 其实jvm也会向操作系统申请直接内存来存储一些数据，这块不属于运行时数据区，资源释放不归垃圾回收器管，所以这块内存的对象用完一定要手动申请释放掉内存。 

各区域介绍：
- 方法区：存储已被虚拟机加载的类的结构信息，一些方法的字节码内容，还有逻辑上存储常量池、Class对象、静态变量等，但是实际实现上，jdk7把这些放到了堆中，其他则继续保留在方法区，jdk7对方法区的实现叫做永久代。jdk8用元空间取代永久代，元空间的存储位置是本地内存。
- 堆：堆就是存对象的嘛，大部分的对象都是存储在堆中，但方法体内的基本数据类型是存储在栈中的，或者一些经过逃逸分析的对象也会存储在栈中，也有一些对象的内存空间是分配在直接内存中的。堆一般情况是占用内存最多的地方，垃圾回收器也主要是回收这里的内存。堆主要分为新生代的eden区，幸存者1区，幸存者2区，还有一个老年代区。
- 虚拟机栈：是用来存储当前线程每一个方法的运行数据，一个方法对应一个栈帧。根据方法调用顺序一个个把对应的栈帧入栈，谁执行完了就会出栈。栈帧主要包括局部变量表(就是存局部变量的)、操作数栈(临时存放你要计算操作的数据，就是比如你要计算1+3，就得先入栈1和3，然后执行引擎会去拿这两个数据来计算，计算完再入栈到操作数栈)、返回地址(正常就是拿程序计数器中的地址，如果抛出异常就是拿异常处理器表的地址)、还有一个动态连接。
- 本地方法栈：和虚拟机栈是差不多的，只不过方法是被native修饰的，代码底层是用c或c++写的，hotspot是直接把虚拟机栈和本地方法栈合二为一的。
- 程序计数器：就是记录当前线程执行到具体哪一个字节码的行号地址，像那些循环、跳转都是依赖这个来实现的。但是它不记录native方法，操作系统有它自己的一个程序计数器来记录native方法的执行地址。
- 直接内存：一般存储一些比较简单的对象，要在直接内存申请空间一般就是调用某一些申请内存空间的native方法、或者是用ByteBuffer来创建的对象，或者是用了 Unsafe的allocateMemory方法。默认情况下，直接内存大小就是堆的可使用的最大值大小，就是总的堆大小减去一个幸存区大小，也可以通过-XX:MaxDirectMemorySize来设置，但是它只能限制用ByteBuffer申请的空间。使用直接内存在比如网络远程传输或者进程之间传输数据的时候，可以省掉从虚拟内存复制到直接内存这一步操作，加快传输速度。还有直接内存减少了垃圾回收器的工作，因为这一块资源不归垃圾回收器管。用直接内存也好扩展空间。不过缺点就是，如果发生内存泄漏，很不好排查。

![image](https://user-images.githubusercontent.com/27798171/180715524-29dd5834-f132-4338-842a-1cd99892d9d3.png)
![image](https://user-images.githubusercontent.com/27798171/180715539-99097ae1-9892-426b-9a08-dd2c40608028.png)

### 对象

#### 对象的创建过程/对象的初始化
- 检查加载：虚拟机遇到一条new指令时，首先检查这个指令的参数能否在常量池中定位到这个类的符号引用，并且检查这个类是否被加载、解析、初始化过。没有就先执行这个类的类加载过程。
- 分配内存：然后就是给对象在堆中分配一块确定大小的内存。（展开描述参考[对象内存分配](#allocObject)）
- 内存空间初始化：内存分配完后，就会内存空间初始化，就是给这个实例对象的某些变量赋初始值。
- 设置对象的对象头：然后设置对象的对象头。（展开描述参考[对象头](#ObjetHeader)）
- 对象的初始化：然后才会把构造体的一些入参，赋值给这个对象属性。

#### 对象内存分配方法
<a id = "allocObject" />
- 对象在堆中分配一块确定大小的内存，可以用按顺序整整齐齐分配内存的方式，叫做指针碰撞，也可以用随机分配内存的方式，叫做空闲列表。空闲列表一般也就只有CMS会用，其他基本都是指针碰撞。选择哪种分配方式由 Java 堆是否规整决定，而 Java 堆是否规整又由所采用的的垃圾收集器是否带有压缩整理功能决定。
    - 指针碰撞就是有个分界点指针，它的一边是已经分配了的内存，另一边是还没分配的内存，分配内存给某个对象的时候就直接移动要分配的内存的距离，这种一般用于老年代的垃圾回收器的。
    - 空闲列表就是，空闲内存和已用内存是随机存放的。所以肯定得有一个记录表来记录哪一些区域是可用的，哪些是不可用的。
    - 还有要注意的点就是，在分配内存的时候涉及到并发问题，（展开描述参考[分配内存的时候涉及到并发问题](#TlabAndCas)）

#### 对象内存分配流程
- 先查看这个对象有没有逃逸，就是有没有传递参数给别的方法，或者有没有赋值给其他线程中的变量，如果没有逃逸，就直接栈上分配。不用垃圾回收，直接随着栈帧的消失而消失。
- 再看这个对象是不是大对象，是就直接放到堆的老年代里面，这样子可以避免大对象在新生代里面复制来复制去
- 然后就可以通过TLAB或者CAS分配到eden区，eden区如果满了就会发生young GC，eden区活下来的对象就会复制到幸存者区，后面每young GC一次，如果还能活下来，就两个幸存者区复制来复制去
- 一般情况就是一个对象的分代年龄满15次就会放到old区，但也有特殊情况，就是年龄1+年龄2+年龄n的多个年龄对象总和超过了幸存者区的一半，此时就会把年龄大于等于n以上的对象都放入老年代。
- 在发生minorGC之前有一个空间分配担保机制，就是说，它会检查老年代最大的连续可用空间是否大于新生代所有的对象空间，如果大于，那就可以minorGC。 如果小于，就根据是否担保jvm配置和之前平均晋升到老年代的对象大小，如果配置成担保且之前平均晋升到老年代的对象大小小于老年代剩余的可用连续空间，就尝试minorGC，如果尝试失败就fullGC。其他情况就直接fullGC。


#### 对象分配内存时并发问题
<a id = "TlabAndCas" />
- 在分配内存的时候涉及到并发问题，可能多个线程同时抢占某一块内存，这种情况一般都是用cas或者TLAB去处理。
    - cas就是自旋比较更新前的情况和要更新时的情况，一样就分配成功。
    - TLAB就是先给每个线程划分一块私有内存空间，然后要给对象分配空间就在这个私有空间分配，不够空间了，就再申请，这样大大减少线程之间的竞争，提高分配效率。

#### 对象头
<a id = "ObjetHeader" />

- 一般一个对象的内存空间包括三个东西:对象头、对象实例数据、对齐填充
- 对象实例数据：对象里各字段的内容。
- 对齐填充：由于JVM规定对象大小必须是8字节的整数倍倍，所以对齐填充就是补位用的。
- 对象头：对象头主要markword、类型指针、数组长度。
    - markword：存储对象自身的运行时数据，对象在不同锁状态下，存储的内容不一样。
        - 正常状态也就是无锁态存储的还有对象的hashcode、分代年龄、是否偏向锁标志位为0、锁标志位。
        - 如果升级为轻量级锁，有一个线程A访问同步块并获取锁时，会在对象头和栈帧中的锁记录记录线程ID，以后该线程在进入和退出同步块时不需要进行CAS操作来进行加锁和解锁，只需要简单的看对象头中的线程ID和当前线程是否一致。此时对象头会把hashCode换成线程id和 epoch（对象更偏向哪个锁），是否偏向锁标志位为1。
        - 在偏向锁的基础上，又有另外一个线程B进来，这时判断对象头中存储的线程A的ID和线程B不一致，就会使用CAS竞争锁，并且升级为轻量级锁。会在线程栈中创建一个锁记录，将Mark Word复制到锁记录中，然后线程尝试使用CAS将对象头的Mark Word替换成指向栈中的锁记录的指针，此时对象头也只会存储这个指针和锁标志位，如果替换成功，则当前线程获得锁失败，表示其他线程竞争锁，当前线程便尝试CAS来获取锁。
        - 当线程没有获得轻量级锁时，线程会CAS自旋10次之后仍然未获得锁，就会升级成为重量级锁。成为重量级锁之后，线程会进入阻塞队列(EntryList)，线程不再自旋获取锁，而是由CPU进行调度，线程串行执行。此时对象头也只会存储重量级锁的指针和锁标志位。
        - 当在GC的时候，对象就只会存储锁标志位。
    - 类型指针：即对象指向它的类元数据的指针，虚拟机通过这个指针来确定这个对象是哪个类的实例。
    - 数组长度：如果对象是数组，那对象头还会存储数组的长度。

32位对象头：
![image](https://user-images.githubusercontent.com/27798171/180725337-9edb8bc2-10b8-412d-81e9-e8ef210e1eea.png)

64位对象头的运行时数据存储：：
![image](https://user-images.githubusercontent.com/27798171/180729096-ab8744be-4cb4-4394-8d8f-9dcf1869742b.png)

# GC
### 是不是垃圾判断
#### 怎么判断对象是不是垃圾
- 引用计数法：就是在对象中添加一个引用计数器，有一个地方引用它就加 一，少一个引用减一，一直减到0，就说明这个对象是垃圾。但是弊端就是不能处理循环依赖。
- 可达性分析：就是先确定所有叫做GCRoot的对象作为起始节点，向下一层一层搜索引用关系，如果一个对象到GCRoot没有任何引用链，就说明这个对象是垃圾。
    - 这里的引用关系，包括强软弱虚四种引用，正常我们用的都是强引用。（展开描述参考[强软弱性引用](#reference)）
    - 可以作为GCRoot的对象包括：虚拟机栈中的本地变量表中引用的对象、本地方法栈中的引用对象、方法区的引用类型的静态变量和字符串常量池引用的对象、 被synchronize持有的对象、jvm的内部引用(比如系统类加载器、class对象、异常 类对象)
    - 不过要注意的一点，就是就算通过可达性分析算出来某个对象是垃圾，他也不一定会被回收，如果他在finalize方法中又给这个对象加了引用链到GCRoot，GC之前可能会调用这个方法，那他就不会被回收。只是可能，因为这个对象可能在执行finalize方法前，可能就被回收掉了，finalize方法是不太可靠的，也不建议用。

#### 怎么判断类是不是无用的类
方法区主要回收的是无用的类，那么如何判断一个类是无用的类呢? 

- 这个类已经没有任何活的实例对象
- 加载该类的类加载器已经被回收了
- Class对象没有被引用且无法通过反射访问这个类

jvm自带的类加载器加载的类是不会被卸载的，但是由我们自定义的类加载器加载的类是可能被卸载的。

#### 强软弱虚引用
- 我们平时 a = new A()用的就是强引用。这种对象只要与GCRoot有引用链，就不会被回收
- 软引用就是要OOM之前就会被回收。 
- 弱引用就是GC的时候一定会被回收。 ThreadLocal中就使用了WeakReference来避免内存泄漏。
- 虚引用就是，也是GC的时候一定会被回收，但是回收的时候会收到一个通知。必须配合引用队列使用，当垃圾回收器准备回收一个对象时，如果发现它还有虚引用，就会在回收对象之前，把这个虚引用加入到与之关联的引用队列中。自己的程序可以写如果发现某个虚引用已经被加入到引用队列，那么就可以在所引用的对象的内存被回收之前采取必要的行动，比如打印点回收日志。虚引用主要用来跟踪对象被垃圾回收的活动。

```java
（1）强引用
public static User user = new User();
（2）软引用
public static SoftReference<User> user = new SoftReference<User>(newUser());
（3）弱引用
public static WeakReference<User> user = new WeakReference<User>(newUser());
（4）虚引用
ReferenceQueue referenceQueue  = new ReferenceQueue()
PhantomReference<byte[]> reference = new PhantomReference<byte[]>(new byte[1], referenceQueue);
referenceQueue.poll()!=null循环获取拿到gc事件
```

### 垃圾回收算法
- 第一个就是复制算法，就是把内存分成两块，一边存满了，就先GC，还活着的对象或者新增的对象就都复制到另外一边，刚刚那一边就整块一起清空掉，又满了，就又复制另外一边。他这种复制得越少效率就越高，所以他适合用于新生代，新生代能活的对象并不多。但是这种分成两块的浪费了一大半空间，所以它有一个优化版本，就是分成8:1:1。这种就只需要浪费10分之一的空间而已。大部分的新生代垃圾回收期都是用这种算法。
- 第二个算法就是标记清除算法，就是先扫描所有对象，标记出垃圾对象，然后再扫描一遍垃圾对象，一个一个清除掉。如果垃圾对象太多，它标记效率就很低，所以这种算法适合老年代。缺点就是会造成很多内存碎片。只有CMS是用这种算法
- 第三个算法就是标记整理算法，就是先扫描所有对象，标记出垃圾对象后，剩下的活的对象全部移动到内存的一端，其余的那块内存就都直接清理掉。除了CMS以外，其他老年代垃圾回收器都是这种算法。

### 垃圾收集器
![image](https://user-images.githubusercontent.com/27798171/182085731-1ae230f2-7b53-4cb1-a73c-6e4fc1b975f8.png)

|分代|名称|优点|缺点|开启参数|算法|使用场景|since version|备注|
| ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- |
|新生代|Serial|简单高效|stop the world|-XX:+UseSerialGC|单线程、复制算法|内存较小的client模式下首选的新生代垃圾收集器|jdk1|无
|新生代|ParNew|多cpu环境下，比Serial效果好|stop the world、cpu核数少时线程切换的开销使得没有serial性能好|‘-XX:+UseParNewGC或’-XX:+UseConcMarkSweepGC|多线程并行、复制算法|server首选、jdk6之前，老年代用CMS，新生代只能用Serial或ParNew|？|无|
|新生代|Parallel Scavenge或称throughput收集器|高吞吐量、带有GC Ergonomics（gc自适应调节策略，使用-XX:+UseAdaptiveSizePolicy开启），无需手工指定新生代大小，survivor比例，晋升老年代对象的年龄等参数。|stop the world；不适合响应时间要求高的应用|‘-XX:+UseParallelGC或-XX:+UseParallelOldGC|多线程、复制算法|关注高吞吐量并且响应性要求不高的应用|？|UseParallelGC时老年代为SerialOldGC，UseParallelOldGC时老年代为ParallelOldGC|
|老年代|Serial Old|简单高效|stop the world|-XX:+UseSerialOldGC|单线程、标记整理|client首选，搭配ParNew使用，或者在jdk6之前搭配Parallel Scavenge使用，也可在server模式下作为CMS的备胎（CMS可能会因为浮动垃圾而发生concurrent mode failure的错误，此时需要serial old上位）|？|无|
|老年代|Parallel Old|高吞吐量|stop the world，不适合响应时间要求高的应用|-XX:+UseParallelOldGC|多线程并行，标记整理|server模式默认选项。搭配Parallel Scavenge，关注吞吐量及cpu资源敏感的场合|jdk6|该选项自动设置 -XX:+UseParallelGC选项|
|老年代|CMS|停顿时间短，响应性高|对cpu资源非常敏感；无法处理浮动垃圾，可能会出现Concurrent mode failure；标记清除算法容易产生垃圾碎片|-XX:+UseConcMarkSweepGC|多线程并发，标记清除|要求高响应性的互联网站或BS服务端|jdk5|打开此开关参数后，使用ParNew+CMS+Serial Old收集器组|
|老年代|G1|当内存很大时，停顿时间高，并且吞吐量高|内存较小时性能没有CMS好|-XX:++UseG1GC|使用空间整合算法，堆被划分成多个连续的大小相等的region，新生代老年代物理上不再隔离，新生代老年代各自在内部也不再要求连续。|内存大于6G的高并发低停顿应用|jdk7|可能会在java9时作为默认GC|


- 最古老的就是单线程的垃圾回收器，就是新生代的Serial 老年代的Serial Old这 两种垃圾回收器，垃圾回收时都是只有一个GC线程，还都得暂停用户线程。这种一般适合几十兆到一百兆的堆空间，这种垃圾回收器已经很少用了。

![image](https://user-images.githubusercontent.com/27798171/182285411-890dbd49-049d-4a30-86d5-d52a856ec9c9.png)

- 后来就有了多线程的垃圾回收器，就是新生代的Parallel Scavenge 和老年代的 Parallel Old，垃圾回收时是多个GC线程的，也得暂停用户线程。这种一般适合一百兆到几个G的堆空间。这种垃圾回收器关注的是吞吐量，jdk8默认也是这一套。后面又有ParNew来搭配CMS，因为Parallel Scavenge没法兼容CMS。ParNew和Parallel Scavenge其实差不多。

![image](https://user-images.githubusercontent.com/27798171/182285440-b9c59337-d656-4c84-9f14-fa356d29418c.png)

- 如果你想缩短响应时间的话，就可以用CMS老年代垃圾回收器，它可以搭配单线程的Serial和多线程的ParNew这两种新生代垃圾回收器。他一般适用于几个G到几十G的堆空间。
    - CMS的流程就是先暂停用户线程来标记所有可以作为GCRoot 的对象，因为 GCRoot对象很少，所以这里暂停时间很短，这一步叫做初始标记。
    - 然后用户线程和GC线程就可以一起跑了，然后GC根据GCRoot对象的扫描所有对象找到可回收的对象，这一步是花费最长的时间，叫做并发标记。
    - 要注意的一点就是GC线程边找垃圾对象，用户线程可能还会对对象的引用进行修改，就是可能会发生漏标问题，就是会产生一些漏标的垃圾对象，而且用户线程还会边产生更多垃圾，所以还得暂停一下用户线程，来重新收集一下还没标记到的垃圾，这一步由于还没标记的垃圾一般不多，所以不会暂停用户线程很久，这一步叫做重新标记。
    - 然后最后用户线程和GC线程再一起跑，GC就去清除掉这些垃圾，这一步叫 做并发清除。
    - CMS的缺点最关键的就是有浮动垃圾，因为最后一步并发清除，GC线程边回收垃圾，用户线程边产生垃圾。而且由于浮动垃圾，GC之前必须预留这部分空间，而且很恐怖的就是，如果浮动垃圾太多，就直接暂停所有线程，用Serial old替代来执行FullGC。第二点就是因为用户线程和GC线程一起跑，只能用标记清除算法，那肯定就会有内存碎片，如果因为没有连续空间存不下对象，也会用Serial old来替代做标记整理清除算法。第三点就是因为并发的GC线程，CPU一般要4核以上比较好;

![image](https://user-images.githubusercontent.com/27798171/182285506-b997d3fc-fdf9-4160-879a-25dd4e349b30.png)

- G1: CMS没有办法控制暂停用户线程 的时间，如果想要尽量控制可以用 G1。它一般适用于几十G到上百G的堆空间。G1就是将堆划分成很多个相等大小的 region，每个region可以是eden区，幸存者区，也可以说老年区，还有一个专门大对象的区域，你也可以直接当做老年区来看。它最大特点，就是在标记垃圾之后，可以根据我们期望的暂停用户线程的时间来挑选出最有回收价值的region来回收。
    - G1的流程就是先暂停用户的线程来标记所有可以作为GCRoot的对象，因为 GCRoot对象很少，所以这里暂停时间很短，然后还有修改两个TAMS的指针，这两个指针就是划分出后面用户线程和GC线程一起跑的时候，用户线程新创建的对象就存到这两个指 针之间的空间，这一步叫做初始标记。
    - 然后用户线程和GC线程就可以一起跑了，然后GC根 据GCRoot对象的扫描所有对象找到可回收的对象，这一步是花费最长的时间，叫做并发标记。
    - 要注意的一点就是GC线程边找垃圾对象，用户线程可能还会对对象的引用进行修改， 就是可能会发生漏标问题，漏标问题等下说，就是会产生一些漏标垃圾的对象，所以还得暂停一下用户线程，来重新标记一下这些漏标的垃圾对象，这一步由于还没标记的垃圾一般不多，所以不会暂停用户线程很久，这一步叫做最终标记。
    - 然后GC就去统计哪些region最有回收价值，根据设置的期望用户线程暂停时间的值来设置回收计划，把region活的对象赋值到别的地方去，然后再清理掉整个region，这一步由于要移动对象，所以用户线程是必 须暂停的，这一步叫做筛选回收。

# 性能调优
### 常用jvm参数
关于栈的：
- -Xss：每个线程的栈大小

关于堆的：
- -Xms：堆的初始大小，默认物理内存的1/64
- -Xmx：堆的最大大小，默认物理内存的1/4

关于新生代的：
- -Xmn：新生代大小
- -XX:NewRatio：默认2，代表新生代大小:老年代大小=1:2
- -XX:SurvivorRatio：默认8，代表survivor区:eden区=1:8
- -XX:+UseAdaptiveSizePolicy：默认开启，根据GC的情况自动计算Eden、From 和 To 区的大小，会导致-XX:SurvivorRatio发生变化。
    - 不要和SurvivorRatio参数显示设置搭配使用，一起使用会导致参数失效。
    - 在JDK 1.8 中，如果使用CMS，无论 UseAdaptiveSizePolicy 如何设置，都会将 UseAdaptiveSizePolicy 设置为 false；

关于youngGc的：
- -XX:PretenureSizeThreshold：大对象的大小，如果对象超过设置大小会直接进入老年代，不会进入年轻代，这个参数只在 Serial 和ParNew两个收集器下有效。
- -XX:MaxTenuringThreshold：默认15，CMS默认6，Minor GC时对象年龄达到该值则晋升到老年代。
- -XX:TargetSurvivorRatio：默认50%，Minor GC之后，年龄1+年龄2+年龄n的多个年龄对象总和超过了幸存者区的一半，此时就会把年龄大于等于n以上的对象都放入老年代。
- -XX:-HandlePromotionFailure：是否设置空间分配担保，JDK7及以后这个参数就失效了。年轻代每次minor gc之前JVM都会检查老年代最大的连续可用空间是否大于新生代所有的对象空间(包括垃圾对象)， 如果大于，那就可以minorGC。 如果小于，就根据是否担保jvm配置和之前平均晋升到老年代的对象大小，如果配置成担保且之前平均晋升到老年代的对象大小小于老年代剩余的可用连续空间，就尝试minorGC，如果尝试失败就fullGC。其他情况就直接fullGC。

关于方法区的：
- -XX:MaxMetaspaceSize：元空间最大值，默认-1（不限制）
- -XX:MetaspaceSize：元空间触发fullgc的初始阈值（元空间无固定初始大小），默认是21M左右，达到该值就会触发fullgc进行类型泄卸载。
    - 备注1：同时收集器会对该值进行调整: 如果释放了大量的空间， 就适当降低该值; 如果释放了很少的空间， 那么在不超过-XX:MaxMetaspaceSize(如果设置了的话) 的情况下，适当提高该值。
    - 备注2：由于调整元空间的大小需要FullGC，这是非常昂贵的操作，如果应用在启动的时候发生大量Full GC，通常都是由于永久代或元空间发生了大小调整，基于这种情况，一般建议在JVM参数中将MetaspaceSize和MaxMetaspaceSize设置成一样的值，并设置得比初始值要大， 对于8G物理内存的机器来说，一般我会将这两个值都设置为256M。
- -XX:PermSize：永久代的初始容量

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

# 最重要的 JVM 参数总结

# 并发标记要解决什么问题？并发标记带来了什么问题？如何解决并发扫描时对象消失问题？

# 并发的可达性分析

# 讲一下内存分配策略

# 什么时即时编译器

# 虚拟机是怎么识别出热点代码的

# SafePoint 是什么

