## JVM基础面试题及原理讲解
> 原文出处：http://www.importnew.com/31126.html

> 本文从 JVM 结构入手，介绍了 Java 内存管理、对象创建、常量池等基础知识，对面试中 JVM 相关的基础题目进行了讲解。

<!--目录结构-->
[toc]

## 写在前面（常见面试题）  

> **基本问题**  
1.介绍下 Java 内存区域（运行时数据区）  
2.Java 对象的创建过程（五步，建议能默写出来并且要知道每一步虚拟机做了什么）  
3.对象的访问定位的两种方式（句柄和直接指针两种方式）  

> **拓展问题**  
1.String类和常量池
2.8种基本类型的包装类和常量池

## 1 概述 
> 对于 Java 程序员来说，在虚拟机自动内存管理机制下，不再需要像C/C++程序开发程序员这样为内一个 new 操作去写对应的 delete/free 操作，不容易出现内存泄漏和内存溢出问题。正是因为 Java 程序员把内存控制权利交给 Java 虚拟机，一旦出现内存泄漏和溢出方面的问题，如果不了解虚拟机是怎样使用内存的，那么排查错误将会是一个非常艰巨的任务。

## 2 运行时数据区域 
> Java 虚拟机在执行 Java 程序的过程中会把它管理的内存划分成若干个不同的数据区域。
![avatar](https://github.com/hyblogs/blogs/blob/master/Java_Images/Interview_Base/01.jpg)

> 这些组成部分一些是线程私有的，其他的则是线程共享的。

> *线程私有的*：

> (1).程序计数器.  
> (2).虚拟机栈.  
> (3).本地方法栈.  

> *线程共享的*：

> (1).堆.  
> (2).方法区.  
> (3).直接内存.  

### 2.1 程序计数器
> 程序计数器是一块较小的内存空间，可以看作是当前线程所执行的字节码的行号指示器。字节码解释器工作时通过改变这个计数器的值来选取下一条需要执行的字节码指令，分支、循环、跳转、异常处理、线程恢复等功能都需要依赖这个计数器来完。

> 另外，为了线程切换后能恢复到正确的执行位置，每条线程都需要有一个独立的程序计数器，各线程之间计数器互不影响，独立存储，我们称这类内存区域为“线程私有”的内存。

> **从上面的介绍中我们知道程序计数器主要有两个作用：**

> 1.字节码解释器通过改变程序计数器来依次读取指令，从而实现代码的流程控制，如：顺序执行、选择、循环、异常处理。  
> 2.在多线程的情况下，程序计数器用于记录当前线程执行的位置，从而当线程被切换回来的时候能够知道该线程上次运行到哪儿了。
> **注意**：程序计数器是唯一不会出现 OutOfMemoryError 的内存区域，它的生命周期随着线程的创建而创建，随着线程的结束而死亡。

### 2.2 Java 虚拟机栈
> 与程序计数器一样，Java虚拟机栈也是线程私有的，它的生命周期和线程相同，描述的是 Java 方法执行的内存模型。

> Java 内存可以粗糙的区分为堆内存（Heap）和栈内存（Stack）其中栈就是现在说的虚拟机栈，或者说是虚拟机栈中局部变量表部分。 （实际上，Java虚拟机栈是由一个个栈帧组成，而每个栈帧中都拥有局部变量表、操作数栈、动态链接、方法出口信息）

> 局部变量表主要存放了编译器可知的各种数据类型（boolean、byte、char、short、int、float、long、double）、对象引用（reference类型，它不同于对象本身，可能是一个指向对象起始地址的引用指针，也可能是指向一个代表对象的句柄或其他与此对象相关的位置）。

> **Java 虚拟机栈会出现两种异常：StackOverFlowError 和 OutOfMemoryError。**

> *StackOverFlowError： 若Java虚拟机栈的内存大小不允许动态扩展，那么当线程请求栈的深度超过当前Java虚拟机栈的最大深度的时候，就抛出StackOverFlowError异常。  
> *OutOfMemoryError： 若 Java 虚拟机栈的内存大小允许动态扩展，且当线程请求栈时内存用完了，无法再动态扩展了，此时抛出OutOfMemoryError异常。  

> Java 虚拟机栈也是线程私有的，每个线程都有各自的Java虚拟机栈，而且随着线程的创建而创建，随着线程的死亡而死亡。

### 2.3 本地方法栈
> 和虚拟机栈所发挥的作用非常相似，区别是： 虚拟机栈为虚拟机执行 Java 方法 （也就是字节码）服务，而本地方法栈则为虚拟机使用到的 Native 方法服务。 在 HotSpot 虚拟机中和 Java 虚拟机栈合二为一。

> 本地方法被执行的时候，在本地方法栈也会创建一个栈帧，用于存放该本地方法的局部变量表、操作数栈、动态链接、出口信息。

> 方法执行完毕后相应的栈帧也会出栈并释放内存空间，也会出现 StackOverFlowError 和 OutOfMemoryError 两种异常。

### 2.4 堆
> Java 堆是 Java 虚拟机所管理的内存中最大的一块，Java 堆是所有线程共享的一块内存区域，在虚拟机启动时创建。此内存区域的唯一目的就是存放对象实例，几乎所有的对象实例以及数组都在这里分配内存。

> Java 堆是垃圾收集器管理的主要区域，因此也被称作GC堆（Garbage Collected Heap）.从垃圾回收的角度，由于现在收集器基本都采用分代垃圾收集算法，所以Java堆还可以细分为：新生代和老年代：再细致一点有：Eden空间、From Survivor、To Survivor空间等。进一步划分的目的是更好地回收内存，或者更快地分配内存。
![avatar](https://github.com/hyblogs/blogs/blob/master/Java_Images/Interview_Base/02.png)

> 在 JDK 1.8中移除整个永久代，取而代之的是一个叫元空间（Metaspace）的区域（永久代使用的是JVM的堆内存空间，而元空间使用的是物理内存，直接受到本机的物理内存限制）。

> 推荐阅读：[Java8内存模型——永久代（PermGen）和元空间（Metaspace）](#toc_18)

### 2.5 方法区
> 方法区与 Java 堆一样，是各个线程共享的内存区域，它用于存储已被虚拟机加载的类信息、常量、静态变量、即时编译器编译后的代码等数据。虽然Java虚拟机规范把方法区描述为堆的一个逻辑部分，但是它却有一个别名叫做 Non-Heap（非堆），目的应该是与 Java 堆区分开来。

> HotSpot 虚拟机中方法区也常被称为 “永久代”，本质上两者并不等价。仅仅是因为 HotSpot 虚拟机设计团队用永久代来实现方法区而已，这样 HotSpot 虚拟机的垃圾收集器就可以像管理 Java 堆一样管理这部分内存了。但是这并不是一个好主意，因为这样更容易遇到内存溢出问题。

> 相对而言，垃圾收集行为在这个区域是比较少出现的，但并非数据进入方法区后就“永久存在”了。

### 2.6 运行时常量池
> 运行时常量池是方法区的一部分。Class 文件中除了有类的版本、字段、方法、接口等描述信息外，还有常量池信息（用于存放编译期生成的各种字面量和符号引用）
既然运行时常量池是方法区的一部分，自然受到方法区内存的限制，当常量池无法再申请到内存时会抛出 OutOfMemoryError 异常。

> JDK1.7及之后版本的 JVM 已经将运行时常量池从方法区中移了出来，在 Java 堆（Heap）中开辟了一块区域存放运行时常量池。
![avatar](https://github.com/hyblogs/blogs/blob/master/Java_Images/Interview_Base/03.jpg)


> 推荐阅读：[Java 中几种常量池的区分](#toc_23)

### 2.7 直接内存
> 直接内存并不是虚拟机运行时数据区的一部分，也不是虚拟机规范中定义的内存区域，但是这部分内存也被频繁地使用。而且也可能导致OutOfMemoryError异常出现。

> JDK1.4中新加入的 NIO(New Input/Output) 类，引入了一种基于通道（Channel） 与缓存区（Buffer） 的 I/O 方式，它可以直接使用Native函数库直接分配堆外内存，然后通过一个存储在 Java 堆中的 DirectByteBuffer 对象作为这块内存的引用进行操作。这样就能在一些场景中显著提高性能，因为避免了在 Java 堆和 Native 堆之间来回复制数据。

> 本机直接内存的分配不会收到 Java 堆的限制，但是，既然是内存就会受到本机总内存大小以及处理器寻址空间的限制。

## 3 HotSpot 虚拟机对象探秘
> 通过上面的介绍我们大概知道了虚拟机的内存情况，下面我们来详细的了解一下 HotSpot 虚拟机在 Java 堆中对象分配、布局和访问的全过程。

### 3.1 对象的创建
> 下图便是 Java 对象的创建过程，我建议最好是能默写出来，并且要掌握每一步在做什么。
![avatar](https://github.com/hyblogs/blogs/blob/master/Java_Images/Interview_Base/04.jpg)


> _Java创建对象过程_  
> **1. 类加载检查：** 虚拟机遇到一条 new 指令时，首先将去检查这个指令的参数是否能在常量池中定位到这个类的符号引用，并且检查这个符号引用代表的类是否已被加载过、解析和初始化过。如果没有，那必须先执行相应的类加载过程。

> **2. 分配内存：** 在类加载检查通过后，接下来虚拟机将为新生对象分配内存。对象所需的内存大小在类加载完成后便可确定，为对象分配空间的任务等同于把一块确定大小的内存从 Java 堆中划分出来。分配方式有 “指针碰撞” 和 “空闲列表” 两种，选择哪种分配方式由 Java 堆是否规整决定，而Java堆是否规整又由所采用的垃圾收集器是否带有压缩整理功能决定。

> **内存分配的两种方式：**（补充内容，需要掌握）

> 选择以上两种方式中的哪一种，取决于 Java 堆内存是否规整。而 Java 堆内存是否规整，取决于 GC 收集器的算法是”标记-清除”，还是”标记-整理”（也称作”标记-压缩”），值得注意的是，复制算法内存也是规整的。
![avatar](https://github.com/hyblogs/blogs/blob/master/Java_Images/Interview_Base/05.png)


> **内存分配并发问题**（补充内容，需要掌握）

> 在创建对象的时候有一个很重要的问题，就是线程安全，因为在实际开发过程中，创建对象是很频繁的事情，作为虚拟机来说，必须要保证线程是安全的，通常来讲，虚拟机采用两种方式来保证线程安全：

> **CAS+失败重试**： CAS 是乐观锁的一种实现方式。所谓乐观锁就是，每次不加锁而是假设没有冲突而去完成某项操作，如果因为冲突失败就重试，直到成功为止。虚拟机采用 CAS 配上失败重试的方式保证更新操作的原子性。  
> **TLAB**： 为每一个线程预先在 Eden 区分配一块内存。JVM 在给线程中的对象分配内存时，首先在 TLAB 分配，当对象大于TLAB 中的剩余内存或 TLAB 的内存已用尽时，再采用上述的 CAS 进行内存分配。  

> **3. 初始化零值：** 内存分配完成后，虚拟机需要将分配到的内存空间都初始化为零值（不包括对象头），这一步操作保证了对象的实例字段在 Java 代码中可以不赋初始值就直接使用，程序能访问到这些字段的数据类型所对应的零值。

> **4. 设置对象头：** 初始化零值完成之后，虚拟机要对对象进行必要的设置，例如这个对象是那个类的实例、如何才能找到类的元数据信息、对象的哈希吗、对象的 GC 分代年龄等信息。 这些信息存放在对象头中。 另外，根据虚拟机当前运行状态的不同，如是否启用偏向锁等，对象头会有不同的设置方式。

> **5. 执行 init 方法：** 在上面工作都完成之后，从虚拟机的视角来看，一个新的对象已经产生了，但从 Java 程序的视角来看，对象创建才刚开始，<init> 方法还没有执行，所有的字段都还为零。所以一般来说，执行 new 指令之后会接着执行 <init> 方法，把对象按照程序员的意愿进行初始化，这样一个真正可用的对象才算完全产生出来。

### 3.2 对象的内存布局  
> 在 Hotspot 虚拟机中，对象在内存中的布局可以分为3块区域：对象头、实例数据和对齐填充。

> Hotspot虚拟机的对象头包括两部分信息，第一部分用于存储对象自身的自身运行时数据（哈希码、GC分代年龄、锁状态标志等等），另一部分是类型指针，即对象指向它的类元数据的指针，虚拟机通过这个指针来确定这个对象是哪个类的实例。

> 实例数据部分是对象真正存储的有效信息，也是在程序中所定义的各种类型的字段内容。

> 对齐填充部分不是必然存在的，也没有什么特别的含义，仅仅起占位作用。 因为 Hotspot 虚拟机的自动内存管理系统要求对象起始地址必须是8字节的整数倍，换句话说就是对象的大小必须是8字节的整数倍。而对象头部分正好是8字节的倍数（1倍或2倍），因此，当对象实例数据部分没有对齐时，就需要通过对齐填充来补全。

### 3.3 对象的访问定位  
> 建立对象就是为了使用对象，我们的Java程序通过栈上的 reference 数据来操作堆上的具体对象。对象的访问方式有虚拟机实现而定，目前主流的访问方式有使用句柄和直接指针两种：

> **1. 句柄：** 如果使用句柄的话，那么 Java 堆中将会划分出一块内存来作为句柄池，reference 中存储的就是对象的句柄地址，而句柄中包含了对象实例数据与类型数据各自的具体地址信息。
![avatar](https://github.com/hyblogs/blogs/blob/master/Java_Images/Interview_Base/06.png)


> 上图是通过句柄访问对象

> **2. 直接指针：** 如果使用直接指针访问，那么 Java 堆对象的布局中就必须考虑如何放置访问类型数据的相关信息，而 reference 中存储的直接就是对象的地址。
![avatar](https://github.com/hyblogs/blogs/blob/master/Java_Images/Interview_Base/07.png)


> 上图是通过直接指针访问对象

> 这两种对象访问方式各有优势。使用句柄来访问的最大好处是 reference 中存储的是稳定的句柄地址，在对象被移动时只会改变句柄中的实例数据指针，而 reference 本身不需要修改。使用直接指针访问方式最大的好处就是速度快，它节省了一次指针定位的时间开销。

## 4 重点补充内容  

### 4.1 String 类和常量池  
> **1 String 对象的两种创建方式**

	String str1 = "abcd";
	String str2 = new String("abcd");
	System.out.println(str1==str2);//false
	
> 这两种不同的创建方法是有差别的，第一种方式是在常量池中拿对象，第二种方式是直接在堆内存空间创建一个新的对象。  
![avatar](https://github.com/hyblogs/blogs/blob/master/Java_Images/Interview_Base/08.jpg)


> **记住**：只要使用了 new 方法，就需要创建新的对象。

> **2 String 类型的常量池比较特殊。它的主要使用方法有两种：**

> + 直接使用双引号声明出来的 String 对象会直接存储在常量池中。  
> + 如果不是用双引号声明的 String 对象，可以使用 String 提供的 intern 方法。String.intern() 是一个 Native 方法，它的作用是：如果运行时常量池中已经包含一个等于此 String 对象内容的字符串，则返回常量池中该字符串的引用；如果没有，则在常量池中创建与此 String 内容相同的字符串，并返回常量池中创建的字符串的引用。
	
	String s1 = new String("计算机");
	String s2 = s1.intern();
	String s3 = "计算机";
	System.out.println(s2);//计算机
	System.out.println(s1 == s2);//false，因为一个是堆内存中的String对象,一个是常量池中	的String对象
	System.out.println(s3 == s2);//true，因为两个都是常量池中的String对象
	
> **3 String 字符串拼接**

	String str1 = "str";
	String str2 = "ing";
	 
	String str3 = "str" + "ing";//常量池中的对象
	String str4 = str1 + str2; //在堆上创建的新的对象     
	String str5 = "string";//常量池中的对象
	System.out.println(str3 == str4);//false
	System.out.println(str3 == str5);//true
	System.out.println(str4 == str5);//false
	
![avatar](https://github.com/hyblogs/blogs/blob/master/Java_Images/Interview_Base/09.png)

> 尽量避免多个字符串拼接，因为这样会重新创建对象。如果需要改变字符串的话，可以使用 StringBuilder(非线程安全，速度快) 或者 StringBuffer(线程安全，速度慢)。_(在不要求线程安全的情况下使用StringBuilder，在要求线程安全的情况下使用StringBuffer)_

> 问：

	String s1 = new String("abc"); // 这句话创建了几个对象？
	
> 答：
> 	创建了两个对象。

> 验证：
	
	String s1 = new String("abc");// 堆内存的地值值
	String s2 = "abc";
	System.out.println(s1 == s2);// 输出false,因为一个是堆内存，一个是常量池的内存，故两	者是不同的。
	System.out.println(s1.equals(s2));// 输出true
	
> **结果：**

	false
	true
	
> **解释：**

> 先有字符串 “abc” 放入常量池，然后 new 了一份字符串 “abc” 放入 Java 堆（字符串常量 “abc” 在编译期就已经确定放入常量池，而 Java 堆上的 “abc” 是在运行期初始化阶段才确定），然后 Java 栈的 s1 指向 Java 堆上的 “abc”。

### 4.2 8种基本类型的包装类和常量池
> Java 基本类型的包装类的大部分都实现了常量池技术，即 Byte、Short、Integer、Long、Character、Boolean；这5种包装类默认创建了数值 [-128，127] 的相应类型的缓存数据，但是超出此范围仍然会去创建新的对象。
两种浮点数类型的包装类 Float、Double 并没有实现常量池技术。
	
	Integer i1 = 33;
	Integer i2 = 33;
	System.out.println(i1 == i2);// 输出true
	Integer i11 = 333;
	Integer i22 = 333;
	System.out.println(i11 == i22);// 输出false
	Double i3 = 1.2;
	Double i4 = 1.2;
	System.out.println(i3 == i4);// 输出false
	
> Integer 缓存源代码：
	
	/**
	 *此方法将始终缓存-128到127（包括端点）范围内的值，并可以缓存此范围之外的其他值。
	 */
	public static Integer valueOf(int i) {
	    if (i >= IntegerCache.low && i <= IntegerCache.high)
	        return IntegerCache.cache[i + (-IntegerCache.low)];
	    return new Integer(i);
	}
	
> 应用场景：

	Integer i1=40；//Java 在编译的时候会直接将代码封装成 Integer 	i1=Integer.valueOf(40);从而使用常量池中的对象。
	
	Integer i1 = new Integer(40); //这种情况下会创建新的对象。
	
	Integer i1 = 40;
	Integer i2 = new Integer(40);
	System.out.println(i1==i2); //输出false

> Integer 比较（==）更丰富的一个例子：

	Integer i1 = 40;
	Integer i2 = 40;
	Integer i3 = 0;
	Integer i4 = new Integer(40);
	Integer i5 = new Integer(40);
	Integer i6 = new Integer(0);
	 
	System.out.println("i1=i2   " + (i1 == i2));
	System.out.println("i1=i2+i3   " + (i1 == i2 + i3));
	System.out.println("i1=i4   " + (i1 == i4));
	System.out.println("i4=i5   " + (i4 == i5));
	System.out.println("i4=i5+i6   " + (i4 == i5 + i6));   
	System.out.println("40=i5+i6   " + (40 == i5 + i6));

> **结果：**
	
	i1=i2   true
	i1=i2+i3   true
	i1=i4   false
	i4=i5   false
	i4=i5+i6   true
	40=i5+i6   true
> **解释：**  
语句 i4 == i5 + i6，因为 + 这个操作符不适用于 Integer 对象，首先 i5 和 i6 进行自动拆箱操作，进行数值相加，即 i4 == 40。然后Integer对象无法与数值进行直接比较，所以i4自动拆箱转为int值40，最终这条语句转为40 == 40进行数值比较。


</p>


> ============================== 我是友好分割线 ==================================

<!--文中链接地址对应展示数据-->
> **以下部分是文中链接展示🔗**

## Java8内存模型—永久代(PermGen)和元空间(Metaspace)
> 本章节内容摘自：http://www.cnblogs.com/paddix/p/5309550.html

### 一、JVM 内存模型(运行时数据区)

> 　　根据 JVM 规范，JVM 内存共分为虚拟机栈、堆、方法区、程序计数器、本地方法栈五个部分。
![avatar](https://github.com/hyblogs/blogs/blob/master/Java_Images/Interview_Base/10.png)


> 　　**1、虚拟机栈**：每个线程有一个私有的栈，随着线程的创建而创建。栈里面存着的是一种叫“栈帧”的东西，每个方法会创建一个栈帧，栈帧中存放了局部变量表（基本数据类型和对象引用）、操作数栈、方法出口等信息。栈的大小可以固定也可以动态扩展。当栈调用深度大于JVM所允许的范围，会抛出StackOverflowError的错误，不过这个深度范围不是一个恒定的值，我们通过下面这段程序可以测试一下这个结果：

> 栈溢出测试源码：
	
	package com.paddx.test.memory;
	 
	public class StackErrorMock {
	    private static int index = 1;
	 
	    public void call(){
	        index++;
	        call();
	    }
	 
	    public static void main(String[] args) {
	        StackErrorMock mock = new StackErrorMock();
	        try {
	            mock.call();
	        }catch (Throwable e){
	            System.out.println("Stack deep : "+index);
	            e.printStackTrace();
	        }
	    }
	}
> 代码段 1

> 运行三次，可以看出每次栈的深度都是不一样的，输出结果如下:
![avatar](https://github.com/hyblogs/blogs/blob/master/Java_Images/Interview_Base/11.png)


> 至于红色框里的值是怎么出来的，就需要深入到 JVM 的源码中才能探讨，这里不作详细阐述。

> 虚拟机栈除了上述错误外，还有另一种错误，那就是当申请不到空间时，会抛出 OutOfMemoryError。这里有一个小细节需要注意，catch 捕获的是 Throwable，而不是 Exception。因为 StackOverflowError 和 OutOfMemoryError 都不属于 Exception 的子类。

> 　　**2、本地方法栈**：

> 　　这部分主要与虚拟机用到的 Native 方法相关，一般情况下， Java 应用程序员并不需要关心这部分的内容。

> 　　**3、PC 寄存器**：

> 　　PC 寄存器，也叫程序计数器。JVM支持多个线程同时运行，每个线程都有自己的程序计数器。倘若当前执行的是 JVM 的方法，则该寄存器中保存当前执行指令的地址；倘若执行的是native 方法，则PC寄存器中为空。

> 　　**4、堆**

> 　　堆内存是 JVM 所有线程共享的部分，在虚拟机启动的时候就已经创建。所有的对象和数组都在堆上进行分配。这部分空间可通过 GC 进行回收。当申请不到空间时会抛出 OutOfMemoryError。下面我们简单的模拟一个堆内存溢出的情况：
	
	package com.paddx.test.memory;
	 
	import java.util.ArrayList;
	import java.util.List;
	 
	public class HeapOomMock {
	    public static void main(String[] args) {
	        List<byte[]> list = new ArrayList<byte[]>();
	        int i = 0;
	        boolean flag = true;
	        while (flag){
	            try {
	                i++;
	                list.add(new byte[1024 * 1024]);//每次增加一个1M大小的数组对象
	            }catch (Throwable e){
	                e.printStackTrace();
	                flag = false;
	                System.out.println("count="+i);//记录运行的次数
	            }
	        }
	    }
	}
> 代码段 2

> 运行上述代码，输出结果如下：　　
![avatar](https://github.com/hyblogs/blogs/blob/master/Java_Images/Interview_Base/12.png)

<font color=red>
注意，这里我指定了堆内存的大小为16M，所以这个地方显示的count=14（这个数字不是固定的），至于为什么会是14或其他数字，需要根据 GC 日志来判断。
</font>
> 　　**5、方法区**：

> 　　方法区也是所有线程共享。主要用于存储类的信息、常量池、方法数据、方法代码等。方法区逻辑上属于堆的一部分，但是为了与堆进行区分，通常又叫“非堆”。 

### 二、PermGen（永久代）

> 　　绝大部分 Java 程序员应该都见过 "java.lang.OutOfMemoryError: PermGen space "这个异常。这里的 “PermGen space”其实指的就是方法区。不过方法区和“PermGen space”又有着本质的区别。前者是 JVM 的规范，而后者则是 JVM 规范的一种实现，并且只有 HotSpot 才有 “PermGen space”，而对于其他类型的虚拟机，如 JRockit（Oracle）、J9（IBM） 并没有“PermGen space”。由于方法区主要存储类的相关信息，所以对于动态生成类的情况比较容易出现永久代的内存溢出。最典型的场景就是，在 jsp 页面比较多的情况，容易出现永久代内存溢出。我们现在通过动态生成类来模拟 “PermGen space”的内存溢出：
	
	package com.paddx.test.memory;
	 
	public class Test {
	
	}
>  代码段 3

	package com.paddx.test.memory;
	 
	import java.io.File;
	import java.net.URL;
	import java.net.URLClassLoader;
	import java.util.ArrayList;
	import java.util.List;
	 
	public class PermGenOomMock{
	    public static void main(String[] args) {
	        URL url = null;
	        List<ClassLoader> classLoaderList = new ArrayList<ClassLoader>();
	        try {
	            url = new File("/tmp").toURI().toURL();
	            URL[] urls = {url};
	            while (true){
	                ClassLoader loader = new URLClassLoader(urls);
	                classLoaderList.add(loader);
	                loader.loadClass("com.paddx.test.memory.Test");
	            }
	        } catch (Exception e) {
	            e.printStackTrace();
	        }
	    }
	}
> 代码段 4

> 运行结果如下：
![avatar](https://github.com/hyblogs/blogs/blob/master/Java_Images/Interview_Base/13.png)

> 　　本例中使用的 JDK 版本是 1.7，指定的 PermGen 区的大小为 8M。通过每次生成不同URLClassLoader对象来加载Test类，从而生成不同的类对象，这样就能看到我们熟悉的 "java.lang.OutOfMemoryError: PermGen space " 异常了。这里之所以采用 JDK 1.7，是因为在 JDK 1.8 中， HotSpot 已经没有 “PermGen space”这个区间了，取而代之是一个叫做 Metaspace（元空间） 的东西。下面我们就来看看 Metaspace 与 PermGen space 的区别。

### 三、Metaspace（元空间）

> 　　其实，移除永久代的工作从JDK1.7就开始了。JDK1.7中，存储在永久代的部分数据就已经转移到了Java Heap或者是 Native Heap。但永久代仍存在于JDK1.7中，并没完全移除，譬如符号引用(Symbols)转移到了native heap；字面量(interned strings)转移到了java heap；类的静态变量(class statics)转移到了java heap。我们可以通过一段程序来比较 JDK 1.6 与 JDK 1.7及 JDK 1.8 的区别，以字符串常量为例：
	
	package com.paddx.test.memory;
	 
	import java.util.ArrayList;
	import java.util.List;
	 
	public class StringOomMock {
	    static String  base = "string";
	    public static void main(String[] args) {
	        List<String> list = new ArrayList<String>();
	        for (int i=0;i< Integer.MAX_VALUE;i++){
	            String str = base + base;
	            base = str;
	            list.add(str.intern());
	        }
	    }
	}
> 这段程序以2的指数级不断的生成新的字符串，这样可以比较快速的消耗内存。我们通过 JDK 1.6、JDK 1.7 和 JDK 1.8 分别运行：

> JDK 1.6 的运行结果：
![avatar](https://github.com/hyblogs/blogs/blob/master/Java_Images/Interview_Base/14.png)


> JDK 1.7的运行结果：
![avatar](https://github.com/hyblogs/blogs/blob/master/Java_Images/Interview_Base/15.png)


> JDK 1.8的运行结果：
![avatar](https://github.com/hyblogs/blogs/blob/master/Java_Images/Interview_Base/16.png)


> 　　从上述结果可以看出，JDK 1.6下，会出现“PermGen Space”的内存溢出，而在 JDK 1.7和 JDK 1.8 中，会出现堆内存溢出，并且 JDK 1.8中 PermSize 和 MaxPermGen 已经无效。因此，可以大致验证 JDK 1.7 和 1.8 将字符串常量由永久代转移到堆中，并且 JDK 1.8 中已经不存在永久代的结论。现在我们看看元空间到底是一个什么东西？

> 　　元空间的本质和永久代类似，都是对JVM规范中方法区的实现。不过元空间与永久代之间最大的区别在于：元空间并不在虚拟机中，而是使用本地内存。因此，默认情况下，元空间的大小仅受本地内存限制，但可以通过以下参数来指定元空间的大小：

> 　　-XX:MetaspaceSize，初始空间大小，达到该值就会触发垃圾收集进行类型卸载，同时GC会对该值进行调整：如果释放了大量的空间，就适当降低该值；如果释放了很少的空间，那么在不超过MaxMetaspaceSize时，适当提高该值。  
> 　　-XX:MaxMetaspaceSize，最大空间，默认是没有限制的。

> 　　除了上面两个指定大小的选项以外，还有两个与 GC 相关的属性：
> 　　-XX:MinMetaspaceFreeRatio，在GC之后，最小的Metaspace剩余空间容量的百分比，减少为分配空间所导致的垃圾收集.  
> 　　-XX:MaxMetaspaceFreeRatio，在GC之后，最大的Metaspace剩余空间容量的百分比，减少为释放空间所导致的垃圾收集.  

> 现在我们在 JDK 8下重新运行一下代码段 4，不过这次不再指定 PermSize 和 MaxPermSize。而是指定 MetaSpaceSize 和 MaxMetaSpaceSize的大小。输出结果如下：
![avatar](https://github.com/hyblogs/blogs/blob/master/Java_Images/Interview_Base/17.png)


> 从输出结果，我们可以看出，这次不再出现永久代溢出，而是出现了元空间的溢出。

### 四、总结

> 　　通过上面分析，大家应该大致了解了 JVM 的内存划分，也清楚了 JDK 8 中永久代向元空间的转换。不过大家应该都有一个疑问，就是为什么要做这个转换？所以，最后给大家总结以下几点原因：

> 　　1、字符串存在永久代中，容易出现性能问题和内存溢出。

> 　　2、类及方法的信息等比较难确定其大小，因此对于永久代的大小指定比较困难，太小容易出现永久代溢出，太大则容易导致老年代溢出。

> 　　3、永久代会为 GC 带来不必要的复杂度，并且回收效率偏低。

> 　　4、Oracle 可能会将HotSpot 与 JRockit 合二为一。




## Java中几种常量池的区分
> 本章节内容摘自：http://tangxman.github.io/2015/07/27/the-difference-of-java-string-pool/

> 在java的内存分配中，经常听到很多关于常量池的描述，我开始看的时候也是看的很模糊，网上五花八门的说法简直太多了，最后查阅各种资料，终于算是差不多理清了，很多网上说法都有问题，笔者尝试着来区分一下这几个概念。

### 1.全局字符串池（string pool也有叫做string literal pool）

> 全局字符串池里的内容是在类加载完成，经过验证，准备阶段之后在堆中生成字符串对象实例，然后将该字符串对象实例的引用值存到string pool中（**记住：string pool中存的是引用值而不是具体的实例对象，具体的实例对象是在堆中开辟的一块空间存放的。**）。  
在HotSpot VM里实现的string pool功能的是一个StringTable类，它是一个哈希表，里面存的是驻留字符串(也就是我们常说的用双引号括起来的)的引用（而不是驻留字符串实例本身），也就是说在堆中的某些字符串实例被这个StringTable引用之后就等同被赋予了”驻留字符串”的身份。这个StringTable在每个HotSpot VM的实例只有一份，被所有的类共享。

### 2.class文件常量池（class constant pool）

> 我们都知道，class文件中除了包含类的版本、字段、方法、接口等描述信息外，还有一项信息就是常量池(constant pool table)，用于存放编译器生成的各种字面量(Literal)和符号引用(Symbolic References)。  
字面量就是我们所说的常量概念，如文本字符串、被声明为final的常量值等。  
符号引用是一组符号来描述所引用的目标，符号可以是任何形式的字面量，只要使用时能无歧义地定位到目标即可（它与直接引用区分一下，直接引用一般是指向方法区的本地指针，相对偏移量或是一个能间接定位到目标的句柄）。  

> 一般包括下面三类常量：
>
+ 类和接口的全限定名
+ 字段的名称和描述符
+ 方法的名称和描述符

> 常量池的每一项常量都是一个表，一共有如下表所示的11种各不相同的表结构数据，这每个表开始的第一位都是一个字节的标志位（取值1-12），代表当前这个常量属于哪种常量类型。
![avatar](https://github.com/hyblogs/blogs/blob/master/Java_Images/Interview_Base/18.png)

每种不同类型的常量类型具有不同的结构，具体的结构本文就先不叙述了，本文着重区分这三个常量池的概念（读者若想深入了解每种常量类型的数据结构可以查看《深入理解java虚拟机》第六章的内容）。

### 3.运行时常量池（runtime constant pool）

> 当java文件被编译成class文件之后，也就是会生成我上面所说的class常量池，那么运行时常量池又是什么时候产生的呢？

> jvm在执行某个类的时候，必须经过加载、连接、初始化，而连接又包括验证、准备、解析三个阶段。而当类加载到内存中后，jvm就会将class常量池中的内容存放到运行时常量池中，由此可知，运行时常量池也是每个类都有一个。在上面我也说了，class常量池中存的是字面量和符号引用，也就是说他们存的并不是对象的实例，而是对象的符号引用值。而经过解析（resolve）之后，也就是把符号引用替换为直接引用，解析的过程会去查询全局字符串池，也就是我们上面所说的StringTable，以保证运行时常量池所引用的字符串与全局字符串池中所引用的是一致的。

> 举个实例来说明一下:
	
	String str1 = "abc";
	String str2 = new String("def");
	String str3 = "abc";
	String str4 = str2.intern();
	String str5 = "def";
	System.out.println(str1 == str3);//true
	System.out.println(str2 == str4);//false
	System.out.println(str4 == str5);//true
	
> 上面程序的首先经过编译之后，在该类的class常量池中存放一些符号引用，然后类加载之后，将class常量池中存放的符号引用转存到运行时常量池中，然后经过验证，准备阶段之后，在堆中生成驻留字符串的实例对象（也就是上例中str1所指向的”abc”实例对象），然后将这个对象的引用存到全局String Pool中，也就是StringTable中，最后在解析阶段，要把运行时常量池中的符号引用替换成直接引用，那么就直接查询StringTable，保证StringTable里的引用值与运行时常量池中的引用值一致，大概整个过程就是这样了。

> 回到上面的那个程序，现在就很容易解释整个程序的内存分配过程了，首先，在堆中会有一个”abc”实例，全局StringTable中存放着”abc”的一个引用值，然后在运行第二句的时候会生成两个实例，一个是”def”的实例对象，并且StringTable中存储一个”def”的引用值，还有一个是new出来的一个”def”的实例对象，与上面那个是不同的实例，当在解析str3的时候查找StringTable，里面有”abc”的全局驻留字符串引用，所以str3的引用地址与之前的那个已存在的相同，str4是在运行的时候调用intern()函数，返回StringTable中”def”的引用值，如果没有就将str2的引用值添加进去，在这里，StringTable中已经有了”def”的引用值了，所以返回上面在new str2的时候添加到StringTable中的 “def”引用值，最后str5在解析的时候就也是指向存在于StringTable中的”def”的引用值，那么这样一分析之后，下面三个打印的值就容易理解了。

> **总结**  
1.全局常量池在每个VM中只有一份，存放的是字符串常量的引用值。  
2.class常量池是在编译的时候每个class都有的，在编译阶段，存放的是常量的符号引用。  
3.运行时常量池是在类加载完成之后，将每个class常量池中的符号引用值转存到运行时常量池中，也就是说，每个class都有一个运行时常量池，类在解析之后，将符号引用替换成直接引用，与全局常量池中的引用值保持一致。  
