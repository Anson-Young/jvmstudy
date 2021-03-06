1、在Java代码中，类型(Class、Interface、Enum)的加载、连接、初始化过程都是在程序运行期间完成的。
其中连接又分为验证、准备、解析三个阶段。解析阶段是将符号引用转化为直接引用。<br/>
符号引用与直接引用
   * 例1：
     ```java
     public interface Intf {
         String str = "abcde";
         int ival = new Random().nextInt();
     }
     
     public class T {
         public static int tint = Intf.ival;
     }
     ```
     T类中静态变量tint的值在编译期间不确定，只能在运行期间确定，因此在编译期间先用符号引用指向Intf的ival字段，等运行的时候再将符号引用解析为直接引用。
   * 例2：
     ```
     public static void main(String[] args) {
         new ArrayList<>();
     }
     ```
     ArrayList在类生命周期的解析阶段之前，这就只是个符号，即符号引用。在解析过程中，JVM会去找ArrayList是否被加载，如果未加载会先加载ArrayList，然后返回ArrayList的引用，这就是直接引用。

2、类的主动使用与被动使用。<br/>
   类在被动使用时不会被初始化！只有当程序访问的静态变量或静态方法确实在当前类或当前接口中定义时，才可以认为是对类或接口的主动使用：
```java
public class MyTest {
     public static void main(String[] args) {
          System.out.println(Child.str);	// parent static block
                                            // hello world
     }
}

class Parent {
     public static String str = "hello world";

     static {
          System.out.println("parent static block");
     }
}

class Child extends Parent {
     static {
          System.out.println("child static block");
     }
}
```
原因：Child类为被动使用，虽然是用Child访问的str字段。对于静态字段来说，只有直接定义了该字段的类才会被初始化。(但是Child类会被加载)

3、可以在编译期确定值的常量称为编译期常量，在编译阶段会存入到调用这个常量的方法所在类的常量池中。本质上，调用类并没有直接引用到定义该常量的类，因此并不会触发定义常量的类的初始化，甚至这个类都没有被加载，可以将定义该常量的类的class文件删除：
```java
public class MyTest {
    public static void main(String[] args) {
        System.out.println(MyParent.str);	// hello world(MyParent里的静态代码块不会被执行)
    }
}

class MyParent {
     public static final String str = "hello world";

     static {
          System.out.println("MyParent static block");
     }
}
```
4、初始化一个类的子类，会先初始化它的父类，但是不会初始化它实现的接口；初始化一个接口，也并不会初始化它继承的父接口。
但是两种情况无论是继承父类、父接口还是实现的接口都会被加载(注意加载和初始化是两个概念)，
如果将实现的接口或继承的父接口的字节码删除，再运行会报NoClassDefFoundError(编译正常，运行时找不到这个类)：
```java
public class MyTest {
     public static void main(String[] args) {
          System.out.println(MyChild.i);            // MyParent
                                                    // a random number
          System.out.println(AnotherInterface.a);	// a random number
     }
}

interface MyInterface {
     Object o = new Object() {
          {
               System.out.println("MyInterface");
          }
     };
}

class MyParent {
     static {
          System.out.println("MyParent");
     }
}

class MyChild extends MyParent implements MyInterface {
     public static final int i = new Random().nextInt(100);
}

interface AnotherInterface extends MyInterface {
     int a = (int) (Math.random() * 100);
}
```
5、类在加载之后的连接准备阶段会为类的静态变量(final常量除外)赋默认值，在初始化阶段才会按顺序为类的静态变量执行初始化语句：
```java
public class MyTest {
     public static void main(String[] args) {
          Singleton singleton = Singleton.getInstance();
          System.out.println(Singleton.a);	// 1
          System.out.println(Singleton.b);	// 0
     }
}

class Singleton {
     public static int a;

     private static Singleton singleton = new Singleton();

     private Singleton() {
          a++;
          b++;
          System.out.println(a);	// 1
          System.out.println(b);	// 1
     }

     public static int b = 0;


     public static Singleton getInstance() {
          return singleton;
     }
}
```
6、可以通过虚拟机参数-XX:+TraceClassLoading来观察有哪些类被加载

7、加载只是类加载的其中一个阶段，类加载的全过程包括加载、连接(验证、准备、解析)、初始化这几个阶段。
加载阶段最终会在虚拟机内存中生成代表这个类的Class对象。

