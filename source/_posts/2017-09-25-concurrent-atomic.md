---
layout: 	post
title: 		Concurrent包-CAS/Unsafe和Atomic系列
date: 		2017-09-25 23:47:19
author:		ruijie
catalog:	true
tags:
	- 并发
---

> Unsafe中提供的CAS原子操作以及具体Atomic相关类中的实现

## 一. CAS和Unsafe

>[CAS](https://en.wikipedia.org/wiki/Compare-and-swap): Compare-and-Swap, is an atomic instruction used in multithreading to achieve synchronization.

java.util.concurrent包下提供了一系列并发相关的类，包括线程安全的集合，数据结构和线程池等，其中大量使用了`CAS(Compare and swap)`和Unsafe类。并且像`ConcurrentHashMap`在`JDK1.8`中也已经放弃使用`Segment`分段锁而采用`Unsafe`和`CAS`来替代。

#### 1. CAS操作

```
/**
* 比较obj的offset处内存位置中的值和期望的值，如果相同则更新。此更新是不可中断的。
* 
* @param obj 需要更新的对象
* @param offset obj中整型field的偏移量
* @param expect 希望field中存在的值
* @param update 如果期望值expect与field的当前值相同，设置filed的值为这个新值
* @return 如果field的值被更改返回true
*/
public native boolean compareAndSwapInt(Object obj, long offset, int expect, int update);
```
简单的说，通过对比这个对象我们的期望值和实际值是不是一样，如果是的话则执行更新。可以看到调用的是`native`方法，是硬件级别的原子操作。

#### 2. CPU指令对于CAS的支持
会不会出现我们比较完期望值和实际值是相同的，在更新赋值之前被其他线程更新了呢，答案是否定的。因为CAS是一种系统原语，原语属于操作系统用语范畴，是由若干条指令组成的，用于完成某个功能的一个过程，并且原语的执行必须是连续的，在执行过程中不允许被中断，也就是说CAS是一条CPU的原子指令，不会造成所谓的数据不一致问题。

#### 3. CAS会有什么问题

- ABA问题：简单的说对于一个变量A=5，线程1将A从5修改为10，在修改为5，线程2同时将A从5修改为6，ABA的问题即线程2可能看不到线程1的修改，因为只关注结果而不是这个过程

#### 4. Unsafe类

Unsafe类存在于sun.misc包中，其内部方法操作可以像C的指针一样直接操作内存，单从名称看来就可以知道该类是非安全的，毕竟Unsafe拥有着类似于C的指针操作，因此总是不应该首先使用Unsafe类，Java官方也不建议直接使用的Unsafe类，CAS操作的执行依赖于Unsafe类的方法，注意Unsafe类中的所有方法都是native修饰的，也就是说Unsafe类中的方法都直接调用操作系统底层资源执行相应任务，关于Unsafe类的主要功能点如下：

- 内存管理，Unsafe类中存在直接操作内存的方法

```
//分配内存指定大小的内存
public native long allocateMemory(long bytes);
//根据给定的内存地址address设置重新分配指定大小的内存
public native long reallocateMemory(long address, long bytes);
//用于释放allocateMemory和reallocateMemory申请的内存
public native void freeMemory(long address);
//将指定对象的给定offset偏移量内存块中的所有字节设置为固定值
public native void setMemory(Object o, long offset, long bytes, byte value);
//设置给定内存地址的值
public native void putAddress(long address, long x);
//获取指定内存地址的值
public native long getAddress(long address);

//设置给定内存地址的long值
public native void putLong(long address, long x);
//获取指定内存地址的long值
public native long getLong(long address);
//设置或获取指定内存的byte值
public native byte  getByte(long address);
public native void  putByte(long address, byte x);
//其他基本数据类型(long,char,float,double,short等)的操作与putByte及getByte相同

//操作系统的内存页大小
public native int pageSize();
```
- 实例化对象

```
//传入一个对象的class并创建该实例对象，但不会调用构造方法
public native Object allocateInstance(Class cls) throws InstantiationException;
```
- 类和实例对象以及变量的操作

```
//获取字段f在实例对象中的偏移量
public native long objectFieldOffset(Field f);
//静态属性的偏移量，用于在对应的Class对象中读写静态属性
public native long staticFieldOffset(Field f);
//返回值就是f.getDeclaringClass()
public native Object staticFieldBase(Field f);
//获得给定对象偏移量上的int值，所谓的偏移量可以简单理解为指针指向该变量的内存地址，
//通过偏移量便可得到该对象的变量，进行各种操作
public native int getInt(Object o, long offset);
//设置给定对象上偏移量的int值
public native void putInt(Object o, long offset, int x);
//获得给定对象偏移量上的引用类型的值
public native Object getObject(Object o, long offset);
//设置给定对象偏移量上的引用类型的值
public native void putObject(Object o, long offset, Object x);
//其他基本数据类型(long,char,byte,float,double)的操作与getInthe及putInt相同
//设置给定对象的int值，使用volatile语义，即设置后立马更新到内存对其他线程可见
public native void  putIntVolatile(Object o, long offset, int x);
//获得给定对象的指定偏移量offset的int值，使用volatile语义，总能获取到最新的int值。
public native int getIntVolatile(Object o, long offset);
//其他基本数据类型(long,char,byte,float,double)的操作与putIntVolatile及getIntVolatile相同，引用类型putObjectVolatile也一样。
//与putIntVolatile一样，但要求被操作字段必须有volatile修饰
public native void putOrderedInt(Object o,long offset,int x);
```
下面是一个简单的例子：
```
public class Test {
    public static void main(String[] args) {
        try {
            //通过反射获取unsafe对象
            //因为getUnsafe()方法只能由Bootstrap类加载器使用
            Field field = Unsafe.class.getDeclaredField("theUnsafe");
            field.setAccessible(true);
            Unsafe unsafe = (Unsafe) field.get(null);
            System.out.println(unsafe);

            //通过allocateInstance方法实例化对象
            //不是通过构造方法
            TestDomain domain = (TestDomain) unsafe.allocateInstance(TestDomain.class);
            System.out.println(domain.toString());

            //获取domain对象的属性
            Field name = domain.getClass().getDeclaredField("name");
            Field age = domain.getClass().getDeclaredField("age");

            Field tag = domain.getClass().getDeclaredField("tag");
            Object staticTag = unsafe.staticFieldBase(tag);

            //domain实例变量在对象内存中的偏移量
            long nameOffset = unsafe.objectFieldOffset(name);
            long ageOffset = unsafe.objectFieldOffset(age);
            long tagOffset = unsafe.staticFieldOffset(tag);

            //通过偏移量给domain对象的实例变量设置值
            unsafe.putObject(domain, nameOffset, "hehe");
            unsafe.putInt(domain, ageOffset, 18);
            unsafe.putObject(staticTag, tagOffset, "HAHA");
            System.out.println(domain.toString());
        } catch (NoSuchFieldException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        } catch (InstantiationException e) {
            e.printStackTrace();
        }
    }
}

class TestDomain{
    public TestDomain() {
        System.out.println("Constructor being invoked");
    }

    private String name;
    private int age;
    private static String tag = "HEHE";

    @Override public String toString() {
        return "TestDomain{" +
            "name='" + name + '\'' +
            ", age=" + age + '\'' +
            ", tag=" + tag +
            '}';
    }
}
```
- 数组操作

```
//获取数组第一个元素的偏移地址
public native int arrayBaseOffset(Class arrayClass);
//数组中一个元素占据的内存空间,arrayBaseOffset与arrayIndexScale配合使用，可定位数组中每个元素在内存中的位置
public native int arrayIndexScale(Class arrayClass);
```
- CAS操作

```
//第一个参数o为给定对象，offset为对象内存的偏移量，通过这个偏移量迅速定位字段并设置或获取该字段的值，
//expected表示期望值，x表示要设置的值，下面3个方法都通过CAS原子指令执行操作。
public final native boolean compareAndSwapObject(Object o, long offset,Object expected, Object x);                                                                                                  
public final native boolean compareAndSwapInt(Object o, long offset,int expected,int x);

public final native boolean compareAndSwapLong(Object o, long offset,long expected,long x);
```
JDK 1.8新增的几个方法：
```
//这是一个CAS操作过程，直到设置成功方能退出循环，返回旧值
public final int getAndAddInt(Object o, long offset, int delta) {
    int v;
    do {
        //获取内存中最新值
        v = getIntVolatile(o, offset);
      //通过CAS操作
    } while (!compareAndSwapInt(o, offset, v, v + delta));
    return v;
 }

//这是一个CAS操作过程，直到设置成功方能退出循环，返回旧值
public final int getAndSetInt(Object o, long offset, int newValue) {
    int v;
    do {
        v = getIntVolatile(o, offset);
    } while (!compareAndSwapInt(o, offset, v, newValue));
    return v;
}

public final long getAndAddLong(Object o, long offset, long delta) {
    long v;
    do {
        v = getLongVolatile(o, offset);
    } while (!compareAndSwapLong(o, offset, v, v + delta));
    return v;
}

public final long getAndSetLong(Object o, long offset, long newValue) {
    long v;
    do {
        v = getLongVolatile(o, offset);
    } while (!compareAndSwapLong(o, offset, v, newValue));
    return v;
}

public final Object getAndSetObject(Object o, long offset, Object newValue) {
    Object v;
    do {
        v = getObjectVolatile(o, offset);
    } while (!compareAndSwapObject(o, offset, v, newValue));
    return v;
}
```

- 线程挂起和恢复
将一个线程进行挂起是通过park方法实现的，调用 park后，线程将一直阻塞直到超时或者中断等条件出现。unpark可以终止一个挂起的线程，使其恢复正常。Java对线程的挂起操作被封装在 LockSupport类中，LockSupport类中有各种版本pack方法，其底层实现最终还是使用Unsafe.park()方法和Unsafe.unpark()方法

```
//线程调用该方法，线程将一直阻塞直到超时，或者是中断条件出现。  
public native void park(boolean isAbsolute, long time);  

//终止挂起的线程，恢复正常.java.util.concurrent包中挂起操作都是在LockSupport类实现的，其底层正是使用这两个方法，  
public native void unpark(Object thread); 
```
- 内存屏障
这里主要包括了loadFence、storeFence、fullFence等方法，这些方法是在Java8新引入的，用于定义内存屏障，避免代码重排序，与Java内存模型相关

```
//在该方法之前的所有读操作，一定在load屏障之前执行完成
public native void loadFence();
//在该方法之前的所有写操作，一定在store屏障之前执行完成
public native void storeFence();
//在该方法之前的所有读写操作，一定在full屏障之前执行完成，这个内存屏障相当于上面两个的合体功能
public native void fullFence();
```
---

## 二. 原子操作类(Atomic系列)

上面是CAS和Unsafe类以及其中提供的方法的一些介绍，下面是在`concurren`包中如何通过它们来具体实现原子操作的。从功能上来说Atomic系列可以简单分为四类：
- 基本类型
- 引用类型
- 数组
- 属性

#### 1. 原子更新基本类型

基本类型相关的类：
- *AtomicBoolean*
- *AtomicInteger*
- *AtomicLong*
下面以`AtomicInteger`类为例看一下具体实现：

```
public class AtomicInteger extends Number implements java.io.Serializable {
    private static final long serialVersionUID = 6214790243416807050L;

    // setup to use Unsafe.compareAndSwapInt for updates
    // unsafe实例
    private static final Unsafe unsafe = Unsafe.getUnsafe();
    //偏移量
    private static final long valueOffset;

    static {
        try {
            //实例变量value在对象内存中的偏移量
            valueOffset = unsafe.objectFieldOffset
                (AtomicInteger.class.getDeclaredField("value"));
        } catch (Exception ex) { throw new Error(ex); }
    }
    //使用volatile修饰，保证属性的内存可见性
    private volatile int value;

    /**
     * 有参构造方法
     *
     * @param initialValue the initial value
     */
    public AtomicInteger(int initialValue) {
        value = initialValue;
    }

    /**
     * 无参构造方法，初始值为0
     */
    public AtomicInteger() {
    }

    /**
     * @return the current value
     */
    public final int get() {
        return value;
    }

    /**
     * @param newValue the new value
     */
    public final void set(int newValue) {
        value = newValue;
    }

    /**
     * 类似懒加载，可能会使得其他线程看见旧值
     * @param newValue the new value
     * @since 1.6
     */
    public final void lazySet(int newValue) {
        unsafe.putOrderedInt(this, valueOffset, newValue);
    }

    /**
     * 设置value为给定值，并且返回旧值
     * @param newValue the new value
     * @return the previous value
     */
    public final int getAndSet(int newValue) {
        return unsafe.getAndSetInt(this, valueOffset, newValue);
    }

    /**
     * compar-and-swa，如果当前值等于期望值，则进行更新
     * 更新成功返回true，否则返回false
     * @param expect the expected value
     * @param update the new value
     * @return {@code true} if successful. False return indicates that
     * the actual value was not equal to the expected value.
     */
    public final boolean compareAndSet(int expect, int update) {
        return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
    }

    /**
     * 执行自加动作，返回旧值
     * @return the previous value
     */
    public final int getAndIncrement() {
        return unsafe.getAndAddInt(this, valueOffset, 1);
    }

    /**
     * 执行自减动作，返回旧值
     * @return the previous value
     */
    public final int getAndDecrement() {
        return unsafe.getAndAddInt(this, valueOffset, -1);
    }
    
    /**
     * 执行自增动作，返回自增后的值
     * @return the updated value
     */
    public final int incrementAndGet() {
        return unsafe.getAndAddInt(this, valueOffset, 1) + 1;
    }

    /**
     * 执行自减动作，返回自减后的值
     * @return the updated value
     */
    public final int decrementAndGet() {
        return unsafe.getAndAddInt(this, valueOffset, -1) - 1;
    }

    /**
     * 执行增加动作，返回旧值
     * @param delta the value to add
     * @return the previous value
     */
    public final int getAndAdd(int delta) {
        return unsafe.getAndAddInt(this, valueOffset, delta);
    }

    /**
     * 执行增加动作，返回修改后的值
     * @param delta the value to add
     * @return the updated value
     */
    public final int addAndGet(int delta) {
        return unsafe.getAndAddInt(this, valueOffset, delta) + delta;
    }
}
```
`AtomicInteger`提供的这些操作，主要是由Unsafe类中的`getAndAddInt()`和`getAndSetInt()`方法来实现的，上面已经看到这个方法的实现：
```
//这是一个CAS操作过程，直到设置成功方能退出循环，返回旧值
public final int getAndAddInt(Object o, long offset, int delta) {
    int v;
    do {
        //获取内存中最新值
        v = getIntVolatile(o, offset);
      //通过CAS操作
    } while (!compareAndSwapInt(o, offset, v, v + delta));
    return v;
 }

//这是一个CAS操作过程，直到设置成功方能退出循环，返回旧值
public final int getAndSetInt(Object o, long offset, int newValue) {
    int v;
    do {
        v = getIntVolatile(o, offset);
    } while (!compareAndSwapInt(o, offset, v, newValue));
    return v;
}
```
国际惯例，demo：
```
public class Test{

    static AtomicInteger ai = new AtomicInteger();

    public static class TestThread implements Runnable{

        public void run() {
            for (int i = 0; i < 10000; i++){
                ai.incrementAndGet();
            }
        }
    }

    public static void main(String[] args) throws InterruptedException {
        Thread [] threads = new Thread[10];
        for (int i = 0; i < 10; i++) {
            threads[i] = new Thread(new TestThread());
        }

        for (Thread thread : threads) {
            thread.start();
        }
        for (Thread thread : threads) {
            thread.join();
        }
        System.out.println(ai.toString());
    }
}
```

#### 2. 原子更新引用类型

和`AtomicInteger`类提供的方法类似，也是由Unsafe类提供的原子操作方法实现

```
public class AtomicReference<V> implements java.io.Serializable {
    private static final long serialVersionUID = -1848883965231344442L;

    //获取unsafe实例
    private static final Unsafe unsafe = Unsafe.getUnsafe();
    private static final long valueOffset;

    static {
        try {
            //获取value的偏移量
            valueOffset = unsafe.objectFieldOffset
                (AtomicReference.class.getDeclaredField("value"));
        } catch (Exception ex) { throw new Error(ex); }
    }

    //使用volatile修饰保证内存可见性
    private volatile V value;

    /**
     * @return the current value
     */
    public final V get() {
        return value;
    }

    /**
     * @param newValue the new value
     */
    public final void set(V newValue) {
        value = newValue;
    }
    
    /**
     * 如果当前值和期望值相同，执行更新
     * 更新成功返回true，否则返回false
     * @param expect the expected value
     * @param update the new value
     * @return {@code true} if successful. False return indicates that
     * the actual value was not equal to the expected value.
     */
    public final boolean compareAndSet(V expect, V update) {
        return unsafe.compareAndSwapObject(this, valueOffset, expect, update);
    }

    /**
     * 执行更新并且返回旧值
     * @param newValue the new value
     * @return the previous value
     */
    @SuppressWarnings("unchecked")
    public final V getAndSet(V newValue) {
        return (V)unsafe.getAndSetObject(this, valueOffset, newValue);
    }
```

其中`getAndSet()`方法调用的是`unsafe.getAndSetObject()`的具体实现如下：
```
public final Object getAndSetObject(Object o, long offset, Object newValue) {
    Object v;
    do {
        v = getObjectVolatile(o, offset);
    } while (!compareAndSwapObject(o, offset, v, newValue));
    return v;
}
```

demo如下：
```
public class Test {
    static AtomicReference<User> user = new AtomicReference<User>();

    public static void main(String[] args) {
        User user1 = new User(18, "test");
        user.set(user1);

        User user2 = new User(15, "haha");
        user.compareAndSet(user1, user2);
        System.out.println(user.get().toString());
    }
}

class User{
    private int age;
    private String name;

    public User(int age, String name) {
        this.age = age;
        this.name = name;
    }

    @Override public String toString() {
        return "User{" +
            "age=" + age +
            ", name='" + name + '\'' +
            '}';
    }
}
```

#### 3. 原子更新数组

Atomic系列下数组相关的类有：
- *AtomicInteger*
- *AtomicLong*
- *AtomicReference*

具体实现如下：
```
public class AtomicIntegerArray implements java.io.Serializable {
    private static final long serialVersionUID = 2862133569453604235L;

    private static final Unsafe unsafe = Unsafe.getUnsafe();
    //数组第一个元素的内存起始地址
    private static final int base = unsafe.arrayBaseOffset(int[].class);
    //偏移量，用于计算数组每个元素地址
    private static final int shift;
    private final int[] array;

    static {
        //scal，每个数组元素占用的内存大小，int为4个字节
        int scale = unsafe.arrayIndexScale(int[].class);
        if ((scale & (scale - 1)) != 0)
            throw new Error("data type scale not a power of two");
        //31 - (scale转为二进制后头零个数=29) = 2
        shift = 31 - Integer.numberOfLeadingZeros(scale);
    }

    private long checkedByteOffset(int i) {
        if (i < 0 || i >= array.length)
            throw new IndexOutOfBoundsException("index " + i);

        return byteOffset(i);
    }

    private static long byteOffset(int i) {
        //shift为2，即base + i << 2,其中i为index，base为数组第一个元素的起始地址
        return ((long) i << shift) + base;
    }
    
    /**
     * index为i的值，具有volatile效果
     *
     * @param i the index
     * @return the current value
     */
    public final int get(int i) {
        return getRaw(checkedByteOffset(i));
    }

    private int getRaw(long offset) {
        //调用unsafe的方法，具有volatile效果
        return unsafe.getIntVolatile(array, offset);
    }

    /**
     * @param i the index
     * @param newValue the new value
     */
    public final void set(int i, int newValue) {
        unsafe.putIntVolatile(array, checkedByteOffset(i), newValue);
    }

    /**
     * 更新index为i的值，并且返回旧值
     * @param i the index
     * @param newValue the new value
     * @return the previous value
     */
    public final int getAndSet(int i, int newValue) {
        return unsafe.getAndSetInt(array, checkedByteOffset(i), newValue);
    }

    /**
     * 如果index为i的值和期望值相同，则执行更新，更新成功返回true
     * @param i the index
     * @param expect the expected value
     * @param update the new value
     * @return {@code true} if successful. False return indicates that
     * the actual value was not equal to the expected value.
     */
    public final boolean compareAndSet(int i, int expect, int update) {
        return compareAndSetRaw(checkedByteOffset(i), expect, update);
    }

    private boolean compareAndSetRaw(long offset, int expect, int update) {
        return unsafe.compareAndSwapInt(array, offset, expect, update);
    }

    /**
     * 把index为i的值执行自增+1，返回旧值
     * @param i the index
     * @return the previous value
     */
    public final int getAndIncrement(int i) {
        return getAndAdd(i, 1);
    }

    /**
     * 把index为i的值执行自减少-1，返回新值
     * @param i the index
     * @return the previous value
     */
    public final int getAndDecrement(int i) {
        return getAndAdd(i, -1);
    }

    /**
     * 执行相应的增加和减少动作，调用的是unsafe里的原子操作
     * @param i the index
     * @param delta the value to add
     * @return the previous value
     */
    public final int getAndAdd(int i, int delta) {
        return unsafe.getAndAddInt(array, checkedByteOffset(i), delta);
    }

    /**
     * @param i the index
     * @return the updated value
     */
    public final int incrementAndGet(int i) {
        return getAndAdd(i, 1) + 1;
    }

    /**
     * @param i the index
     * @return the updated value
     */
    public final int decrementAndGet(int i) {
        return getAndAdd(i, -1) - 1;
    }

    /**
     * Atomically adds the given value to the element at index {@code i}.
     *
     * @param i the index
     * @param delta the value to add
     * @return the updated value
     */
    public final int addAndGet(int i, int delta) {
        return getAndAdd(i, delta) + delta;
    }
```
和`AtomicInteger`这种基本类型所不同的是，`AtomicIntegerArray`中需要有数组元素的偏移量的计算，再看下计算方法：

> 元素内存地址：第一个元素起始地址 + index * 每个元素占用内存空间

即`byteOffset()`方法,通过下标计算每个元素的内存地址
```
//计算数组中每个元素的的内存地址
private static long byteOffset(int i) {
     return ((long) i << shift) + base;
 }
```

```
`shift = 31 - Integer.numberOfLeadingZeros(scale) = 2`
替换后如下：

base + (0 * 4) --> base + 0 << 2
base + (1 * 4) --> base + 1 << 2
```

demo如下：
```
public class Test{
    static AtomicIntegerArray aia = new AtomicIntegerArray(10);

    public static class TThread implements Runnable{
        public void run() {
            for (int i = 0; i < 10000; i++) {
                aia.incrementAndGet(i % aia.length());
            }
        }
    }

    public static void main(String[] args) throws InterruptedException {
        Thread [] threads = new Thread[10];
        for (int i = 0; i < 10; i++) {
            threads[i] = new Thread(new TThread());
        }

        for (Thread thread : threads) {
            thread.start();
        }
        for (Thread thread : threads) {
            thread.join();
        }
        System.out.println(aia.toString());
    }
}
```
#### 4. 原子更新属性

可以通过原子更新属性的方式给原本是普通变量的，增加原子更新的支持。Atomic中提供了以下三个类：
- *AtomicIntegerFieldUpdater*
- *AtomicLongFieldUpdater*
- *AtomicReferenceFieldUpdater*

但是使用条件相对比较严格：
- 操作的字段不能是static类型
- 操作的字段不能是final类型
- 字段必须是volatile修饰的
- 必须保证操作类和被操作类之间的可见性

```
//主要的代码实际处于*AtomicIntegerFieldUpdaterImpl*中
@CallerSensitive
    public static <U> AtomicIntegerFieldUpdater<U> newUpdater(Class<U> tclass,
                                                              String fieldName) {
        return new AtomicIntegerFieldUpdaterImpl<U>
            (tclass, fieldName, Reflection.getCallerClass());
    }
    
```
具体`AtomicIntegerFieldUpdaterImpl`的构造器如下：
```
private static class AtomicIntegerFieldUpdaterImpl<T>
            extends AtomicIntegerFieldUpdater<T> {
        private static final Unsafe unsafe = Unsafe.getUnsafe();
        private final long offset;
        private final Class<T> tclass;
        private final Class<?> cclass;

        AtomicIntegerFieldUpdaterImpl(final Class<T> tclass,
                                      final String fieldName,
                                      final Class<?> caller) {
            final Field field;
            final int modifiers;
            try {
                field = AccessController.doPrivileged(
                    new PrivilegedExceptionAction<Field>() {
                        public Field run() throws NoSuchFieldException {
                            //通过反射获取属性
                            return tclass.getDeclaredField(fieldName);
                        }
                    });
                modifiers = field.getModifiers();
                //检查access权限
                sun.reflect.misc.ReflectUtil.ensureMemberAccess(
                    caller, tclass, null, modifiers);
                ClassLoader cl = tclass.getClassLoader();
                ClassLoader ccl = caller.getClassLoader();
                if ((ccl != null) && (ccl != cl) &&
                    ((cl == null) || !isAncestor(cl, ccl))) {
                  sun.reflect.misc.ReflectUtil.checkPackageAccess(tclass);
                }
            } catch (PrivilegedActionException pae) {
                throw new RuntimeException(pae.getException());
            } catch (Exception ex) {
                throw new RuntimeException(ex);
            }

            Class<?> fieldt = field.getType();
            //检查field是否为int类型
            if (fieldt != int.class)
                throw new IllegalArgumentException("Must be integer type");
            //检查是否为volatile修饰
            if (!Modifier.isVolatile(modifiers))
                throw new IllegalArgumentException("Must be volatile type");
            this.cclass = (Modifier.isProtected(modifiers) &&
                           caller != tclass) ? caller : null;
            this.tclass = tclass;
            //属性在对象中的偏移量
            offset = unsafe.objectFieldOffset(field);
        }
```

看一下其中的`incrementAndGet()`方法的具体实现：
```
public int incrementAndGet(T obj) {
    return getAndAdd(obj, 1) + 1;
}

//最终调用的还是unsafe.getAndAddInt()方法
public int getAndAdd(T obj, int delta) {
    if (obj == null || obj.getClass() != tclass || cclass != null) fullCheck(obj);
    return unsafe.getAndAddInt(obj, offset, delta);
}
```
下面是一个demo，通过使用`AtomicInteger`和`AtomicIntegerFieldUpdater`检查多线程下是否能正确执行累加操作
```
public class Test{
    public static class Candidate{
        int id;
        volatile int score;
    }

    static AtomicIntegerFieldUpdater<Candidate> atIntegerUpdater
        = AtomicIntegerFieldUpdater.newUpdater(Candidate.class, "score");

    //用于验证分数是否正确
    public static AtomicInteger allScore=new AtomicInteger(0);


    public static void main(String[] args) throws InterruptedException {
        final Candidate stu=new Candidate();
        Thread[] t=new Thread[10000];
        //开启10000个线程
        for(int i = 0 ; i < 10000 ; i++) {
            t[i]=new Thread() {
                public void run() {
                    if(Math.random()>0.4){
                        atIntegerUpdater.incrementAndGet(stu);
                        allScore.incrementAndGet();
                    }
                }
            };
            t[i].start();
        }

        for(int i = 0 ; i < 10000 ; i++) {  t[i].join();}
        System.out.println("最终分数score="+stu.score);
        System.out.println("校验分数allScore="+allScore);
        
    }
}
```
---

## 三. 使用

#### 1. 使用*AtomicStampedReference*解决CAS的ABA问题

上面提到ABA问题，如果多个线程同时更新一个值，假设其中一个线程将多次修改后又将值修改为原值，其他线程能正常更新，但是无法判断这个值是否已经被修改过，即CAS乐观算法只关心最终结果。比如ETCD中，每个属性都有一个版本号，即使值相同我们也能通过版本号来判断是不是最初的原值。

下面通过`AtomicStampedReference`来解决这个问题，通过添加时间戳的方式，下面是此类的一些具体实现：
```
//内部静态类
private static class Pair<T> {
    final T reference;
    final int stamp;
    private Pair(T reference, int stamp) {
        this.reference = reference;
        this.stamp = stamp;
    }
    static <T> Pair<T> of(T reference, int stamp) {
        return new Pair<T>(reference, stamp);
    }
}

private volatile Pair<V> pair;

/**
 * Creates a new {@code AtomicStampedReference} with the given
 * initial values.
 *
 * @param initialRef the initial reference
 * @param initialStamp the initial stamp
 */
public AtomicStampedReference(V initialRef, int initialStamp) {
    pair = Pair.of(initialRef, initialStamp);
}
```
更新时的具体实现：
```
/**
     * Atomically sets the value of both the reference and stamp
     * to the given update values if the
     * current reference is {@code ==} to the expected reference
     * and the current stamp is equal to the expected stamp.
     *
     * @param expectedReference the expected value of the reference
     * @param newReference the new value for the reference
     * @param expectedStamp the expected value of the stamp
     * @param newStamp the new value for the stamp
     * @return {@code true} if successful
     */
    public boolean compareAndSet(V   expectedReference,
                                 V   newReference,
                                 int expectedStamp,
                                 int newStamp) {
        Pair<V> current = pair;
        return
            //对比期望reference和时间戳，相同才会执行casPair()
            //最终仍然是调用Unsafe里的CAS操作
            expectedReference == current.reference &&
            expectedStamp == current.stamp &&
            ((newReference == current.reference &&
              newStamp == current.stamp) ||
             casPair(current, Pair.of(newReference, newStamp)));
    }
```

下面是一个例子：
```
public class Test{
    static AtomicInteger atIn = new AtomicInteger(100);

    //初始化时需要传入一个初始值和初始时间
    static AtomicStampedReference<Integer> atomicStampedR =
        new AtomicStampedReference<Integer>(200,0);


    static Thread t1 = new Thread(new Runnable() {
        public void run() {
            //更新为200
            atIn.compareAndSet(100, 200);
            //更新为100
            atIn.compareAndSet(200, 100);
        }
    });


    static Thread t2 = new Thread(new Runnable() {
        public void run() {
            try {
                TimeUnit.SECONDS.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            boolean flag=atIn.compareAndSet(50,500);
            System.out.println("flag:"+flag+",newValue:"+atIn);
        }
    });


    static Thread t3 = new Thread(new Runnable() {
        public void run() {
            int time=atomicStampedR.getStamp();
            //更新为200
            atomicStampedR.compareAndSet(100, 200,time,time+1);
            //更新为100
            int time2=atomicStampedR.getStamp();
            atomicStampedR.compareAndSet(200, 100,time2,time2+1);
        }
    });


    static Thread t4 = new Thread(new Runnable() {
        public void run() {
            int time = atomicStampedR.getStamp();
            System.out.println("sleep 前 t4 time:"+time);
            try {
                TimeUnit.SECONDS.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            boolean flag=atomicStampedR.compareAndSet(100,500,time,time+1);
            System.out.println("flag:"+flag+",newValue:"+atomicStampedR.getReference());
        }
    });

    public static  void  main(String[] args) throws InterruptedException {
        t1.start();
        t2.start();
        t1.join();
        t2.join();

        t3.start();
        t4.start();
        /**
         * 输出结果:
         flag:true,newValue:500
         sleep 前 t4 time:0
         flag:false,newValue:200
         */
    }
}
```

#### 2. 使用*AtomicReference*实现一个自旋锁
假如有多个线程尝试去获取锁，获取不到锁的线程进入阻塞等待，等待下一次获取锁。我们知道线程的阻塞，唤醒等操作需要操作系统介入，会有用户态和内核态的切换，有很大的性能损耗。那么在某些情况下，我们假设这个线程会获取到锁，只是需要一定时间的等待，那么我喜欢能通过某种方式，不进行用户态和内核态的切换，我们让获取不到锁的线程执行一些空循环，而不是阻塞，这就是自旋锁，执行一定次数的循环后仍然获取不到锁，那么线程才会被挂起。但是如果这个时候线程很多，竞争很激烈，同样会造成CPU时间大量消耗，影响性能。

```
public class Test{
    public static class SpinLock {
        static final int MAX_RETRY = 5000;
        private AtomicReference<Thread> sign =new AtomicReference<Thread>();
        public boolean lock(){
            AtomicInteger lockCount = new AtomicInteger();
            Thread current = Thread.currentThread();
            while(!sign.compareAndSet(null, current)){
                if (lockCount.get() > MAX_RETRY){
                    //hang the thread
                    System.out.println(current.getName() + "Cannot get the lock!");
                    return false;
                }
                lockCount.incrementAndGet();
            }
            System.out.println(current.getName() + "acquired the lock");
            return true;
        }

        public void unlock (){
            Thread current = Thread.currentThread();
            sign.compareAndSet(current, null);
            System.out.println(current.getName() + "release the lock!");
        }
    }

    static class Thread1 implements Runnable{
        SpinLock spinLock;
        public Thread1(SpinLock spinLock) {
            this.spinLock = spinLock;
        }
        public void run() {
            boolean result = spinLock.lock();
            try {
                Thread.sleep(3000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            if (result) {
                spinLock.unlock();
            }
        }
    }
    
    public static void main(String[] args) throws InterruptedException {
        SpinLock spinLock = new SpinLock();
        Thread thread1 = new Thread(new Thread1(spinLock));
        thread1.start();
        Thread.sleep(2000);

        Thread thread2 = new Thread(new Thread1(spinLock));
        thread2.start();

    }
}
```

上面使用了`AtomicInteger`和`AtomicReference`来模拟，`lock()`的期望值是null，将更新为当前线程的引用类型，`unlock()`正好相反。两个线程同时执行获取锁，并且设置超时次数，如果线程1一直不释放锁，线程2尝试一定次数后获取不到锁将退出(模拟挂起)。

