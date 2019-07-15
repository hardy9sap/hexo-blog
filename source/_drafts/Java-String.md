title: About Java String
categories: Coding
tags: [Basic]
toc: true
date: 2015-12-13 16:43:03
---

Java 的 String 类几乎是 Java 中最常使用到的对象类型，关于 String 的一些基础知识对开发者来说显得尤为重要。下面就对 String 相关的一些知识点进行列举和归纳，希望能够加深对 String 对象的认识。

<!--more-->

## 字符串常量池

JVM为了提高性能和减少内存开销，在实例化字符串常量的时候进行了一些优化。为了减少在JVM中创建的字符串的数量，字符串类维护了一个字符串池，每当代码创建字符串常量时，JVM会首先检查字符串常量池。如果字符串已经存在池中，就返回池中的实例引用。如果字符串不在池中，就会实例化一个字符串并放到池中。**Java能够进行这样的优化是因为字符串是不可变的，可以不用担心数据冲突进行共享。** 

另外需要注意**通过new操作符创建的字符串对象不指向字符串池中的任何对象，但是可以通过使用字符串的intern()方法来指向其中的某一个。**

## String.itern()的基本原理和作用

String.intern()是一个Native方法，底层调用C++的 StringTable::intern 方法，源码注释：当调用 intern 方法时，如果常量池中已经该字符串，则返回池中的字符串；否则将此字符串添加到常量池中，并返回字符串的引用。

所以明面上，它有两大好处，一是重复的字符串，会用同一个引用代替；二是字符串比较，不再需要逐个字符的equals()比较，而用==对比引用是否相同即可。


## `String s = new String(“abc”)`语句创建了几个对象？

首先括号里的"abc"先到String pool里看有没"abc"这个对象，没有则在pool里创建这个对象。此时在pool创建了一个"abc"对象。然后通过new语句又创建了一个"abc"对象，而这个对象是放在内存的堆里，这里的s指向堆里的对象。

故创建常量池和堆内存中两个对象，两个对象的地址值不一样，返回的是堆内存的地址。

## String 的 `==`、`equals` 以及 `hashcode`    

先来看一段代码：

```java
public static void main(String[] args) {
	String a = "hello";
	String b = "hello";
	String c = new String("hello");
	String d = new String("hello");
	System.out.println(a.hashCode() == b.hashCode()); //true， String重写了hashcode方法，只跟String的value[]有关
	System.out.println(a.hashCode() == c.hashCode()); //true 同上
	System.out.println(a.equals(b)); //true，String也重写了equals方法，只跟value[]有关
	System.out.println(a.equals(c)); //true，同上
	System.out.println(a == b); //true,因为a和b都在常量池中，
	System.out.println(a == c); //false，a在常量池中，c在堆中
	System.out.println(c == d); //false，c和d都在堆中，但是为不同的对象
	
	String hello = "Hello", lo = "lo";
	System.out.println((hello == ("Hel"+"lo"))); //true,"Hel"+"lo"在编译时进行计算，被当做常量
	System.out.println((hello == ("Hel"+lo))); //false，在运行时通过连接计算出的字符串是新创建的，因此是堆中创建的新对象
	System.out.println(hello == ("Hel"+lo).intern()); //true，通过使用字符串的intern()方法来指向字符串池中的对象
}
```


再来看看 String 重写的 hashcode 及 equals 方法

```java    
public int hashCode() {
    int h = hash;
    if (h == 0 && value.length > 0) {
        char val[] = value;

        for (int i = 0; i < value.length; i++) {
            h = 31 * h + val[i];
        }
        hash = h;
    }
    return h;
}   

public boolean equals(Object anObject) {
    if (this == anObject) {
        return true;
    }
    if (anObject instanceof String) {
        String anotherString = (String)anObject;
        int n = value.length;
        if (n == anotherString.value.length) {
            char v1[] = value;
            char v2[] = anotherString.value;
            int i = 0;
            while (n-- != 0) {
                if (v1[i] != v2[i])
                    return false;
                i++;
            }
            return true;
        }
    }
    return false;
}    
```

结论：

