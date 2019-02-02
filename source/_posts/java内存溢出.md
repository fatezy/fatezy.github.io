---
title: java内存溢出

date: 2016-05-10 21:54:17

categories: 

- jvm

tags: [java,jvm]

---





#### java堆溢出



java堆用来存储对象实例，只有不断地创建对象，并保证GC Roots到对象之间有可到达路径来避免垃圾回收机制

，那么当对象的数量达到最大堆的容量后就会产生内存溢出异常。



VM arguments:

-verbose:gc -Xms20M -Xmx20M -Xmn10M -XX:+PrintGCDetails -XX:SurvivorRatio=8

设置堆的最大内存为20MB，最小内存也是20MB

<!-- more -->

```java 

public class HeapOOM {

    static class OOMObject{



    }



    public static void main(String[] args) {

        List<OOMObject> list = new ArrayList<OOMObject>();

        while (true){

            list.add(new OOMObject());



        }

    }

}



```





*** 

Exception in thread "main" java.lang.OutOfMemoryError: Java heap space

	at java.util.Arrays.copyOf(Arrays.java:3210)

	at java.util.Arrays.copyOf(Arrays.java:3181)

	at java.util.ArrayList.grow(ArrayList.java:261)

	at java.util.ArrayList.ensureExplicitCapacity(ArrayList.java:235)

	at java.util.ArrayList.ensureCapacityInternal(ArrayList.java:227)

	at java.util.ArrayList.add(ArrayList.java:458)

	at OutOfMemory.HeapOOM.main(HeapOOM.java:19)

***



#### 虚拟机栈与本地方法栈溢出

注：

	* 平时所指的栈就是这一部分，或者说是虚拟机栈中的局部变量表

	* 本地方法栈与虚拟机栈类似，只不过本地方法栈为虚拟机使用的Native方法服务，即可能是别的语言实现的方法



* 如果请求的栈深度大于虚拟机所允许的最大深度，将抛出StackOverflowError异常

* 如果虚拟机在扩展栈的时候无法申请到足够的内存空间，则抛出OutOfMemoryError异常



虽然看似严谨，其实并没有卵用



VM arguments: -Xss128k

注： 其实不设的默认的虚拟机栈空间也并不大

	



``` java

public class JavaVMStackSOF {

	private int stackLength = 1;

	

	

	public void stackLeak(){

		stackLength++;

		stackLeak();

	}

	public static void main(String[] args) {

		JavaVMStackSOF oom = new JavaVMStackSOF();

		try{

		oom.stackLeak();

		}catch(Throwable e){

			System.out.println("stack length"+ oom.stackLength);

			throw e;

		}

	

	}

}

```



***

stack length987

Exception in thread "main" java.lang.StackOverflowError

	at OutOfMemory.JavaVMStackSOF.stackLeak(JavaVMStackSOF.java:17)

	at OutOfMemory.JavaVMStackSOF.stackLeak(JavaVMStackSOF.java:18)

	at OutOfMemory.JavaVMStackSOF.stackLeak(JavaVMStackSOF.java:18)

***



#### 方法区与运行时常量溢出



jdk1.7开始逐步去除永久代



String.intern()是一个Native方法，他的作用是： 如果字符串常量包含一个等于此对象的字符串，则返回这个字符串的String对象；否则添加至常量池



VM：-XX:PermSize=3M -XX:MaxPermSize=3M



```java

public static void main(String[] args) {

		for(String arg: args){

			System.out.println(arg);

		}

		//避免回收常量池

		List<String> list = new ArrayList<String>();

		int i = 10000000;

		while(true){

			//try{

			list.add(String.valueOf(i++).intern());

		//	}catch(Throwable e){

			//	System.out.println("添加的常量数量："+ i);

			//	throw e;

			//}

		}

	}

```



1. java1.8并不会抛出异常

2. java1.7之前会抛出OutOfMemoryError: 由于常量池分配于永久代，可以通过Vm配置限制方法区的内存现在常量池的内存



* JDK7中符号表被移动到 Native Heap中，字符串常量和类引用被移动到 Java Heap中。

* 在 JDK8 中，永久代已完全被元空间(Meatspace)所取代。（JDK8 HotSpot JVM 使用本地内存来存储类元数据信息并称之为：元空间（Metaspace）；）





##### 在jdk8中限制堆的大小会抛出如下异常

##### java7也是这个结果，不知道是我编译器没配好，还是别的原因



---

Exception in thread "main" java.lang.OutOfMemoryError: GC overhead limit exceeded

	at java.lang.Integer.toString(Integer.java:401)

	at java.lang.String.valueOf(String.java:3099)

	at OutOfMemory.RuntimeConstantPoolOOM.main(RuntimeConstantPoolOOM.java:26)

Picked up JAVA_TOOL_OPTIONS: -Dfile.encoding=UTF-8



---



#### 本机直接内存泄漏

* 设置直接内存大小

VM:-Xmx20M -XX:MaxDirectMemorySize=10M



```java

public class DirectMemoryOOM {



    private static final int _1MB = 1024 * 1024;



    

    public static void main(String[] args) throws IllegalArgumentException,

        IllegalAccessException {

      // TODO Auto-generated method stub

      Field unsafeField = Unsafe.class.getDeclaredFields()[0];

      unsafeField.setAccessible(true);

      Unsafe unsafe = (Unsafe) unsafeField.get(null);

      

      while(true){

        unsafe.allocateMemory(_1MB);

      }

    }



  }





```





