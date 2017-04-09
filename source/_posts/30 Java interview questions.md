---
layout: post
title: java面试中高频的30个问题
tags: [java]
category: 编程
---

在网上看到了这篇文章[30个最常被问的java面试题](http://javarevisited.blogspot.sg/2014/02/top-30-java-phone-interview-questions.html)发现这些问题和答案还挺有价值的，将其翻译总结一下。有些地方觉得英语更容易表示就直接引用了原文

1. 为什么String不可变

	安全性和效率两方面的考虑，String常量在堆中的常量池中。

2. 抽象类可以有构造方法吗
	
	可以，默认编译器会生成的，因为子类要调用父类的构造方法。

3. Object中哪两个函数被重写是为了hashtable

	equal和hashCode

4. sleep，yield和wait的区别
     
   wait是Object的函数，yield和sleep是Thread的函数。wait在同步中调用，释放所占用资源给其他线程，被notify唤醒。sleep只是将当前线程暂停一会儿，不会释放资源。yield只是释放当前线程的cpu资源，交给线程调度器来决定谁将得到cpu

5. List和Set的区别
      
	List有重复，Set没有重复；List有序，Set无序，但是有SortedSet，Set用equals比较，SortedSet用comparable和comparator比较；ArrayList, Vector and LinkedList实现List，HashSet和TreeSet实现Set

6. Java怎么构建不可变的类，如String
     构造函数执行之后对象的状态不再可变，任何改变都会导致产生一个新的对象；所有的域都应该是final的；对象在构造的时候一定不要把它的引用泄漏出去；class声明成final的，防止继承的类改变它。

7. Java用什么来表示现金或者财务的计算
	
	用long或者BigDecimal，不要用float或者double，因为不精确。BigDecimal用字符串来初始化。

8. 抽象类和interface用哪个

     用抽象类会失去唯一的继承的机会，比如Runnable和Thread的选择；抽象类中可以有具体的方法，它更适合在演进或者不断变化的类，在抽象类中改变了，所有的子类都会影响到，接口需要改变所有的子类；Interface很适合作为一种接口来定义；抽象类是IS-A的层次 ，接口是 CAN-DO-THIS 的层次；定义一个飞的接口，鸟类可以用，飞机也可以用；List is declared as interface and extends Collection and Iterable interface and AbstractList is an abstract class which implements List. AbstractList provides skeletal implementation of List interface；

9. HashTable和HashMap的区别
      
	HashTable是线程安全的，HashMap不是，需要额外的同步Map m = Collections.synchronizeMap(hashMap)，所以table比较慢；Map可以插入空的key或者value；HashMap和HashTable是用Hash实现的，TreeMap是用红黑树实现的。

10. ArrayList和LinkedList的相同和不同
    相同：都是继承自List；都不是同步的；有序的；允许重复；
    不同：数据结构不同；Deque是用Linked实现的；增删改查不同；Linked用memory多一些

11. Overloading and Overriding（重载和重写）

	重载时发生在编译时的，重写时发生在运行时。只有虚函数可以被重写，static，final和private的方法在java中都不能被重写
	
12. Java有几种引用类型
    
     Strong reference, Weak references, Soft reference and Phantom reference. Except strong, all other reference allows object to be garbage collected. For example, if an object hash only weak reference, than it's eligible for GC, if program needs space

13. Checked vs Unchecked Exception
     
     继承自Exception但是不继承RunTimeException的是Checked的。RunTimeException和继承自Error的是Unchecked Exception。Checked是被编译器检查的，如果块内有exception抛出，要有try，catch块。Unchecked举例：NullPointerException，ArrayIndexOutOfBound，llegalArgumentException，llegalStateException，OutOfMemmoryError。

14. Java的array是Object的实例吗

    是，它还有length属性；他的长度是固定的

15. Does List<Number> can hold Integers
     可以

16. Can we pass ArrayList\<Number\> to a method which accepts List\<Number\> in Java? 

	可以

17. Can we pass ArrayList\<Integer\> to a method which accepts List\<Number\>? 

	No
	
	 How to fix that? (use wildcards e.g. List<? extends Number> to know more about bounded and unbounded wildcards and other generics questions see this post)
	 [http://javarevisited.blogspot.sg/2012/06/10-interview-questions-on-java-generics.html](http://javarevisited.blogspot.sg/2012/06/10-interview-questions-on-java-generics.html)

18. volatile变量
 
     从内存中去读，不cache；只有变量能用volatile修饰；对变量的读写是原子的；

19. difference between CountDownLatch and CyclicBarrier in Java? 

	javadoc里面的描述是这样的:

	CountDownLatch: A synchronization aid that allows one or more threads to wait until a set of operations being performed in other threads completes.

	CyclicBarrier : A synchronization aid that allows a set of threads to all wait for each other to reach a common barrier point.
	
	CountDownLatch: 一个线程(或者多个)， 等待另外N个线程完成某个事情之后才能执行。  
	 CyclicBarrier: N个线程相互等待，任何一个线程完成之前，所有的线程都必须等待。
这样应该就清楚一点了，对于CountDownLatch来说，重点是那个“一个线程”, 是它在等待， 而另外那N的线程在把“某个事情”做完之后可以继续等待，可以终止。而对于CyclicBarrier来说，重点是那N个线程，他们之间任何一个没有完成，所有的线程都必须等待。

	CountDownLatch 是计数器, 线程完成一个就记一个, 就像 报数一样, 只不过是递减的.而CyclicBarrier更像一个水闸, 线程执行就想水流, 在水闸处都会堵住, 等到水满(线程到齐)了, 才开始泄流.
	
	CountDownLatch是一次性的，而CyclicBarrier在调用reset之后还可以继续使用

20. BlockingQueue是线程安全的吗
     
     是

21. Why wait and notify method should be called in loop? 

	To prevent doing task, if condition is not true and thread is awake due to false alarms, checking conditions in loop ensures that processing is only done when business logic allows

22. What is difference between "abc".equals(unknown string) and unknown.equals("abc")
	
	前者不会出现NullPointerException

23. What is marker or tag interface in Java? 

	an interface, which presence means an instruction for JVM or compiler e.g. Serializable, from Java 5 onwards Annotation is better suited for this job, to learn more and answer in detail see this discussion.
	
	[http://javarevisited.blogspot.sg/2012/01/what-is-marker-interfaces-in-java-and.html](http://javarevisited.blogspot.sg/2012/01/what-is-marker-interfaces-in-java-and.html)

24. Difference between Serializable and Externalizable interface in Java? 

	later provides more control over serialization process, and allow you to define custom binary format for your object, later is also suited for performance sensitive application
	
25. Can Enum types implement interface in Java?
	
	 Yes

26. Can enum extend class in Java? 

	不可以，因为java规定一个class只能继承一个class。enum默认继承自java.lang.Enum
	

27. How to prevent your class from being subclassed? 

	Make it final or make constructor private
	
28. 静态方法可以重写吗

     不会出现编译错误，但是不会有重写的效果，因为静态方法是编译时绑定的

29. Which design pattern have you used recently? 

	give any example except Singleton and MVC e.g. Decorator, Strategy or Factory pattern

30. StringBuffer和StringBuilder的区别

     StringBuffer is synchronized while StringBuilder is not which makes StringBuilder faster than StringBuffer.Use String if you require immutability, use Stringbuffer in java if you need mutable + thread-safety and use StringBuilder in Java if you require mutable + without thread-safety.


