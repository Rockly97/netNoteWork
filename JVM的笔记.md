## 内存结构

### 内存结构概述

![](https://pic2.zhimg.com/80/v2-df75e799d05cfd4ed6aaaf84e3ee5d81_720w.jpg)

其中详细图如下：

![](https://pic1.zhimg.com/80/v2-ded3100b831c94a98a35ef475f3aaa4c_720w.jpg)









##  JVM的架构模型

java编译的输入的指令流基本上是一种基于**栈的指令集架构**，另外一种是基于**寄存器的指令集架构** 

**栈的指令集的构架特点**

- 设计简单，使用资源受限的系统。
- 避开了寄存器分配难题，使用零地址指令方式分配。
- 不需要硬件支持，可移植性好。
- 执行过程依赖操作栈，指令集更小，编译器容易实现。

**寄存器的指令集的特点**
- 采用经典的x86二进制指令集，如传统PC。
- 完全依赖硬件。
- 性能优秀执行效率高。
- 花费更少的时间去完成指令。
- 寄存器一般以一地址指令、二地址、三地址为主。

**总结：** 由于跨平台的设计，Java的指令都是根据栈来设计的。

### JVM的生命周期

- **启动**

启动是通过引导类加载器（bootstrap class loader）创建一个初始类（initial class）来完成，这个类是由虚拟机的具体实现来指定的。

- **执行**

一个运行中的Java虚拟机有清晰的任务：执行Java程序

程序开始时他才执行，程序结束时他就停止。

执行Java程序的时候，真真正正的执行的是一个叫Java虚拟机的进程

- **退出**

退出的情况有以下几种：

- 程序的正常结束
- 程序执行中出现异常或错误而终止
- 操作系统出现错误导致Java进程终止
- 某线程调用Runtime类的`halt()`方法或者System类的`exit()`方法，并且Java安全管理器也允许这次的操作。

## 类加载器子系统的加载过程

类加载器子系统负责从文件系统或者网络中加载 class 文件，class 文件在文件开头有特定的文件标识。

ClassLoader 只负责 class 文件的加载，至于它是否可以运行，则由 Execution Engine 决定。

加载的类信息存放于一块称为**方法区**的内存空间。除了类的信息外，方法区中还会存放**运行时常量池信息**，可能还包括**字符串字面量**和**数字常量**（这部分常量信息是 class 文件中常量池部分的内存映射）

![](https://pic2.zhimg.com/80/v2-ad83444dc7a931551fbb1517c19342e9_720w.jpg)



1. class file 存在于本地硬盘上，可以理解为设计师画在纸上的模板，而最终这个模板在执行的时候是要加载到JVM 当中来根据这个文件实例化出 n 个一模一样的实例。

2. class file 加载到 JVM 中，被称为 DNA 元数据模板，放在方法区。

3. 在 class 文件 -> JVM -> 最终成为元数据模板，此过程就要一个运输工具（类装载器Class Loader），扮演一个快递员的角色。

   

![](https://pic4.zhimg.com/80/v2-cff1afc30effd71932626d5006962ca3_720w.jpg)

### 类的加载过程

例如下面的一段简单的代码

```java
public class HelloLoader {
    public static void main(String[] args) {
        System.out.println("我已经被加载啦");
    }
}
```

它的加载过程:

![](https://pic3.zhimg.com/80/v2-4f4ba4e69dd181ebadd50a9ff190ee12_720w.jpg)

完整的流程图如下所示



![img](https://pic4.zhimg.com/80/v2-8656702656d71d625004f77dc9a158d7_720w.jpg)



#### 加载

1. 通过一个类的全限定名获取定义此类的二进制字节流
2. 将这个字节流所代表的静态存储结构转化为方法区的运行时数据结构
3. **在内存中生成一个代表这个类的 java.lang.Class 对象**，作为方法区这个类的各种数据的访问入口

##### 补充：加载 .class 文件的方式

- 从本地系统中直接加载
- 通过网络获取，典型场景：Web Applet
- 从 zip 压缩包中读取，成为日后 jar、war 格式的基础
- 运行时计算生成，使用最多的是：动态代理技术
- 由其他文件生成，典型场景：JSP 应用从专有数据库中提取 .class 文件，比较少见
- 从加密文件中获取，典型的防 class 文件被反编译的保护措施



#### 链接

##### 验证 Verify

- 目的在于确保 class 文件的字节流中包含信息符合当前虚拟机要求，保证被加载类的正确性，不会危害虚拟机自身安全。
- 主要包括四种验证：文件格式验证，元数据验证，字节码验证，符号引用验证。

> 工具：Binary Viewer 查看



![img](https://pic2.zhimg.com/80/v2-2d9388c17832d72e833cf3becb49ae41_720w.jpg)



如果出现不合法的字节码文件，那么将会验证不通过

同时我们可以通过安装 IDEA 的插件，来查看我们的 class 文件



![img](https://pic2.zhimg.com/80/v2-7d906f89418d97f54df3b086daab2c91_720w.jpg)



安装完成后，我们编译完一个 class 文件后，点击 view 即可显示我们安装的插件来查看字节码方法了



![img](https://pic2.zhimg.com/80/v2-daf55d75fdbb8e57ab6d6540b410f681_720w.jpg)



##### 准备 Prepare

为类变量分配内存并且设置该类变量的默认初始值，即零值。

2 3 4 5



```java
public class HelloApp {
    private static int a = 1;  // 准备阶段为0，在下个阶段，也就是初始化的时候才是1
    public static void main(String[] args) {
        System.out.println(a);
    }
}
```

上面的变量 a 在准备阶段会赋初始值，但不是1，而是0。

- 为类变量分配内存并且设置该类变量的默认初始值，即零值；
- **这里不包含用 final 修饰的 static，因为 final 在编译的时候就会分配了，准备阶段会显式初始化；**
- **这里不会为实例变量分配初始化**，类变量会分配在方法区中，而实例变量是会随着对象一起分配到 Java 堆中。

##### 解析 Resolve

- 将常量池内的符号引用转换为直接引用的过程。
- 事实上，解析操作往往会伴随着 JVM 在执行完初始化之后再执行。
- 符号引用就是一组符号来描述所引用的目标。符号引用的字面量形式明确定义在《Java虚拟机规范》的 class文件格式中。直接引用就是直接指向目标的指针、相对偏移量或一个间接定位到目标的句柄。
- 解析动作主要针对类或接口、字段、类方法、接口方法、方法类型等。对应常量池中的 CONSTANT_Class_info、CONSTANT_Fieldref_info、CONSTANT_Methodref_info 等

#### 初始化

- 初始化阶段就是执行类构造器法`<clinit>()`的过程。
- 此方法不需定义，是 javac 编译器自动收集类中的所有类变量的赋值动作和静态代码块中的语句合并而来。
- 也就是说，当我们代码中包含 static 变量的时候，就会有 `<clinit>()` 方法
- 构造器方法中指令按语句在源文件中出现的顺序执行。
- `<clinit>()`不同于类的构造器。（关联：构造器是虚拟机视角下的 `<init>()`）
- 若该类具有父类，JVM 会保证子类的 `<clinit>()`执行前，父类的 `<clinit>()`已经执行完毕。
- 虚拟机必须保证一个类的 `<clinit>()`方法在多线程下被同步加锁。

任何一个类在声明后，都有生成一个构造器，默认是空参构造器

```java
public class ClassInitTest {
    private static int num = 1;
    static {
        num = 2;
        number = 20;
        System.out.println(num);
        System.out.println(number);  //报错，非法的前向引用
    }

    private static int number = 10;

    public static void main(String[] args) {
        System.out.println(ClassInitTest.num); // 2
        System.out.println(ClassInitTest.number); // 10
    }
}
```

关于涉及到父类时候的变量赋值过程

```java
public class ClinitTest1 {
    static class Father {
        public static int A = 1;
        static {
            A = 2;
        }
    }

    static class Son extends Father {
        public static int b = A;
    }

    public static void main(String[] args) {
        System.out.println(Son.b);
    }
}
```

我们输出结果为 2，也就是说首先加载 ClinitTest1 的时候，会找到 main 方法，然后执行 Son 的初始化，但是Son 继承了 Father，因此还需要执行 Father 的初始化，同时将 A 赋值为2。我们通过反编译得到 Father 的加载过程，首先我们看到原来的值被赋值成1，然后又被复制成2，最后返回

```bash
iconst_1
putstatic #2 <com/atguigu/java/chapter02/ClinitTest1$Father.A>
iconst_2
putstatic #2 <com/atguigu/java/chapter02/ClinitTest1$Father.A>
return
```

虚拟机必须保证一个类的 `<clinit>()`方法在多线程下被同步加锁。

```java
public class DeadThreadTest {
    public static void main(String[] args) {
        new Thread(() -> {
            System.out.println(Thread.currentThread().getName() + "\t 线程t1开始");
            new DeadThread();
        }, "t1").start();

        new Thread(() -> {
            System.out.println(Thread.currentThread().getName() + "\t 线程t2开始");
            new DeadThread();
        }, "t2").start();
    }
}
class DeadThread {
    static {
        if (true) {
            System.out.println(Thread.currentThread().getName() + "\t 初始化当前类");
            while(true) {

            }
        }
    }
}
```

上面的代码，输出结果为

```text
线程t1开始
线程t2开始
线程t2 初始化当前类
```

从上面可以看出初始化后，只能够执行一次初始化，这也就是同步加锁的过程



### 类加载器的分类

JVM 支持两种类型的类加载器 。分别为**引导类加载器（Bootstrap ClassLoader）**和**自定义类加载器（User-Defined ClassLoader）**。

从概念上来讲，自定义类加载器一般指的是程序中由开发人员自定义的一类类加载器，但是 Java 虚拟机规范却没有这么定义，而是**将所有派生于抽象类 ClassLoader 的类加载器都划分为自定义类加载器**。

无论类加载器的类型如何划分，在程序中我们最常见的类加载器始终只有3个，如下所示：



![img](https://pic2.zhimg.com/80/v2-ce5e9f71925dc56f3942a4478ac3e8c5_720w.jpg)



**这里的四者之间是包含关系，不是上层和下层，也不是父子的继承关系。**

我们通过一个类，获取它不同的加载器

```java
public class ClassLoaderTest {
    public static void main(String[] args) {
        // 获取系统类加载器
        ClassLoader systemClassLoader = ClassLoader.getSystemClassLoader();
        System.out.println(systemClassLoader);

        // 获取其上层的：扩展类加载器
        ClassLoader extClassLoader = systemClassLoader.getParent();
        System.out.println(extClassLoader);

        // 试图获取 根加载器
        ClassLoader bootstrapClassLoader = extClassLoader.getParent();
        System.out.println(bootstrapClassLoader);

        // 获取自定义加载器
        ClassLoader classLoader = ClassLoaderTest.class.getClassLoader();
        System.out.println(classLoader);

        // 获取String类型的加载器
        ClassLoader classLoader1 = String.class.getClassLoader();
        System.out.println(classLoader1);
    }
}
```



得到的结果，从结果可以看出，根加载器无法直接通过代码获取，同时目前用户代码所使用的加载器为系统类加载器。同时我们通过获取 String 类型的加载器，发现是 null ，那么说明 String 类型是通过根加载器进行加载的，也就是说 **Java 的核心类库都是使用根加载器进行加载的**。



### 虚拟机自带的加载器

#### 启动类加载器（引导类加载器，Bootstrap ClassLoader）

- 这个类加载使用 C/C++ 语言实现的，嵌套在 JVM 内部。
- 它用来加载 Java 的核心库（JAVAHOME/jre/lib/rt.jar、resources.jar 或 sun.boot.class.path 路径下的内容），用于提供 JVM 自身需要的类
- 并不继承自 java.lang.ClassLoader，没有父加载器。
- 加载扩展类和应用程序类加载器，并指定为他们的父类加载器。
- 出于安全考虑，Bootstrap 启动类加载器只加载包名为 java、javax、sun 等开头的类

#### 扩展类加载器（Extension ClassLoader）

- Java 语言编写，由 `sun.misc.Launcher$ExtClassLoader` 实现。
- 派生于 ClassLoader 类
- 父类加载器为启动类加载器
- 从 java.ext.dirs 系统属性所指定的目录中加载类库，或从 JDK 的安装目录的 jre/lib/ext 子目录（扩展目录）下加载类库。如果用户创建的 JAR 放在此目录下，也会自动由扩展类加载器加载。

####  应用程序类加载器（系统类加载器，AppClassLoader）

- Java 语言编写，由` sun.misc.LaunchersAppClassLoader `实现
- 派生于 ClassLoader 类
- 父类加载器为扩展类加载器
- 它负责加载环境变量 classpath 或系统属性 java.class.path 指定路径下的类库
- 该类加载是程序中默认的类加载器，一般来说，Java 应用的类都是由它来完成加载
- 通过 ClassLoader#getSystemClassLoader() 方法可以获取到该类加载器

####  用户自定义类加载器

在 Java 的日常应用程序开发中，类的加载几乎是由上述3种类加载器相互配合执行的，在必要时，我们还可以自定义类加载器，来定制类的加载方式。 为什么要自定义类加载器？

- 隔离加载类
- 修改类加载的方式
- 扩展加载源
- 防止源码泄漏

用户自定义类加载器实现步骤：

- 开发人员可以通过继承抽象类 java.lang.ClassLoader 类的方式，实现自己的类加载器，以满足一些特殊的需求
- 在 JDK 1.2 之前，在自定义类加载器时，总会去继承 ClassLoader 类并重写 loadClass() 方法，从而实现自定义的类加载类，但是在 JDK 1.2 之后已不再建议用户去覆盖 loadClass() 方法，而是建议把自定义的类加载逻辑写在 findClass() 方法中
- 在编写自定义类加载器时，如果没有太过于复杂的需求，可以直接继承 URIClassLoader 类，这样就可以避免自己去编写 findClass() 方法及其获取字节码流的方式，使自定义类加载器编写更加简洁。

#### 查看根加载器所能加载的目录

刚刚我们通过概念了解到了，根加载器只能够加载 java/lib目录下的class，我们通过下面代码验证一下

```java
public class ClassLoaderTest1 {
    public static void main(String[] args) {
        System.out.println("*********启动类加载器************");
        // 获取BootstrapClassLoader 能够加载的API的路径
        URL[] urls = sun.misc.Launcher.getBootstrapClassPath().getURLs();
        for (URL url : urls) {
            System.out.println(url.toExternalForm());
        }

        // 从上面路径中，随意选择一个类，来看看他的类加载器是什么：得到的是null，说明是  根加载器
        ClassLoader classLoader = Provider.class.getClassLoader();
    }
}
```

得到的结果

```text
*********启动类加载器************
file:/E:/Software/JDK1.8/Java/jre/lib/resources.jar
file:/E:/Software/JDK1.8/Java/jre/lib/rt.jar
file:/E:/Software/JDK1.8/Java/jre/lib/sunrsasign.jar
file:/E:/Software/JDK1.8/Java/jre/lib/jsse.jar
file:/E:/Software/JDK1.8/Java/jre/lib/jce.jar
file:/E:/Software/JDK1.8/Java/jre/lib/charsets.jar
file:/E:/Software/JDK1.8/Java/jre/lib/jfr.jar
file:/E:/Software/JDK1.8/Java/jre/classes
null

```

### 关于ClassLoader

ClassLoader 类，它是一个抽象类，其后所有的类加载器都继承自 ClassLoader（不包括启动类加载器）

![img](https://pic2.zhimg.com/80/v2-8049a22c79132a8b0912eb5271066921_720w.jpg)



`sun.misc.Launcher` 它是一个 Java 虚拟机的入口应用



![img](https://pic4.zhimg.com/80/v2-64e3ec9e5167dbbc14d3076440e1250b_720w.jpg)



获取 ClassLoader 的途径

- 获取当前 `ClassLoader：clazz.getClassLoader()`
- 获取当前线程上下文的 `ClassLoader：Thread.currentThread().getContextClassLoader()`
- 获取系统的 `ClassLoader：ClassLoader.getSystemClassLoader()`
- 获取调用者的 `ClassLoader：DriverManager.getCallerClassLoader()`

## 双亲委派机制

Java 虚拟机对 class 文件采用的是**按需加载**的方式，也就是说当需要使用该类时才会将它的 class 文件加载到内存生成 class 对象。而且加载某个类的 class 文件时，Java 虚拟机采用的是**双亲委派模式**，即把请求交由父类处理，它是一种任务委派模式。

### 工作原理

- 如果一个类加载器收到了类加载请求，它并不会自己先去加载，而是把这个请求委托给父类的加载器去执行；
- 如果父类加载器还存在其父类加载器，则进一步向上委托，依次递归，请求最终将到达顶层的启动类加载器；
- 如果父类加载器可以完成类加载任务，就成功返回，倘若父类加载器无法完成此加载任务，子加载器才会尝试自己去加载，这就是双亲委派模式。



![img](https://pic1.zhimg.com/80/v2-1ca130f69bea8101050848e06a74a460_720w.jpg)



### 双亲委派机制举例

当我们加载 JDBC.jar 用于实现数据库连接的时候，首先我们需要知道的是 JDBC.jar是基于 SPI 接口进行实现的，所以在加载的时候，会进行双亲委派，最终从根加载器中加载 SPI 核心类，然后在加载 SPI 接口类，接着在进行反向委派，通过线程上下文类加载器进行实现类 JDBC.jar 的加载。



![img](https://pic4.zhimg.com/80/v2-0d61615ad2a5b16cfd3902770eca909f_720w.jpg)



### 沙箱安全机制

自定义 String 类，但是在加载自定义 String 类的时候会率先使用引导类加载器加载，而引导类加载器在加载的过程中会先加载 JDK 自带的文件（rt.jar包中java\lang\String.class），报错信息说没有 main 方法，就是因为加载的是 rt.jar 包中的 String 类。这样可以保证对 Java 核心源代码的保护，这就是沙箱安全机制。

### 双亲委派机制的优势

通过上面的例子，我们可以知道，双亲机制可以

- 避免类的重复加载
- 保护程序安全，防止核心 API 被随意篡改
- 自定义类：java.lang.String
- 自定义类：java.lang.ShkStart（报错：阻止创建 java.lang 开头的类）

## 其它

### 如何判断两个 class 对象是否相同

在 JVM 中表示两个 class 对象是否为同一个类存在两个必要条件： - 类的完整类名必须一致，包括包名。 - 加载这个类的 ClassLoader（指 ClassLoader 实例对象）必须相同。

换句话说，在 JVM 中，即使这两个类对象（class对象）来源同一个 Class 文件，被同一个虚拟机所加载，但只要加载它们的 ClassLoader 实例对象不同，那么这两个类对象也是不相等的。

JVM 必须知道一个类型是由启动加载器加载的还是由用户类加载器加载的。如果一个类型是由用户类加载器加载的，那么 JVM 会将**这个类加载器的一个引用作为类型信息的一部分保存在方法区中**。当解析一个类型到另一个类型的引用的时候，JVM 需要保证这两个类型的类加载器是相同的。

### 类的主动使用和被动使用

Java 程序对类的使用方式分为：主动使用和被动使用。 主动使用，又分为七种情况：

- 创建类的实例
- 访问某个类或接口的静态变量，或者对该静态变量赋值
- 调用类的静态方法
- 反射（比如：Class.forName（"com.atguigu.Test"））
- 初始化一个类的子类
- Java 虚拟机启动时被标明为启动类的类
- JDK 7 开始提供的动态语言支持：
- java.lang.invoke.MethodHandle 实例的解析结果 REF getStatic、REF putStatic、REF invokeStatic 句柄对应的类没有初始化，则初始化

除了以上七种情况，其他使用 Java 类的方式都被看作是对类的**被动使用**，都**不会导致类的初始化**。

---

# 运行时数据区概述及线程

> 举例子：厨师做饭就相当于执行引擎，而放在旁的锅碗瓢盆和切好备用的菜就是运行时的数据

而运行时的数据区就是在内存中，而JVM内存布局规定了Java在运行中的内存的申请、分配、管理的策略。保证了JVM的高效运行。 

**不同的JVM对内存的划分方式和管理机制存在部分的差异**

运行区的数据区有--> **方法器、程序计数器、本地方法栈、虚拟机栈、堆**

每个线程独享-- > 程序计数器、栈、本地方法栈。  

线程间共享-->  堆、方法区、堆外内存（永久代或元空间、代码缓存）

> 每个JVM都对应一个Runtime的实例。既为运行环境，相当于内存结构中间的那个框框：运行时环境。

> 一个JVM对应一个运行时的数据区，一个Java程序对应一个进程，一个进程对应一个JVM实例，一个进程中的多个线程需要共享一个方法区和堆，每个线程都拥有独立的程序计数器、本地方法栈、虚拟机栈

## 线程

线程是一个程序的运行单元，一个JVM中允许一个应用有多个线程并行的运行
> 在Hostpot JVM中，每个线程都与操作系统中的本地线程直接映射。

操作系统中负责所有的线程的安排调度到任何一个可用的`CPU`上去。一旦本地的线程初始化成功他就会调用，它就会调用到Java的线程中的`run()`方法中去。

在Hotspot JVM中后台系统线程主要是以下几个：
- **虚拟机线程** 这种线程的操作需要JVM到达安全点，这种操作的必须在不同的线程中发生的原因是他们都需要JVM到达安全点，这样堆才不会发生变化。
- **周期任务线程**  在时间周期事件的体现（比如中断），他们一般用于周期性操作的调度任务执行。
- **GC线程** 为JVM提供不同种类的垃圾收集行为提供支持
- **编译线程** 运行时将字节码编译到本地代码。
- **信号调度线程** 接收信号发送给JVM，在它的内部通过调用适当的方法进行处理。


## PC寄存器（程序计数器 Program Counter Register）
> JVM的PC寄存器是对物理PC寄存器的一种抽象

PC寄存器的用来存储指向下一条的地址，也即将将要执行的指令代码。由执行引擎读取下一条指令。

它是一块很小的空间，也是运行速度最快的存储区域。
在JVM中每个线程都有自己的程序计数器，它是线程私有的，生命周期与线程周期保持一致。

任何时间一个线程只有一个方法在执行，也就是所谓的当前方法。

程序的控制流的指示器(可以理解为迭代器)，分支、循环、跳转、异常处理、线程恢复等基础功能都依赖这个程序计数器完成。
**唯一一个在JVM规范中没有规定任何的`OutOtMemoryError`情况的区域**

### 举例
```java
    public class PCRegisterTest{
        public static void main(String[] args){
            int i = 0;
            int j = 20;
            int k = i + j; 
        }
        
    }
```
使用`javap -v PCRegisterTest.class` 编译class字节码文件查看。

PC寄存器去存储指令地址，而执行引擎去读取PC寄存器里的值对应的操作指令值翻译为机器指令，操作局部变量表、操作数栈。

**PC寄存器存储的字节码的地址作用？** 
CPU需要不停的切换各个线程，切换回来后就知道从哪个位置开始继续执行，改变PC寄存器的值来明确下一条该执行什么样的字节码指令。

**为什么PC寄存器会被设定为私有？**
为了能够精确地记录正在执行的当前字节码的指令地址，最好的办法自然是为每一个线程都分配一个PC寄存器。

## 虚拟机栈

栈是运行时的单位，而堆是存储的单位

栈解决的是程序如何执行，如何处理数据；堆解决的是数据存储的问题，数据如何放，放在那里。

每个线程都会创建一个虚拟机栈（私有的），其内部保存一个个的**栈帧**（Stack Frame），对应的一次次的**Java方法调用**。

生命周期与线程一致

只要是管Java程序的运行，保存方法局部变量（8种基本数据类型，对象的引用地址），部分结果，并参与方法的调用和返回。

### 特点
栈是一种快速有效的分配存储方式，访问速度仅仅次于程序计数器。

JVM直接对Java栈操作有两个：
    每个方法执行，伴随的进栈/入栈。执行任务结束后出栈工作。

> 对于每个栈来说不存在垃圾回收问题(只有进栈/出栈)

### 出现异常
Java中允许栈的大小是动态的或者是固定不变的。
- 如果采用固定大小，那每一个线程的Java容量在创建时独立选定容量。如果请求分配的栈超过了虚拟机的允许的最大容量就会抛出StackOverflowError异常。
- Java虚拟机扩展为动态的，在申请动态的扩展时无法申请到足够的空间内存，或者创建新线程没有足够的内存创建对应的虚拟机栈就会抛出一个OutOfMemoryError异常。


> 设置栈内存大小使用参数 -Xss size
> （-Xss1m -Xss1024k -Xss1048576）来设置线程的最大栈空间（栈的大小决定了函数调用的最大可达深度）


---

### 栈帧

每个线程都有自己的栈，栈中的数据都是以栈帧格式存在的，在这个线程上执行的每个方法都对应一个栈帧。

栈帧是一个内存区块，一个数据集，维系着方法执行过程中的各个数据信息。

在一条活动线程中，一个时间点上，只有一个活动的栈帧。当前正在执行的方法的栈帧是有效的就是**当前栈帧**，当前栈帧对应的方法就是**当前方法**，定义这个方法类就是**当前类**

执行引擎运行的字节码指令只针对于当前栈帧进行的操作。

如果在该方法中调用的其他方法，对应的新的栈帧会被创建出来，放在栈顶。

不同的线程中的栈帧是不允许存在相互引用的，既不可能在一个栈帧中去引用其他线程的栈帧。

**Java方法返回函数有两种，一种是正常函数的返回，二是异常抛出返回 这两种都会导致栈帧被弹出**


---

### 栈帧的内部结构

#### 局部操作表(Local Variables)
也称局部变量数组或本地变量表 

定义为**数字数组**，主要用于存储方法参数和定义在方法体内部的局部变量，这些数据类型包括基本类型，对象饮用以及return Address类型。

不存在数据安全问题（线程私有的）

局部变量表是容量大小是在编译器确定下来的，保存在Code属性的maximun local variables数据项，在方法运行期是不会改变局部变量表大小。

方法嵌套次数由栈帧决定。栈越大，方法嵌套的次数越多。
局部变量表中的变量只在当前方法调用中有效。方法结束局部变量表销毁。

##### Slot
局部变量表的基本单位。 

32位（包括return Address类型）以内的占一个slot，64位类型（long 和 double）占两个slot。

如果访问一个64bit的局部变量时，只需要前面的一个索引即可。

如果当前的帧是由构造方法或者实例方法创建的，那该对象的引用this将会放在index为0的slot处，
其余继续按照顺序排序。

栈帧中的局部变量表的槽位可以重复利用。如果一个局部变量用过了其作用域，在其作用域之后申明的新的局部变量就很可能会复用过期的局部变量的槽位。

> 局部变量在使用时必须要显示的赋值（局部变量表中不存在系统初始化过程，一旦定义就必须人为的初始化）


---
局部变量表中的变量也是重要的垃圾回收根节点，只要局部变量表中直接引用或者间接引用的对象都不会被回收。


#### 操作数栈(Operand stack)
也可以称为**表达式栈**

在方法执行过程中，根据字节码指令，往栈中写数据或提取数据，既入栈/出栈

某些字节码指令将压入栈操作数栈，其余字节码指令将操作数取出栈，使用他们把结果压入栈。（如复制，交换，求和等操作）

**如果被调用的方法带有返回值，其返回值会被压入当前栈帧的操作数帧，并更新PC寄存器中的下一条需要执行的字节码指令**

主要用于保存计算过程中变量临时的存储空间。
；每个操作数栈的深度在编译期就定义好了，保存在code的属性中，为max-stack的值
；栈中的元素任意类型：32bit占一位栈深度；64bit占二位栈深度。

#### 动态链接(Dynamic Linking)

每个栈帧内部都包含一个指向运行时常量池中该栈帧的所属方法的引用，包含这个引用的目的就是为了支持当前的方法的代码能实现动态链接

Java源文件被编译到字节码文件中时，所有的**变量**和**方法引用**都是作为符号引用（System Reference）保存在class文件的常量池（Constant pool）中，作用是为了将符号引用转换为调用方法的直接引用。

##### 方法调用
在JVM中将符号引用转换为调用方法的直接引用与方法的绑定机制有关
静态链接：

当以一个字节码文件被装进JVM内部时，被调用的目标方法在编译期可知，且运行期保持不变，这种情况将调用方法转换为直接引用

动态链接：

被调用的方法在编译期无法被确定下来，只能够在程序运行期将调用方法的符号引用转换为直接引用，由于这种引用转换过程具备动态性，因此也称为动态链接。

对应的绑定机制，绑定是一个字段、方法或者类在符号引用被替换为直接引用的过程，这仅仅发生一次。

早期绑定：

晚期绑定：

如果在编译期就确定的具体调用的版本，这个版本在运行是不可变的，这样的方法为**非虚方法**

静态方法、私有方法、final方法、实例构造器、父类方法都是非虚方法。其他都是虚方法。


#### 方法返回地址(Return Address)
存放该方法的PC寄存器的值 调用者的PC计数器的值作为返回地址，既调用该方法的指令的下一条指令的地址，而异常退出的需要同异常表来确定的，栈帧一般不会保存这部分信息。

正常退出与异常退出的区别在于：通过异常完成的退出的不会给他的上层给他的上层调用者产生任何返回值。


#### 一些附加信息
对程序的调试的支持信息


---


### 本地方法接口
本地方法？

一个Native Method就是一个Java调用非Java代码接口（该方法由非Java语言实现比如C）

==标识符native可以和所有的其他的Java标识符连用，但是abstract除外==

使用Native Method的原因 

- 需要与Java环境交互
- 与操作系统交互
- Sum‘s Java


## 本地方法栈（Native Method Stack）
Java虚拟机栈是用于管理Java方法调用，而本地方法栈用于管理本地方法的调用

==线程私有==

当某个线程调用一个本地方法的时候，它就进入了一个全新的并且不再受虚拟机限制的世界。它和虚拟机拥有同样的权限。

在`Hostpot JVM`中 直接将本方法和虚拟机栈合二为一



## 堆（Heap）
一个JVM只存在一个堆内存，堆也是Java内存管理的核心区域。

JVM启动的时候就创建堆，其大小空间也确定，堆是JVM最大的内存（内存大小可调节）

所有的线程都共享堆，在这里可以划分线程的私有的缓冲区（TLAB）

> Java虚拟机规范中，堆可以处于物理不连续的内存空间中，但在逻辑中它被视为连续的

> 所有的对象实例以及数组都应当在运行时分配在堆上

在方法结束后，堆中的对象不会马上的被移除，仅仅就是在垃圾收集的时候才会移除

堆，是GC（垃圾收集器）的执行垃圾回收的重要区域

### 堆内存细分

**为什么要进行分代？**

- 堆内存是虚拟机管理的最大的一块内存区域，也是垃圾回收最为频繁的区域。程序运行时的所有对象实例都在这里保存。内存分代管理就是为对象的内存分配和销毁回收提高效率。试想一下，如果不进行分代划分，所有的对象都放在一起，那些新生的和一些  　　　　生命周期很长的都在一起，每次进行GC的时候，都必须得扫描遍历全部的对象，这个过程是十分耗费资源的，会严重的影响GC的效率。
- 有了内存的分代，会大大提升垃圾回收的效率。新生成的对象在新生代中分配内存，经过几次的垃圾回收依然存活下来的就存放到老年代中，永久代中存放静态属性以及类信息等。新生代中的对象，生命周期最短，相对的来说GC的频率就很高。老年代的生命
- 周期较长，不需要进行频繁的内存回收。永久代基本不需要进行回收操作。当然了，回收机制是可以根据不同代的特点来选择合适的垃圾回收算法的。





现代垃圾收集器大部分基于分代理论设计：

在JDK7  --- > 分为 新生区 + 养老区 + 永久区

在JDK8  --- > 分为 新生区 + 养老区 + 元空间

新生区又分为 Eden区 和 Survivor区


### 设置堆空间大小与OOM

`-Xms`表示堆区的起始内存,等价于`-XX：InitialHeapSize`

`-Xmx`表示堆区的最大内存,等价于`-XX：MaxlHeapSize`

默认情况下的大小

- 初始内存大小： 物理电脑内存大小 / 64

- 最大内存大小： 物理电脑内存大小 / 4

一但堆区中的内存超出`-Xmx`的最大内存会抛出`OutOfMemoryError`

通常`-Xms`于`-Xmx`的值设置相同，目的就是为了在Java垃圾回收机制清理完堆区后不需要重新分割计算堆区的大小，提高性能


`-Xms`用来设置堆空间（年轻代 + 老年代）初始大小

### 年轻代与老年代
Java中的对象可以被划分为2类
- 一类生命周期段，这类的对象创建和消亡迅速。
- 另一类的对象生命周期长，在极端的情况下还能够与JVM的生命周期保持一致
  

  ![image](https://img-blog.csdnimg.cn/20191208160358777.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ0NTQzNTA4,size_16,color_FFFFFF,t_70)
  
  配置新生代与老年代在堆中的占比(默认值2)
  
  `-XX:NewRation=2` 表示新生代占1，老年代占2，新生代占整个堆的1/3
  
  `-XX:NewRation=4`表示新生代占1，老年代占4，新生代占整个堆的1/5

在HotSpot中，Eden空间和另外两个大的Survivor空间的缺省值的比例为8：1：1，可以使用`-XX:SurvivorRatio`调整空间比例 如`-XX:SurvivorRatio=8`，如果不是上面的比例那就是，默认情况下有个默认机制自适应内存需要设置参数`-XX:-UseAdaptiveSizePolicy`关闭自适应内存策略

绝大多数的Java的对象都是在新生代进行销毁的，几乎所有的Java的对象都是在Eden区被new出来的，如果Eden放不下就后移

可以使用参数`-Xmn`来设置新生代的最大的内存（一般不设置）

### 图解分配对象

![image](https://img-blog.csdn.net/20180617112302206?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MDczOTgzMw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

HotSpot实现的复制算法流程如下:

1. 当Eden区满的时候,会触发第一次Minor gc,把还活着的对象拷贝到Survivor From区；当Eden区再次触发Minor gc的时候,会扫描Eden区和From区域,对两个区域进行垃圾回收,经过这次回收后还存活的对象,则直接复制到To区域,并将Eden和From区域清空
1. 当后续Eden又发生Minor gc的时候,会对Eden和To区域进行垃圾回收,存活的对象复制到From区域,并将Eden和To区域清空
1. 部分对象会在From和To区域中复制来复制去,如此交换15次(由JVM参数`MaxTenuringThreshold`决定,这个参数默认是15),最终如果还是存活,就存入到老年代。


==对应s0，s1区总结：复制后有交换，谁空谁是to==

==垃圾回收：频繁在新生代收集，很少在老年代收集，几乎不再元空间/永久区收集==

### 对象分配的特殊情况

![image](https://img-blog.csdnimg.cn/20200825113054479.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FkbWluNzQxYWRtaW4=,size_16,color_FFFFFF,t_70#pic_center)

代码演示：

```java
public class HeapInstanceTest {
    byte[] buffer = new byte[new Random().nextInt(1024 * 200)];

    public static void main(String[] args) {
        ArrayList<HeapInstanceTest> list = new ArrayList<HeapInstanceTest>();
        while (true) {
            list.add(new HeapInstanceTest());
            try {
                Thread.sleep(10);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
//在老年代和新生代都满了，就出现OOM
```

![image](https://img-blog.csdnimg.cn/2020082511550229.gif#pic_center)

### Minor GC、Major GC、Full GC

JVM在进行GC时，并非每次都针对上面三个内存区域（新生代、老年代、方法区）一起回收的，大部分时候回收都是指新生代。

针对hotSpot VM的实现，它里面的GC按照回收区域又分为两大种类型：一种是部分收集（Partial GC），一种是整堆收集（Full GC）

部分收集：不是完整收集整个Java堆的垃圾收集。其中又分为：

- 新生代收集（Minor GC/Young GC）：只是新生代（Eden/S0，S1）的垃圾收集
- 老年代收集（Major GC/o1d GC）：只是老年代的圾收集

==注意：很多时候Major GC 会和 Full GC混淆使用，具体需要分辨是老年代回收还是整堆回收==
- 混合收集（MixedGC）：收集整个新生代以及部分老年代的垃圾收集


整堆收集（FullGC）：收集整个java堆和方法区的垃圾收集

### 年轻代的GC（Minor GC）触发机制

- 当年轻代空间不足时，就会触发MinorGC，这里的年轻代满指的是Eden代满，Survivor满不会引发GC。（每次Minor
- GC会清理年轻代的内存。）
- 因为Java对象大多都具备 朝生夕灭 的特性，所以Minor
- GC非常频繁，一般回收速度也比较快。这一定义既清晰又易于理解。
Minor GC会引发STW，暂停其它用户的线程，等垃圾回收结束，用户线程才恢复运行


### 老年代的GC（Major GC）触发机制

- 指发生在老年代的GC，对象从老年代消失时，我们说 “Major GC” 或 “Full GC” 发生了
- 出现了Major GC，经常会伴随至少一次的Minor GC（但非绝对的，在Parallel Scavenge收集器的收集策略里就有直接进行Major GC的策略选择过程）
- Major GC的速度一般会比MinorGc慢10倍以上，STW的时间更长
- 如果Major GC后，内存还不足，就报OOM了

### Full GC触发机制
**触发FullGC执行的情况有如下五种：**
- 调用System.gc()时，系统建议执行FullGC，但是不必然执行
- 老年代空间不足
- 方法区空间不足
- 通过Minor GC后进入老年代的平均大小大于老年代的可用内存
- 由Eden区、survivor spacee（From Space）区向survivor spacel（To
Space）区复制时，对象大小大于To Space可用内存，则把该对象转存到老年代，且老年代的可用内存小于该对象大小

==Full GC是开发中或者调优需要避免的，这样暂停时间就会短一些==

代码示例：
```java
public class GCTest {
    public static void main(String[] args) {
        int i = 0;
        try {
            List<String> list = new ArrayList<>();
            String a = "atguigu";
            while(true) {
                list.add(a);
                a = a + a;
                i++;
            }
        }catch (Exception e) {
            e.getStackTrace();
        }
    }
}
```
> 触发OOM的时候，一定是进行了一次Full GC，因为只有在老年代空间不足时候，才会爆出OOM异常

### 内存分配策略
如果对象在Eden出生并经过第一次Minor GC后仍然存活，并且能被Survivor容纳的话，将被移动到survivor空间中，并将对象年龄设为1。对象在survivor区中每熬过一次MinorGC，年龄就增加1岁，当它的年龄增加到一定程度（默认为15岁，其实每个JVM、每个GC都有所不同）时，就会被晋升到老年代

> 对象晋升老年代的年龄阀值，可以通过选项-xx:MaxTenuringThreshold来设置

不同的年龄段对象分配原则：
- 优先分配到Eden
- 大对象直接分配到老年代
- 长期存活的对象分配到老年代 （**尽量避免程序中出现过多的大对象**）
- 动态对象年龄判断  
    **如果survivor区中相同年龄的所有对象大小的总和大于Survivor空间的一半，年龄大于或等于该年龄的对象可以直接进入老年代，无须等到MaxTenuringThreshold 中要求的年龄。**
- 空间分配担保： -Xx:HandlePromotionFailure

测试:大对象直接进入老年代
```java
/** 测试：大对象直接进入老年代
 * -Xms60m -Xmx60m -XX:NewRatio=2 -XX:SurvivorRatio=8 -XX:+PrintGCDetails
 */
public class YoungOldAreaTest {
    // 新生代 20m ，Eden 16m， s0 2m， s1 2m
    // 老年代 40m
    public static void main(String[] args) throws InterruptedException {
        //Eden 区无法存放buffer  晋升老年代
            byte[] buffer = new byte[1024 * 1024 * 20];//20m
    }
}
```

### 为对象分配内存：TLAB（堆当中的线程私有缓存区域）

Threadd Local Allocation Buffer

**堆空间都是共享的么？**

不一定，因为还有TLAB这个概念，在堆中划分出一块区域，为每个线程所独占

为什么有TLAB？
- TLAB：Thread Local Allocation Buffer，也就是为每个线程单独分配了一个缓冲区
- 堆区是线程共享区域，任何线程都可以访问到堆区中的共享数据
- 由于对象实例的创建在JVM中非常频繁，因此在并发环境下从堆区中划分内存空间是线程不安全的
- 为避免多个线程操作同一地址，需要使用加锁等机制，进而影响分配速度。

**什么是TLAB?**

- 从内存模型而不是垃圾收集的角度，对Eden区域继续进行划分，**JVM为每个线程分配了一个私有缓存区域，它包含在Eden空间内**。
- 多线程同时分配内存时，使用TLAB可以避免一系列的**非线程安全问题**，同时还能够提升内存分配的吞吐量，因此我们可以将这种内存分配方式称之为**快速分配策略**。
- 据我所知所有OpenJDK衍生出来的JVM都提供了TLAB的设计。

![image](https://img-blog.csdnimg.cn/20200826203314436.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FkbWluNzQxYWRtaW4=,size_16,color_FFFFFF,t_70#pic_center)

**TLAB的再说明：**

- 尽管不是所有的对象实例都能够在TLAB中成功分配内存，但**JVM确实是将TLAB作为内存分配的首选**。
- 在程序中，开发人员可以通过选项`-Xx:UseTLAB`设置是否开启TLAB空间。(默认是开启的)
- 默认情况下，TLAB空间的内存非常小，==仅占有整个Eden空间的1%==，当然我们可以通过选项“-Xx:TLABWasteTargetPercent”设置TLAB空间所占用Eden空间的百分比大小。
- 一旦对象在TLAB空间分配内存失败时，JVM就会尝试着通过使用==加锁机制==确保数据操作的原子性，从而直接在Eden空间中分配内存。


#### TLAB对象分配过程
> 对象首先是通过TLAB开辟空间，如果不能放入，那么需要通过Eden来进行分配


![image](https://img-blog.csdnimg.cn/20200826204208458.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FkbWluNzQxYWRtaW4=,size_16,color_FFFFFF,t_70#pic_center)

### 小结堆空间的参数设置

- -XX：+PrintFlagsInitial：查看所有的参数的默认初始值
- -XX：+PrintFlagsFinal：查看所有的参数的最终值（可能会存在修改，不再是初始值）(修改过的那行，=号前面有:)
  
    > 具体查看某个参数的指令：jps:查看当前运行中的进程  jinfo -flag xxx  进程id： 查看xx指令
- -Xms：初始堆空间内存（默认为物理内存的1/64）
- -Xmx：最大堆空间内存（默认为物理内存的1/4）
- -Xmn：设置新生代的大小。（初始值及最大值）
- -XX:NewRatio：配置新生代与老年代在堆结构的占比
- -XX:SurvivorRatio：设置新生代中Eden和S0/S1空间的比例
- -XX:MaxTenuringThreshold：设置新生代垃圾的最大年龄
- -XX：+PrintGCDetails：输出详细的GC处理日志
    * 打印gc简要信息：①-Xx：+PrintGC ② - verbose:gc
- -XX:HandlePromotionFalilure：是否设置空间分配担保
- -XX:+DoEscapeAnalysis 开启逃逸分析(默认开启)
- -xx：+EliminateAllocations：开启了标量替换（默认打开），允许将对象打散分配在栈上


### 堆是分配对象的唯一选择吗?

在《深入理解Java虚拟机》中关于Java堆内存有这样一段描述：
> 随着JIT编译期的发展与逃逸分析技术逐渐成熟，栈上分配、标量替换优化技术将会导致一些微妙的变化，所有的对象都分配到堆上也渐渐变得不那么“绝对”了。

但是，有一种特殊情况，那就是如果经过**逃逸分析（Escape Analysis**）后发现，**一个对象并没有逃逸出方法的话，那么就可能被优化成栈上分配。这样就无需在堆上分配内存，也无须进行垃圾回收了。** 这也是最常见的堆外存储技术。

> 创新的GCIH（GC invisible heap）技术实现off-heap，将生命周期较长的Java对象从heap中移至heap外，并且GC不能管理GCIH内部的Java对象，以此达到降低GC的回收频率和提升GC的回收效率的目的

### 逃逸分析
如何将堆上的对象分配到栈，需要使用逃逸分析手段。
这是一种可以有效减少Java程序中同步负载和内存堆分配压力的跨函数全局数据流分析算法。通过逃逸分析，Java Hotspot编译器能够分析出一个新的对象的引用的使用范围从而决定是否要将这个对象分配到堆上。逃逸分析的基本行为就是分析对象动态作用域：

- 当一个对象在方法中被定义后，对象只在方法内部使用，则认为没有发生逃逸。
- 当一个对象在方法中被定义后，它被外部方法所引用，则认为发生逃逸。例如作为调用参数传递到其他地方中。
- 
> 如何快速的判断是否发生了逃逸分析，就看new的对象实体是否有可能在方法外被调用

```java
public void my_method() {
    V v = new V();
    // use v
    // ....
    v = null;
}
//没有发生逃逸的对象，则可以分配到栈上，随着方法执行的结束，栈空间就被移除，每个栈里面包含了很多栈帧，也就是发生逃逸分析
```
逃逸分析代码演示：
```java
    public static StringBuffer createStringBuffer(String s1, String s2) {
        StringBuffer sb = new StringBuffer();
        sb.append(s1);
        sb.append(s2);
        return sb; //sb返回外部发生逃逸
    }
```

**解决上面的问题要StringBuffer sb不发生逃逸，可以这样写**

```java
    public static String createStringBuffer(String s1, String s2) {
        StringBuffer sb = new StringBuffer();
        sb.append(s1);
        sb.append(s2);
        return sb.toString();
    }
```

代码演示：
```java
/**
 * 逃逸分析
 *
 *  如何快速的判断是否发生了逃逸分析，就看“new的对象实体”是否有可能在方法外被调用。
 *  如果当前的obj引用声明为static的？仍然会发生逃逸。
 */
public class EscapeAnalysis {

    public EscapeAnalysis obj;

    /*
    方法返回EscapeAnalysis对象，发生逃逸
     */
    public EscapeAnalysis getInstance(){
        return obj == null? new EscapeAnalysis() : obj;
    }
    /*
    为成员属性赋值，发生逃逸
     */
    public void setObj(){
        this.obj = new EscapeAnalysis();
    }
    //思考：如果当前的obj引用声明为static的？仍然会发生逃逸。

    /*
    对象的作用域仅在当前方法中有效，没有发生逃逸
     */
    public void useEscapeAnalysis(){
        EscapeAnalysis e = new EscapeAnalysis();
    }
    /*
    引用成员变量的值，发生逃逸
     */
    public void useEscapeAnalysis1(){
        EscapeAnalysis e = getInstance();
        //getInstance().xxx()同样会发生逃逸
    }
}
```

> 如何快速的判断是否发生了逃逸分析，就看==new的对象实体==是否有可能在方法外被调用

在JDK 1.7 版本之后，HotSpot中默认就已经开启了逃逸分析
如果使用的是较早的版本，开发人员则可以通过：

- 选项`-xx：+DoEscapeAnalysis`显式开启逃逸分析
- 通过选项`-xx：+PrintEscapeAnalysis`查看逃逸分析的筛选结果

> **开发中能使用局部变量的，就不要使用在方法外定义**

### 基于逃逸分析代码优化
使用逃逸分析，编译器可以对代码做如下优化：

- ==栈上分配==：将堆分配转化为栈分配。如果一个对象在子程序中被分配，要使指向该对象的指针永远不会发生逃逸，对象可能是栈上分配的候选，而不是堆上分配
   *  JIT编译器在编译期间根据逃逸分析的结果，发现如果一个对象并没有逃逸出方法的话，就可能被优化成栈上分配。分配完成后，继续在调用栈内执行，最后线程结束，栈空间被回收，局部变量对象也被回收。这样就无须进行垃圾回收了。
   
   

- ==同步省略==：如果一个对象被发现只有一个线程被访问到，那么对于这个对象的操作可以不考虑同步。
- ==分离对象或标量替换==：有的对象可能不需要作为一个连续的内存结构存在也可以被访问到，那么对象的部分（或全部）可以不存储在内存，而是存储在CPU寄存器中。

#### 栈上分配

常见场景：==在逃逸分析中，已经说明了。分别是给成员变量赋值、方法返回值、实例引用传递。==

Code Demo:
```java
public class StackAllocation {
    public static void main(String[] args) throws InterruptedException {
        long start = System.currentTimeMillis();
        for (int i = 0; i < 100000000; i++) {
            alloc();
        }
        long end = System.currentTimeMillis();
        System.out.println("花费的时间为：" + (end - start) + " ms");

        // 为了方便查看堆内存中对象个数，线程sleep
        Thread.sleep(10000000);
    }

    private static void alloc() {
        User user = new User();
    }

    static class User{

    }
}
```
分析：

 默认开启逃逸分析
花费的时间为：3 ms

User对象 ，没能发生逃逸，他们存储在栈中，随着栈的销毁而消失，(不触发GC)

// 关闭逃逸分析

` -Xmx1G -Xms1G -XX:+DoEscapeAnalysis -XX:+PrintGCDetails`
花费的时间为：590 ms

User对象 ，直接发生逃逸，储存在堆空间中(触发GC)

#### 同步省略

==线程同步的代价是相当高的，同步的后果是降低并发性和(降低)性能。==

> 在动态编译同步块的时候，JIT编译器可以借助逃逸分析来判断**同步块所使用的锁对象是否只能够被一个线程访问而没有被发布到其他线程**。如果没有，那么JIT编译器在编译这个同步块的时候就会取消对这部分代码的同步。这样就能大大提高并发性和性能。这个取消同步的过程就叫同步省略，也叫**锁消除**。

Code Demo:
```java
public void f() {
    Object hellis = new Object();
    //实际中这个锁没用 ，但是还是会影响性能
    synchronized(hellis) {
        System.out.println(hellis);
    }
}
```
代码中对hellis这个对象加锁，但是hellis对象的生命周期只在f()方法中，并不会被其他线程所访问到，所以在JIT编译阶段就会被优化掉，优化成：

```java
public void f() {
    Object hellis = new Object();
	System.out.println(hellis);
}
```



#### 分离对象或标量替换

- 标量（scalar）是指一个无法再分解成更小的数据的数据。Java中的原始数据类型就是标量。
- 相对的，那些还可以分解的数据叫做聚合量（Aggregate），Java中的对象就是聚合量，因为他可以分解成其他聚合量和标量。

> 在JIT阶段，如果经过逃逸分析，发现一个对象不会被外界访问的话，那么经过JIT优化，就会把这个对象拆解成若干个其中包含的若干个成员变量来代替。这个过程就是标量替换。

```java
public static void main(String args[]) {
    alloc();
}
class Point {
    private int x;//标量
    private int y;
}
private static void alloc() {
    Point point = new Point(1,2);// 聚合量
    System.out.println("point.x" + point.x + ";point.y" + point.y);
}
```
以上代码，经过标量替换后，就会变成
```java
private static void alloc() {
    int x = 1;
    int y = 2;
    System.out.println("point.x = " + x + "; point.y=" + y);
}
```

**可以看到，Point这个聚合量经过逃逸分析后，发现他并没有逃逸，就被替换成两个聚合量了。那么标量替换有什么好处呢？**

==就是可以大大减少堆内存的占用。因为一旦不需要创建对象了，那么就不再需要分配堆内存了。 标量替换为栈上分配提供了很好的基础。==

上述代码在主函数中进行了1亿次alloc。调用进行对象创建，由于User对象实例需要占据约16字节的空间，因此累计分配空间达到将近1.5GB。如果堆空间小于这个值，就必然会发生GC。使用如下参数运行上述代码：

`-server -Xmx10m -Xms10m -XX:+DoEscapeAnalysis -XX:+PrintGC -XX:+EliminateAllocations`

- 参数-server：启动Server模式，因为在server模式下，才可以启用逃逸分析。
- 参数-XX:+DoEscapeAnalysis：启用逃逸分析
- 参数-Xmx10m：指定了堆空间最大为10MB
- 参数-XX:+PrintGC：将打印Gc日志。
- 参数-xx：+EliminateAllocations：开启了标量替换（默认打开），允许将对象打散分配在栈上，比如对象拥有id和name两个字段，那么这两个字段将会被视为两个独立的局部变量进行分配

```java
/**
 * 标量替换测试
 *  -Xmx100m -Xms100m -XX:+DoEscapeAnalysis -XX:+PrintGC -XX:-EliminateAllocations
 */
public class ScalarReplace {
    public static class User {
        public int id;//标量（无法再分解成更小的数据）
        public String name;//聚合量（String还可以分解为char数组）
    }

    public static void alloc() {
        User u = new User();//未发生逃逸
        u.id = 5;
        u.name = "www.atguigu.com";
    }

    public static void main(String[] args) {
        long start = System.currentTimeMillis();
        for (int i = 0; i < 10000000; i++) {
            alloc();
        }
        long end = System.currentTimeMillis();
        System.out.println("花费的时间为： " + (end - start) + " ms");
    }
}
// 即使开启了逃逸分析但是没有开启标量替换，他还是会在堆上分配
```

==即使开启了逃逸分析但是没有开启标量替换，他还是会在堆上分配==