8、ClassNotFoundException与NoClassDefFoundError区别：<br/>
ClassNotFoundException间接继承自Exception，为受检异常，程序当中可以捕获，抛出此异常的常见场景是
Class.forName("xxx.yyy.zzz.className");在运行期找不到需要加载的类，也就是在外围类初始化完毕后运行过程中发生的异常。
NoClassDefFoundError继承自LinkageError，通过名字可以看出是在类加载的连接阶段发生的错误。具体发生在连接阶段所包含的3个阶段中的最后一个阶段——解析阶段，如果在解析阶段找不到相应的类就会报NoClassDefFoundError。例如上面4的叙述内容，程序正常编译后删除所实现或继承接口的class文件，然后在运行时先进入子类加载的加载阶段，在加载的过程中进行解析，解析过程中要将所实现或继承接口的符号引用替换为直接引用，在替换时发现找不到所实现或继承的接口，就会报NoClassDefFoundError。注意：加载阶段与连接阶段是按照先后顺序开始，并不是按照先后顺序进行或完成，这两个阶段是交叉进行的，加载阶段尚未完成，连接阶段可能已经开始。
另外，NoClassDefFoundError往往是在解析的时候发现需要加载其他类，而这个类又不存在的时候发生的，亦即在将符号引用转化为直接引用时发生的。
注意：符号引用为“引用”，是一个指向具体值的指针，如果类中有诸如private SomeClass someClass;的代码，程序在编译时不会将其转化为符号引用，
因为这仅仅是一个声明，并没有将其赋值，也就不存在指向具体值的指针，这种情况在类的加载、连接的过程中并不会解析SomeClass，也就不会加载它，
因此就不会报告NoClassDefFoundError错误。
还有一种情况是在类的方法中存在SomeClass的引用，比如new SomeClass();，外围类在编译的时候会将其转化为符号引用，在连接的时候也会将其转化为
直接引用，如果SomeClass的class文件不存在，虚拟机在解析的时候不会报告NoClassDefFoundError，直到SomeClass所在的方法被调用，这种情况称之为
Lazy implementation of JVM，这是由JVM的具体实现所决定的，不同的JVM可能有不同的实现，即其他实现可能在解析阶段就报NoClassDefFoundError。<br/>
参考链接：https://blog.csdn.net/qq_36781505/article/details/89053304

9、使用java.lang.reflect包的方法对类进行反射调用的时候，如果类没有进行初始化，则需要先触发其初始化。例如Class.forName("xxx.yyy.zzz.className");
而调用ClassLoader类的loadClass方法加载一个类，并不是对类的主动使用，不会导致类的初始化

