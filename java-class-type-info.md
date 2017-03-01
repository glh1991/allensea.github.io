JAVA是如何在运行中识别对象和类的信息的? 主要采用两种方式: 

* RTTI: 假定我们在编译时就已经知道了所有的类型
* 反射机制: 允许我们在运行时发现和使用类的信息

**关于RTTI**

在JAVA中,所有的类型转换都是在运行时进行正确性检查的.而RTTI的含义就是: 在运行时,识别到一个对象的类型. 

类是程序的一部分, 每个类都有一个Class对象, Class对象包含了与类有关的信息, 实际上, Class对象也是用来创建类的所有的常规对象的.

每当编译一个新类, 就会产生一个Class对象(物理上是一个同名的.class文件), .class文件是这个类的字节码文件. JVM的类加载器首先检查这个类的Class对象是否已经被加载了,如果还未被加载就通过findClass()读取同名文件,将其转成byte数组.然后再通过defineClass()方法来生成这个类的对象. 一旦某个类的Class对象被加载到内存, 他就可以被用来创建这个类的所有对象.

Class对象提供了那些方法呢?

* Class.forName("className"): forName这个方法是Class类的一个static的成员,类对象就和其他对象一样,我们可以获取并操作他的引用. forName()是取得Class对象的引用的一种方法.当然forName()也有副作用,比如没有找到相关的class会抛出ClassNotFoundException异常.需要注意的是这个className字符串必须使用全限定名(包含包名).
* newInstance(): newInstance()是Class的一个实例方法,可以通过newInstance()方法来创建类

除了使用forName,JAVA还提供一个更安全的的方法: 类字面常量.