- String的hashCode之和value有关
- equals比较的也只是value（指向同一个对象时value也必然相同，换句话就是`a==b ||（a.length=b.length && {a[i]=b[i]}）`）
- a==b比较的是ref
- hashSet中比较是否重复的依据是`a.hasCode()=b.hasCode() && a.equals(b)`
- 因此两个不同ref的String对象在hashSet中会被认为是同一个元素
- **两个String的hashCode相同并不代表着equals比较时会相等**，**即不同的String可能会产生相同的hash值**，例如`Aa`和`BB`的hashcode都是2112


## hashcode 方法的作用

Java的Object类中定义了`public native int hashCode()`方法，也就是说每一个java类都会继承这个方法（有的类会重写此方法），那这个方法到底有什么用，为什么这么重要？

**在Java中，hashCode方法的主要作用是为了配合基于散列的集合一起正常运行，这样的散列集合包括HashSet、HashMap以及HashTable。**

举个例子：当向集合中插入对象时，如何判别在集合中是否已经存在该对象了？（注意：集合中不允许重复的元素存在）。

很容易想到的方法是调用equals方法来逐个进行比较，这个方法确实可行，但是如果集合中已经存在一万条数据或者更多的数据，如果采用equals方法去逐一比较，效率必然是一个问题。

JDK中实现的方法是：当集合要添加新的对象时，先调用这个对象的hashCode方法，得到对应的hashcode值，实际上在HashMap的具体实现中会用一个table保存已经存进去的对象的hashcode值（hashcode值一般都会被缓存），如果table中没有该hashcode值，它就可以直接存进去，不用再进行任何比较了；如果存在该hashcode值， 就调用它的equals方法与新元素进行比较，相同的话就不存了，不相同就散列其它的地址。这样一来实际调用equals方法的次数就大大降低了。

总结一句话就是：**可以根据hashcode值判断两个对象是否不等，这样效率更高！**


## 可以直接根据hashcode值判断两个对象是否相等吗？

不可以！前面也演示了不同的对象可能会生成相同的hashcode值。虽然不能根据hashcode值判断两个对象是否相等，但是可以直接根据hashcode值判断两个对象不等，如果两个对象的hashcode值不等，则必定是两个不同的对象。

* 两个对象调用equals方法得到的结果为true，则两个对象的hashcode值必定相等；
* 如果equals方法得到的结果为false，则两个对象的hashcode值不一定不同
* 如果两个对象的hashcode值不等，则equals方法得到的结果必定为false；
* 如果两个对象的hashcode值相等，则equals方法得到的结果未知。


## 在重写equals方法的同时，必须重写hashCode方法。为什么这么说？

重写 equals 方法后必须重写 hashCode 是为了让 Java 中所有使用到 Hash 算法的数据结构能够正常运行。

举个例子：假如 a 和 b 是两个对象，你重写了 equals() 方法，a.equals(b)结果为true。

然后创建一个 HashMap，执行 map.put(a,c); map 中插入了一条数据，键是 a 值是 c，调用 map.get(a) 可以返回对象 c，但是调用 map.get(b) 却不能返回对象 c， 但是 a 和 b 两个对象是相等的，相等的对象却得不到相同的结果，就不符合逻辑了。因为 HashMap 是根据键对象的 HashCode 来进行快速查找的，所以你必须保证 a 和 b 这两个相同对象的 HashCode 也相同，因此你需要重写 hashCode() 方法。另外，如果你要用到 HashSet，在这个例子中 a 和 b 可以同时插入到 HashSet 中，然而这两个对象在逻辑上有时相等的，这不符合 HashSet 的定义。

另外这也是hashcode方法的要求，在Object的hashcode方法注释中明确做了如下说明：
> If two objects are equal according to the equals(Object) method, then calling the hashCode() method on each of the two objects must produce the same integer result.


## hashcode 是对象的内存地址吗

首先String就不是，因为它重写了hashcode方法。这个问题问的应该是object中的hashcode的native方法。因为不同JVM的实现不一样，可能有JVM这样实现，但大多时候并不是这样，只能说可能存储地址有一定关联。



## String, StringBuffer，StringBuilder的区别？