10、对于数组实例来说，其类型是由JVM在运行期动态生成，表示为[Lxxx.yyy.zzz.className这种形式。动态生成的类型，其父类型是Object。

11、虽然说数组类的Class对象不是由类加载器创建出来的，但是通过Class.getClassLoader()返回的类加载器与其包含的元素类型的类加载器是一样的(对于元素类型是引用类型的数组)

12、类加载器的双亲委托机制：
   * 自底向上检查类是否已经加载；
   * 自顶向下尝试加载类

13、类加载器的命名空间：
   * 每个类加载器都有自己的命名空间，命名空间由该加载器及所有父加载器所加载的类组成；
理解：子加载器可以看到父加载器加载的类，而父加载器看不到子加载器加载的类(利用到下面14的MyClassLoader)：
     ```java
     public class ClassLoaderTest {
         public static void main(String[] args) throws Exception {
             MyClassLoader classLoader = new MyClassLoader();
             classLoader.setPath("F:\\other\\backup\\jvm\\");
             Class<?> clazz = classLoader.loadClass("com.yc.demo.MySample");
             clazz.newInstance();
         }
     }
     
     class MySample {
         public MySample() {
             System.out.println("MySample is loaded by: " + this.getClass().getClassLoader());
             // 正常执行，MySample由子加载器MyClassLoader加载，MyCat由父加载器AppClassLoader加载
             // 子加载器可以访问到父加载器加载的类
             System.out.println("from MySample: " + MyCat.class);
             new MyCat();
         }
     }
     
     class MyCat {
         public MyCat() {
             System.out.println("MyCat is loaded by: " + this.getClass().getClassLoader());
             // 报告NoClassDefFoundError，MyCat由父加载器AppClassLoader加载，MySample由子加载器
             // MyClassLoader加载，父加载器无法访问子加载器加载的类
             System.out.println("from MyCat: " + MySample.class);
         }
     }
     ```
	 注：将MySample的class文件移至classLoader所指定path路径下，程序运行报告NoClassDefFoundError(MySample)
	     因为MySample由MyClassLoader加载，在加载MyCat的时候委托给了父加载器AppClassLoader加载，加载成功，
	     在加载MySample的时候发现在自己的命名空间里找不到这个类（MySample是由子加载器MyClassLoader加载）。 
     
   * 在同一个命名空间中，不会出现类的完整名字（包括类的包名）相同的两个类；
   * 在不同的命名空间中，有可能会出现类的完整名字（包括类的包名）相同的两个类

14、自定义类加载器应该继承自ClassLoader类，并实现ClassLoader的findClass方法。加载类的入口方法是ClassLoader类的loadClass(String name)，参数name往往是类的全限定名，
   * 如果类已经被加载，则直接返回类的Class对象；
   * 自定义类加载器委托父加载器加载类，如果加载成功，返回类的Class对象；
   * 自定义类加载器调用自己实现的findClass方法加载类的字节码生成Class对象。

举例：
   * 例1：
     ```java
     public class MyClassLoader extends ClassLoader {
         private String path;
         public void setPath(String path) {
             this.path = path;
         }
     
         public MyClassLoader() {}
         public MyClassLoader(ClassLoader parent) {
             super(parent);
         }
     
         @Override
         // 如果AppClassLoader能加载到className所指定的class，则此方法不执行
         // 将className所指定的class字节码文件移至path路径下，则会执行此方法
         protected Class<?> findClass(String className) throws ClassNotFoundException {
             System.out.println("findClass invoked...");
             System.out.println("loading class: " + className);
             byte[] data = loadClassData(className);
             return super.defineClass(className, data, 0, data.length);
         }
     
         private byte[] loadClassData(String className) {
             File file = new File(this.path + className.replace(".", "/") + ".class");
             byte[] b = new byte[(int) file.length()];
             FileInputStream inputStream = null;
     
             try {
                 inputStream = new FileInputStream(file);
                 inputStream.read(b);
             } catch (Exception e) {
                 e.printStackTrace();
             } finally {
                 if (inputStream != null) {
                     try {  
                         inputStream.close();
                     } catch (IOException e) {
                         e.printStackTrace();
                     }
                 }
             }
             return b;
         }
     
         public static void main(String[] args) throws Exception {
             MyClassLoader classLoader1 = new MyClassLoader();	            // 默认以系统类加载器AppClassLoader为父加载器
             classLoader1.setPath("E:/others/test/out/production/demo/");	// 如果AppClassLoader能加载到类，则此设置无效
             Class<?> clazz1 = classLoader1.loadClass("com.yc.demo.classloader.Dummy");
             System.out.println("class: " + clazz1.hashCode());
             System.out.println("object: " + clazz1.newInstance());
             System.out.println();
             MyClassLoader classLoader2 = new MyClassLoader();
             classLoader2.setPath("E:/others/test/out/production/demo/");
             Class<?> clazz2 = classLoader2.loadClass("com.yc.demo.classloader.Dummy");
             System.out.println("class: " + clazz2.hashCode());
             System.out.println("object: " + clazz2.newInstance());
         }
     }
     ```
	 注：
	    * 若Dummy就在当前classpath下，则classLoader1委托父加载器AppClassLoader加载Dummy，加载出来的Class对象
	      存在于父加载器AppClassLoader的缓存区域；classLoader2尝试加载Dummy在执行到ClassLoader的406行时，
	      并不会直接返回Dummy的Class对象，因为在classLoader2的缓存区域里不存在Dummy的Class对象
	      （甚至在classLoader1的缓存区域里也不存在），因此classLoader2还会继续委托父加载器
	      AppClassLoader来加载Dummy，而此时AppClassLoader发现已经加载过Dummy，因此会直接返回
	      Dummy的Class对象，所以两次输出的hashcode值一样 
          
        * 若将Dummy的class文件移至MyClassLoader所指定的path路径下，则Dummy将由自定义类加载器加载，由于
	      classLoader1和classLoader2属不同的命名空间，因此加载出来的Class对象不同，所以两次输出的hashcode值
	      不一样
     
   * 例2：
     ```java
     public class ClassLoaderTest {
         public static void main(String[] args) {
             MyClassLoader classLoader1 = new MyClassLoader();
             classLoader1.setPath("F:/other/backup/jvm/");
             MyClassLoader classLoader2 = new MyClassLoader();
             classLoader2.setPath("F:/other/backup/jvm/");
             Class<?> clazz1 = classLoader1.loadClass("com.yc.demo.MyPerson");
             Class<?> clazz2 = classLoader2.loadClass("com.yc.demo.MyPerson");
     
             System.out.println(clazz1 == clazz2);
     
             Object obj1 = clazz1.newInstance();
             Object obj2 = clazz2.newInstance();
     
             Method method = clazz1.getDeclaredMethod("setMyPerson", Object.class);
             method.invoke(obj1, obj2);
         }
     }
     
     class MyPerson {
         private MyPerson myPerson;
         public void setMyPerson(Object object) {
     		 this.myPerson = (MyPerson) object;
         }
     }
     ```
	 注：
	   * MyPerson的class文件在当前classpath下，虽然classLoader1与classLoader2是两个不同的类加载器实例，
	     但是它们在加载MyPerson的时候都会委托父加载器AppClassLoader去加载，因此加载出来的Class对象是相同的。
	     System.out.println(clazz1 == clazz2);输出结果为true，程序正常执行
	   * 将MyPerson的class文件从当前classpath移至MyClassLoader所指定的path路径下，由于classLoader1与
	     classLoader2属于不同的命名空间，因此加载出来的MyPerson的Class对象不同，
	     各自实例化出来的对象也在不同的命名空间，不能相互转化，因此
	     System.out.println(clazz1 == clazz2);输出结果为false，程序执行报告异常：
         ```
         Exception in thread "main" java.lang.reflect.InvocationTargetException
             at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
             at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
             at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
             at java.lang.reflect.Method.invoke(Method.java:498)
             at com.yc.demo.ClassLoaderTest02.main(ClassLoaderTest02.java:21)
         Caused by: java.lang.ClassCastException: com.yc.demo.MyPerson cannot be cast to com.yc.demo.MyPerson
             at com.yc.demo.MyPerson.setMyPerson(MyPerson.java:7)
             ... 5 more
总结：在运行期，一个Java类是由该类的完全限定名（binary name，二进制名）和用于加载该类的定义类加载器（defining loader）所共同
决定的。如果同样名字（即相同的完全限定名）的类是由两个不同的类加载器所加载，那么这些类就是不同的，即便.class文件的字节码
完全一样，并且从相同的位置加载亦如此。

15、若有一个类加载器能够成功加载Test类，那么这个类加载器被称为定义类加载器，
所有能成功返回Class对象引用的类加载器（包括定义类加载器）都被称为初始类加载器。（了解）

16、每个类都会尝试使用自己的类加载器去加载依赖的类：
```java
public class ClassLoaderTest { 
    public static void main(String[] args) throws Exception { 
        MyClassLoader classLoader = new MyClassLoader();
        classLoader.setPath("E:/others/test/out/production/demo/");
        Class<?> clazz = classLoader.loadClass("com.yc.demo.classloader.MySample");
        clazz.newInstance();
    }
}

class MySample { 
    public MySample() { 
        System.out.println("MySample is loaded by: " + this.getClass().getClassLoader());
        new MyCat();	// MySample会使用自己的类加载器去加载MyCat 
    }
}

class MyCat { 
    public MyCat() { 
        System.out.println("MyCat is loaded by: " + this.getClass().getClassLoader()); 
    }
}
```
注：
   * 将MySample和MyCat的class文件移至classLoader所设置的path路径下，最终的打印结果是：
	 ```
	 findClass invoked...
	 loading class: com.yc.demo.classloader.MySample
	 MySample is loaded by: com.yc.demo.classloader.MyClassLoader@4554617c
	 findClass invoked...
	 loading class: com.yc.demo.classloader.MyCat
	 MyCat is loaded by: com.yc.demo.classloader.MyClassLoader@4554617c
	 ```
     即MySample和MyCat的类加载器均是MyClassLoader 
	 
   * 若仅仅是将MySample的class文件保留在原来的classpath下，则程序运行会报NoClassDefFoundError，
	 因为加载MySample的是AppClassLoader，AppClassLoader在当前classpath里加载不到MyCat。
	
   * 若仅仅是将MyCat的class文件保留在原来的classpath下，则程序正常执行，打印结果：
	 ```	
	 findClass invoked: com.yc.demo.MySample
	 MySample is loaded by: com.yc.demo.MyClassLoader@74a14482
	 MyCat is loaded by: sun.misc.Launcher$AppClassLoader@18b4aac2
	 ```
	 MyClassLoader加载MySample，在加载MyCat的时候委托给了父加载器AppClassLoader加载

17、类加载器的双亲委托模型的好处：
   * 可以确保Java核心库的类型安全：所有的Java应用都至少会引用java.lang.Object类，也就是说在运行期，
java.lang.Object这个类会被加载到JVM中，如果这个过程是由Java应用自己的类加载器所完成的，那么很可能就会在
JVM中存在多个版本的java.lang.Object类，而且这些类之间还是不兼容且相互不可见的（正是命名空间在发挥作用）
借助于双亲委托机制，Java核心类库中的类的加载工作都是由启动类加载器来统一完成，从而确保了Java应用所使用
的都是同一个版本的Java核心类库，它们之间是相互兼容的。
   * 可以确保Java核心类库所提供的类不会被自定义的类所替代。例如自己也定义了一个java.lang.Object，默认情况下
这个自定义的类由AppClassLoader加载，在双亲委托模型下，最终这个类会委托给根类加载器（Bootstrap）加载，
而根类加载器已经加载过一个java.lang.Object（JRE\lib\rt.jar），
因此会直接返回这个jre所自带的类，自定义的类无效。
   * 不同的类加载器可以为相同名称（binary name）的类创建额外的命名空间，相同名称的类可以并存在JVM中，
只需要用不同的类加载器来加载它们即可，不同类加载器所加载的类之间是不兼容的，这就相当于在JVM内部创建了
一个又一个相互隔离的Java类空间，这类技术在很多框架中都得到了实际应用。
	 
18、扩展类加载器在加载类时是在其被设定的扫描路径下（默认为java.ext.dirs）寻找包含这个类的jar包，而不是直接寻找这个类的class文件：
```java
public class MyTest { 
    public static void main(String[] args) { 
        System.out.println(MyTest.class.getClassLoader()); 
    }
}
```
若要输出ExtClassLoader，除了要在当前classpath下执行java -Djava.ext.dirs=./ 将加载路径改为当前路径外，还要在当前classpath下将MyTest.class文件达成jar包。

19、内建于JVM中的启动类加载器会加载java.lang.ClassLoader以及其他的Java平台类（java.lang.Object, java.lang.String...），
    当JVM启动时，一块特殊的机器码会运行，它会加载扩展类加载器与系统类加载器，这块特殊的机器码叫做启动类加载器（Bootstrap）。
   * 启动类加载器并不是Java类，而其他的类加载器则都是Java类，
   * 启动类加载器是特定于平台的机器指令，它负责开启整个加载过程。
   * 所有类加载器（除了启动类加载器）都被实现为Java类。不过，总归要有一个组件来加载第一个Java类加载器，从而让整个加载过程能够顺利进行下去，加载第一个纯Java类加载器就是启动类加载器的职责。
   * 启动类加载器还会负责加载供JRE正常运行所需要的基本组件，这包括java.util与java.lang包中的类等等。

20、通过修改java.system.class.loader系统属性（往往是一个类的全限定名）可以设定系统类加载器，而原来的系统类加载器AppClassLoader会作为
默认系统类加载器（default system class loader）而成为所设定的系统类加载器的父加载器，且所设定的类加载器被默认系统类加载器加载。
详细可以参照ClassLoader类的getSystemClassLoader方法的文档说明以及源码！

21、当前类加载器（Current ClassLoader）与线程上下文类加载器（Context ClassLoader）
   * 当前类加载器（Current ClassLoader）：每个类都会使用自己的类加载器（即加载自身的类加载器）来尝试去加载其他类（指的是所依赖的类），
	 如果ClassX引用了ClassY，那么ClassX的类加载器就会尝试加载ClassY（前提是ClassY尚未被加载ClassX的类加载器所属的命名空间里的加载器加载过）
   * 线程上下文类加载器（Context ClassLoader）：线程上下文类加载器是从JDK1.2开始引入的，类Thread中的getContextClassLoader()与setContextClassLoader(ClassLoader cl)
     分别用来获取和设置线程上下文类加载器。线程上下文类加载器就是当前线程的Current ClassLoader。如果没有通过setContextClassLoader(ClassLoader cl)进行设置的话，线程将继承其父线程的上下文类加载器。
    Java应用运行时的初始线程上下文类加载器是系统类加载器。在线程中运行的代码可以通过该类加载器来加载类与资源。
   * 线程上下文类加载器的重要性：父ClassLoader可以使用当前线程Thread.currentThread().getContextClassLoader()所制定的classLoader所加载的类。
	 这就改变了父ClassLoader不能使用子ClassLoader或者是其他没有直接父子关系的ClassLoader所加载的类的情况，即改变了双亲委托模型。<br/>
	 典型应用是**SPI**（Service Provicer Interface）在双亲委托模型下，类加载器是由下至上的，即下层的类加载器会委托上层进行加载。但是对于SPI来说，有些接口是Java核心库所提供的，
     而Java核心库是由启动类加载器来加载的，而这些接口的实现却来自于不同的jar包（厂商提供），Java的启动类加载器是不会加载其他来源的
     jar包，这样传统的双亲委托模型就无法满足SPI的要求。而通过给当前线程设置上下文类加载器，就可以有设置的上下文类加载器来实现对于
     接口实现类的加载。例：
	 ```java
	 import java.sql.Driver;
	 import java.util.ServiceLoader;
	 public class MyTest {
	     public static void main(String[] args) {
	         ServiceLoader<Driver> loader = ServiceLoader.load(Driver.class);
	         for (Driver driver: loader) {
	             System.out.println("class: " + driver.getClass() + ", loader: " + driver.getClass.getClassLoader());
	             // driver: class com.mysql.cj.jdbc.Driver, loader: sun.misc.Launcher$AppClassLoader@73d16e93
	         }
	         System.out.println("current thread context classloader: " + Thread.currentThread().getContextClassLoader());
	         // current thread context classloader: sun.misc.Launcher$AppClassLoader@73d16e93  
			System.out.println("classloader of ServiceLoader: " + ServiceLoader.class.getClassLoader());
			// classloader of ServiceLoader: null
	     }
	 }
  	 ```
     注：
     ServiceLoader会带着当前线程上下文类加载器去寻找mysql驱动包里的META-INF/services/java.sql.Driver文件来加载文件里面列出的驱动类。
     若在main方法里的第一行有如下代码：`Thread.currentThread().setContextClassLoader(MyTest.class.getClassLoader().getParent());`
     当前线程的上下文类加载器将变成加载MyTest类的父加载器，即AppClassLoader的父加载器，亦即ExtClassLoader
     for循环里的代码也将不会执行，因为程序在执行到ServiceLoader类的354行时返回false（具体可以debug调试）
     具体可以参照ServiceLoader类的文档说明及源码！
	 
22、Class.forName(className); 除了会触发类的加载还会触发类的初始化。因为底层调用的是forName0(className, true, classLoader);
	第二个参数表示是否将加载的类进行初始化，而第三个参数是加载这个类的类加载器，是调用Class.forName(className);的类的类加载器。
	也可以显式调用Class.forName(className, boolean, classLoader); 来指定类是否初始化和使用哪个类加载器
	 
23、加载数据库驱动程序不需要显式写代码：`Class.forName("com.mysql.jdbc.Driver");`因为在获取数据库连接时`DriverManager.getConnection(url, username, password);`会去加载数据库驱动类。执行过程：
   * 调用DriverManager.getConnection会导致DriverManager类的静态代码块执行；
   * 静态代码块里执行loadInitialDrivers方法，会使用ServiceLoader去加载（ServiceLoader的370行）并实例化（ServiceLoader的380行）数据库驱动类（使用的是当前线程的上下文类加载器去加载）；
   * 实例化驱动类会导致驱动类的静态代码块执行，会调用DriverManager的registerDriver方法将自己注册添加到DriverManager的registerDrivers；
   * DriverManager.getConnection方法正式执行，会循环registerDrivers获取数据库连接，获取到即返回这个连接。

24、关于Java的跨平台原理，可参照如下文章：https://www.jianshu.com/p/03a947d5bc50

25、使用javap -verbose命令分析一个字节码文件时，将会分析该字节码文件的魔数、版本号、常量池、类信息、类的构造方法、类变量与成员变量等信息，加上-p参数，会显示出私有方法信息。
   * 魔数：所有的.class字节码文件的前4个字节都是魔数，魔数值为固定值：0xCAFEBABE。魔数之后的4个字节为版本信息，前两个字节表示minor version(次版本号)，后两个字节表示major version(主版本号)。这里的版本号为
	 00 00 00 34，换算成十进制，表示次版本号为0，主版本号为52(对应jdk版本是1.8，51对应1.7，以此类推)。所以，该文件的版本号为1.8.0
   * 常量池(constant pool)：紧接着主版本号之后的就是常量池入口。一个Java类中定义的很多信息都是由来维护和描述的，可以将常量池看作是class文件的资源库，比如说Java类中定义的方法与变量信息，都是存储在常量池中。常量池中主要存储两类常量：字面量和符号引用。字面量
	 如文本字符串(Java中声明为final的常量等)，而符号引用如类和接口的全限定名，字段的名称和描述符，方法的名称和描述符等。
	 常量池的总体结构：Java类所对应的常量池主要由常量池数量与常量池数组(常量表)这两部分组成。常量池数量紧跟在主版本号后面，占据2个
	 字节；常量池数组则紧跟在常量池数量之后。常量池数组与一般的数组不用的是，常量池数组中不同的元素类型、结构都是不同的，长度当然
	 也就不同；但是，每一种元素的第一个数据都是一个u1类型，该字节是个标志位，占据1个字节。JVM在解析常量池时，会根据这个u1类型来获取
	 元素的具体类型。值得注意的是，常量池数组中的元素个数 = 常量池数 - 1(其中0暂时不使用)，目的是满足某些常量池索引值的数据在特定
	 情况下需要表达"不引用任何一个常量池"的含义；根本原因在于，索引为0也是一个常量(保留常量)，只不过它不位于常量表中，这个常量就对应
	 null值；所以，常量池的索引从1而非0开始。
   * 在JVM规范中，每个变量/字段都有描述信息，描述信息的主要作用是描述字段的数据类型、方法的参数列表
	 (包括数量、类型与顺序)与返回值。根据描述符规则，基本数据类型和代表无返回值的void类型都用一个大写字符
	 来表示，对象类型则使用字符L+对象的全限定名来表示，目的是为了压缩字节码文件的体积。如下所示：
	 B - byte，C - char，D - double，F - float，I - int，J - long，S - short，Z - boolean，V - void，
	 L - 对象类型(Ljava/lang/String;)
   * 对于数组类型来说，每一个维度使用一个前置的[来表示，如int[]被记录为[I，
         String[][]被记录为[[Ljava/lang/String;
   * 用描述符描述方法时，按照先参数列表，后返回值的顺序来描述。参数列表按照参数的严格顺序放在一组()之内，
         如方法：String getRealnamebyidAndNickname(int id, String name)的描述符为：
	 (I, Ljava/lang/String;)Ljava/lang/String;
	 
26、从字节码里<init>方法的Code属性可以看出，类里成员变量的初始化其实是在该<init>方法里即源文件里的构造方法里完成的，
	无论是源文件里没有构造方法编译器自动生成的构造方法，还是源文件里本来就写好的构造方法(无论写了多少个)。
	原因是编译器在编译源文件时会将指令做一个重排序。

27、对于Java类中的每一个实例方法(非static方法)，其在编译后所生成的字节码当中，方法参数的数量总是会比源代码中方法参数的数量多一个(this)，
	它位于方法的第一个参数位置处，这样，我们就可以在Java的实例方法中使用this来去访问当前对象的属性以及其他方法。
	这个操作是在编译器完成的，即由javac编译器在编译的时候将对this的访问转化为对一个普通实例方法第一个参数的访问，接下来在运行i期间，
	由JVM在调用实例方法时，自动向实例方法传入该this参数。所以，在实例方法的局部变量表中，至少会有一个指向当前对象的局部变量。
	这一点除了在字节码中体现，在Java8的新特性方法引用里也能够体现：
```java
class This {
    String two(int i, double d) {
        return "";
    }
}

interface TwoArgs {
    String call2(This athis, int i, double d);
}

public class MultiUnbound {
    public static void main(String[] args) { 
        TwoArgs twoargs = This::two;	// 编译后的字节码中，This的two方法实为3个参数(第一个参数为this)，
                                        // 与TwoArgs的call2方法签名相同
        This athis = new This();
        twoargs.call2(athis, 11, 3.14);
    }
}
```    
28、有些符号引用(类、方法、字段的全限定名)是在类加载阶段(连接阶段的解析阶段)就会转换为直接引用，
这种转换叫做静态解析；
另外一些符号引用则是在每次运行期转换为直接引用，这种转换叫做动态链接，这体现为Java的多态性。例如：
```java
abstract class Animal {
    abstract public void eat();
}
class Cat extends Animal {
    @Override
    public void eat() {
        System.out.println("cat eat...");
    }
}
class Dog extends Animal {
    @Override
    public void eat() {
        System.out.println("dog eat...");
    }
}

public class Test {
    public void test(Animal animal) {
        animal.eat(); 
    }
}
```
animal调用eat方法，在编译期引用的是Animal类的eat方法(符号引用)，只有在运行期才能动态确定animal被赋予的
是Animal的哪个子类对象，从而确定是调用哪个子类的eat方法，这时才将eat的符号引用转换为具体子类对象的eat方法的直接引用地址。

29、字节码里方法调用的5种形态：
   * invokeinterface：调用接口中的方法，实际上是在运行期决定的，决定到底调用实现该接口的哪个对象的特定方法。
   * invokestatic：调用静态方法。
   * invokespecial：调用自己的私有方法、构造方法(<init>)以及父类方法。
   * invokevirtual：调用虚方法，运行期动态查找的过程。
   * invokedynamic：动态调用方法。

30、静态解析的4种情形：
   * 静态方法
   * 父类方法
   * 构造方法
   * 私有方法(无法被重写)

以上4类方法称作非虚方法，它们是在类加载阶段就可以将符号引用转换为直接引用。

31、 方法的静态分派(涉及到的概念为方法重载)：
```java
public class MyTest {

    // 方法重载是一种静态行为，编译器就可以完全确定

    public void test(Grandpa grandpa) {
        System.out.println("grandpa");
    }

    public void test(Father father) {
        System.out.println("father"); 
    }

    public void test(Son son) {
        System.out.println("son"); 
    }

    public static void main(String[] args) {
        // g1的静态类型是Grandpa，而g1的实际类型(真正指向的类型)是Father。
        // 变量的静态类型是不会发生变化的，而变量的实际类型则是可以发生变化的(多态的一种体现)
        // 实际类型是在运行期才可确定。
        Grandpa g1 = new Father();
        Grandpa g2 = new Son();
        MyTest myTest = new MyTest();
        myTest.test(g1);	// grandpa，对应的字节码指令为invokevirtual
        myTest.test(g2);	// grandpa，对应的字节码指令为invokevirtual 
    }
}

class Grandpa {}
class Father extends Grandpa {}
class Son extends Father {}
```
32、实例化对象的字节码指令一般有4个：
   * new：创建指定类型的对象实例，对其进行默认初始化，并将指向该实例的一个引用压入操作数栈顶
   * dup：将操作数栈顶的引用复制一份(duplicate)，以备这个引用被invokespecial指令消耗掉之后还能被astore指令使用
   * invokespecial：调用自身构造器<init>，会传入隐式的this参数，这个this就是new指令压入操作数栈顶的实例引用，它在invokespecial指令执行时会被消耗掉
   * astore_(n)：也会把操作数栈顶的那个引用消耗掉，保存到指定的局部变量中去，因此在invokespeial指令执行消耗掉操作数栈顶引用之前先dup一份引用

33、方法的动态分派(涉及到的概念为方法重写)：
方法重写与方法重载区别的重要标志是：方法接收者，即调用方法的对象。
```java
public class MyTest {
    public static void main(String[] args) {
        Fruit apple = new Apple();
        Fruit orange = new Orange();

        apple.test();	// apple，对应的字节码指令为invokespecial
        orange.test();	// orange，对应的字节码指令为invokespecial
    }
}

class Fruit {
    public void test() {
        System.out.println("fruit");
    }
}

class Apple extends Fruit {
    @Override 
    public void test() {
        System.out.println("apple");
    }
}

class Orange extends Fruit {
    @Override
    public void test() {
        System.out.println("orange");
    }
}
```
注：invokespecial字节码指令的动态查找流程：
   * 找到操作数栈顶的第一个元素引用所指向的实际类型的对象
   * 在常量池中查找与所重写的方法签名(即符号引用)相同方法名和描述符的方法引用，若找到了，则执行第(3)步；若没找到，则执行第(4)步
   * 将invokespecial字节码指令所对应的符号引用转换为所查找到的直接引用
   * 在父类对象的常量池中查找，重复执行(2)、(3)、(4)，直到找到为止，若没找到则抛异常

34、jdk动态代理的本质是程序在运行过程中动态生成所代理类的字节码，程序通过加载该字节码生成Class对象，而后
通过Class对象反射进行实例化最终生成动态代理对象。字节码class文件是否生成到硬盘由系统参数
sun.misc.ProxyGenerator.saveGeneratedFiles是否为true决定。Object类的hashCode、equals、toString方法也会被
代理，即动态生成的代理类当中，这3个方法会以和被代理类里的方法同样的代理方法生成形式生成到代理类当中：
```
private static Method m1;
static {
    try {
        m1 = Class.forName("java.lang.Object").getMethod("equals", Class.forName("java.lang.Object"));
    } catch...
    }
}

public final boolean equals(Object var1) {
    try {
        return (Boolean) super.h.invoke(this, m1, new Object[] { var1 });
    } catch...
    }
}
```