- String 是字符串常量，是不可变的，而 StringBuffer 及 StringBuilder 可变
- 在做字符串拼接时，直接操作 String 比较耗资源，因为它的不可变性，会导致创建多个中间对象
- String、StringBuffer 是线程安全的，StringBuilder 线程不安全

三者之间的关系看这张图：

![](//
jverson.oss-cn-beijing.aliyuncs.com/1294873a37967d1f431ce3a5db02fe75.jpg)

看源码会发现 StringBuffer 和 StringBuilder 继承了同样的接口和抽象类，其中的方法和实现也几乎都一样，唯一的区别就是在 StringBuffer 中很多方法都加了 `synchronized` 修饰符
从而达到线程安全的目的。

## String是不可变的有什么好处？
       
1. 由于String是不可变类，所以在多线程中使用是安全的，我们不需要做任何其他同步操作。
2. 不同的字符串变量可以引用池中的相同的字符串，节省大量内存空间。


## Java 能不能自己创建一个 `java.lang.String` 的对象

先说结论，下面会详细解释。可以创建，但是不能正常的加载。首先 JVM 类加载的`双亲委托机制`使得所有的类都优先从父类或者启动类加载，导致同名自定义的类没有机会加载；其次即使定义了一个父类找不到的类名从而轮到自定义的类加载器加载，也会因为 JVM 限制包名以 `java.*` 开头的类加载，会抛出一个安全异常。

其实这个问题和 String 没什么关系，所有 rt.jar 包下的类都同理。这里主要涉及到 Java 的类加载机制，以及类加载的一些特殊限制（处于安全考虑）。下面对类加载相关的知识进行简单总结。

Java 类加载有一个称为双亲委托的机制：**某个特定的类加载器在接到加载类的请求时，首先将加载任务委托给父类加载器，依次递归，如果父类加载器可以完成类加载任务，就成功返回；只有父类加载器无法完成此加载任务时，才自己去加载。**

使用双亲委派模型的好处在于 Java 类随着它的类加载器一起具备了一种带有优先级的层次关系。例如类java.lang.Object，它存在在 rt.jar 中，无论哪一个类加载器要加载这个类，最终都是委派给处于模型最顶端的 Bootstrap ClassLoader 进行加载，因此 Object 类在程序的各种类加载器环境中都是同一个类。

jdk 自带了3种类加载器，分别是启动类加载器（Bootstrap ClassLoader），扩展类加载器（Extension ClassLoader），应用程序类加载器（Application ClassLoader）。启动类加载器由native实现（HotSpot虚拟机中由c++实现），后两种加载器是继承自抽象类 java.lang.ClassLoader。


在 rt.jar 包中的 java.lang.ClassLoader 类中，我们可以查看类加载实现过程的代码如下：

```java
protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException
{
    synchronized (getClassLoadingLock(name)) {
        // First, check if the class has already been loaded
        Class<?> c = findLoadedClass(name); //首先检查该类是否已经加载过
        if (c == null) { //没加载过的话按照以下步骤加载
            long t0 = System.nanoTime();
            try {
                if (parent != null) { //1. 有父加载器时递归调用父加载器加载
                    c = parent.loadClass(name, false);
                } else { //2. 没有父加载器时调用启动加载器加载
                    c = findBootstrapClassOrNull(name);
                }
            } catch (ClassNotFoundException e) {
                // ClassNotFoundException thrown if class not found
                // from the non-null parent class loader
            }
            //3. 前面几步都没有加载的情况下调用加载器实现类自定义的findClass方法自定义加载
            if (c == null) {
                // If still not found, then invoke findClass in order
                // to find the class.
                long t1 = System.nanoTime();
                c = findClass(name);

                // this is the defining class loader; record the stats
                sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                sun.misc.PerfCounter.getFindClasses().increment();
            }
        }
        if (resolve) {
            resolveClass(c);
        }
        return c;
    }
}
```

如果需要自定义类加载器，方法注释中可以看到 *Subclasses of ClassLoader are encouraged to override findClass(String) rather than this method.* 也就是说一般子类只需要重写这个方法即可，而不是重写整个 loadClass() 方法。





