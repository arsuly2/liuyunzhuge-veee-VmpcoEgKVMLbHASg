ThreadLocal是什么？thread是线程，local是本地的意思字面意思是线程本地。其实更通俗的理解是给每个线程设置一个缓存。这个缓存用来存储当前线程在未来的业务逻辑中需要执行到的变量。我们先来看怎么用：

首先创建全局变量ThreadLocal，各自启动一个线程任务：线程任务将变量设置到缓存中。线程任务需要用到缓存中的变量时，直接从缓存中取即可。

```
 1 import java.util.concurrent.TimeUnit;
 2 
 3 /**
 4  * @discription
 5  */
 6 public class ThreadLocalLearn {
 7     static ThreadLocal threadLocal = new ThreadLocal<>();
 8 
 9     public static void main(String[] args) {
10         Runnable r = new Runnable() {
11             @Override
12             public void run() {
13                 threadLocal.set(Thread.currentThread().getName());
14                 sayMyName();
15                 threadLocal.remove();
16             }
17 
18             public void sayMyName() {
19                 for (int i = 0; i < 3; i++) {
20                     String name = threadLocal.get();
21                     System.out.println(Thread.currentThread().getName() + " say: im a thread, name:" + name);
22                     try {
23                         TimeUnit.SECONDS.sleep(3);
24                     } catch (Exception e) {
25                         //...
26                     }
27                 }
28             }
29         };
30         Thread t1 = new Thread(r);
31         t1.start();
32         Thread t2 = new Thread(r);
33         t2.start();
34     }
35 }
```

它的使用非常简单，（1）先set()存储值；（2）使用时get()取出值；（3）用完了使用remove()清理掉；

输出如下：

```
Connected to the target VM, address: '127.0.0.1:56863', transport: 'socket'
Thread-0 say: im a thread, name:Thread-0
Thread-1 say: im a thread, name:Thread-1
Thread-0 say: im a thread, name:Thread-0
Thread-1 say: im a thread, name:Thread-1
Thread-1 say: im a thread, name:Thread-1
Thread-0 say: im a thread, name:Thread-0
Disconnected from the target VM, address: '127.0.0.1:56863', transport: 'socket'
```

很多人第一次见到ThreadLocal，第一直觉它的实现是用Map 。(防盗连接：本文首发自https://github.com/jilodream/ )但是深入研究之后，你会发现threadLocal的实现要比这样一个map 精妙的多，也好用的多。我们通过查看java源码，可以依次探索ThreadLocal是如何实现缓存的：

类整体的关系大概是这样的：

![tlleitu](https://img2024.cnblogs.com/blog/704073/202509/704073-20250929152250727-1012654768.png)

 查看源码，我们可以发现如下特性：

1、ThreadLocal本身并不是缓存，它只是起到一个缓存的key 的作用。我们每次创建一个ThreadLocal 并不是真正的创建了一个缓存，其实只是创建了一个缓存的标识。源码如下：this 就是ThreadLocal实例

```
1     public void set(T value) {
2         Thread t = Thread.currentThread();
3         ThreadLocalMap map = getMap(t);
4         if (map != null) {
5             map.set(this, value);
6         } else {
7             createMap(t, value);
8         }
9     }
```

2、真正的缓存保存在Thread中，缓存被定义为:ThreadLocal.ThreadLocalMap threadLocals;从名字可以发现，这个缓存的类型是在ThreadLocal 中定义的一个静态内部类。这个类就是用来真正存放缓存的地方。这就像是thread小书包一样，每个线程有一个自己的独立的存储空间。设计疑问：它（ThreadLocalMap）为什么没有定义在Thread类中，毕竟它是Thread的缓存。

源码如下：*Thread.java*

```
1     /* ThreadLocal values pertaining to this thread. This map is maintained
2      * by the ThreadLocal class. */
3     ThreadLocal.ThreadLocalMap threadLocals = null;
```

 3、查看ThreadLocalMap的源码，我们发现它并没有实现Map接口，就像其他map一样，ThreadLocalMap实现了常用的Map中的set，get，getEntry，setThreshold，，remove 等方法。

并且它内部使用了线性探测法来解决哈希冲突。设计疑问：它（ThreadLocalMap）为什么没有实现Map接口？源码如下：ThreadLocal.Java

```
 1 static class ThreadLocalMap {
 2 
 3         //...
 4 
 5         private static final int INITIAL_CAPACITY = 16;
 6 
 7 
 8         private Entry[] table;
 9 
10 
11         private int size = 0;
12 
13 
14         private int threshold; // Default to 0
15 
16 
17         private void setThreshold(int len) {
18             threshold = len * 2 / 3;
19         }
20 
21 
22         private Entry getEntry(ThreadLocal key) {
23                  ...
24         }
25 
26 
27 
28         private void set(ThreadLocal key, Object value) {
29         ...
30         }
31 
32 
33         private void remove(ThreadLocal key) {
34         ...
35         }
36 
37 
38         private void rehash() {
39             ...
40         }
41 
42         private void resize() {
43            ...
44         }
45       ....
46     }
```

4、继续看源码，我们发现ThreadLocalMap类像其他Map实现一样，在内部定义了Entry。并且这个Entry居然继承了弱引用，弱引用被定义在Entry的key上，而且key的类型是ThreadLocal。

至于什么是弱引用，我以前的文章中介绍过，请看（浅谈Java中的引用   https://github.com/jilodream/p/6181762.html），一定要对弱引用了解，否则ThreadLocal的核心实现以及它会存在的问题，就无法更深理解了。

这里又会有疑问，为什么要使用弱引用，使用强引用不好吗？弱引用万一被回收导致空引用等问题怎么办？

源码如下：ThreadLocal.Java

```
1         static class Entry extends WeakReference> {
2             /** The value associated with this ThreadLocal. */
3             Object value;
4 
5             Entry(ThreadLocal k, Object v) {
6                 super(k);
7                 value = v;
8             }
9         }
```

我们依次回答这几个问题：**（1）设计疑问：它（ThreadLocalMap）为什么没有定义在Thread类中，毕竟它是Thread的缓存**这恰恰是Thread符合开闭原则的优秀设计。如果是将ThreadLocalMap添加到Thread中，那么Thread类就太重了，以后只要和线程相关的业务都要将代码添加到Thread中，那Thread就无限膨胀了，变成超级类了，试想什么业务和线程能脱离关系呢？况且他们只是类依赖关系而不是组合关系（对类关系不了解的同学可以看我的这篇文章：统一建模语言UML---类图  https://github.com/jilodream/p/16693511.html）。

Map怎么实现，缓存怎么维护，这些都是Thread不需要考虑的，我们就是需要用到你的特性。

**（2）设计疑问：它（ThreadLocalMap）为什么没有实现Map接口？**实现接口是为了统一化提供接口，让外界可以只依赖接口，而不是接口的实现。但是ThreadLocalMap并不是给外界使用的，并不需要暴露出来。他就是为了给ThreadLocal业务使用的。只要完成最核心的Map能力，用空间换时间，将理论时间复杂度推向O(1)即可。因此完全没有必要实现Map接口。实现了Map接口反而要将内部方法暴露为public，这也不符合最少知道原则。一句话就是没必要，还添乱。

**（3）为什么要使用弱引用，使用强引用不好吗？弱引用万一被回收导致空引用等问题怎么办？**我们需要先了解弱引用的特性：当一个变量只有弱引用关联时，那么在下次GC回收时，不论我们内存是否足够，都将回收掉该内存。第一眼感觉这很危险，毕竟我们非常担心就是一个变量用着用着突然不能用了，出现空引用了，漫天的空引用这太不可控了。其实这完全多虑了，注意看：我们是如何使用缓存的，是通过threadlocal.get(),也就是说我们想要使用缓存就一定要使用threadlocal的实例，也就是强引用，有了强引用，使用时就一定不会被回收。因此完全不用担心使用缓存中，弱引用key突然变为null的情况了。那什么时候弱引用key会被回收呢？这就是当外界的强引用被手动设置为null时，(防盗连接：本文首发自https://github.com/jilodream/ )或者是作为局部变量跳出了方法栈，超出生命周期被回收掉了。试想一下，真要是发生这两种情况，那么其实这个缓存也就根本无法再用到了同时，key被尽快回收，反而对内存更有利。那么弱引用这么好用，为什么value不设置为弱引用呢？其实细想一下就会发现value一定不能设置为弱引用，为什么呢？key设置为弱引用，是因为想要使用这个缓存，key就一定要有强引用关联。而value则不一定有外界强引用关联，它在外界的强引用可能早就消失了。比如下面这个例子：

```
 1 import java.util.concurrent.TimeUnit;
 2 
 3 /**
 4  * @discription
 5  */
 6 public class ThreadLocalLearn {
 7     static ThreadLocal userContext = new ThreadLocal<>();
 8 
 9     public static void main(String[] args) {
10         Runnable r = new Runnable() {
11             @Override
12             public void run() {
13                 setUserInfo();
14                 handle();
15                 userContext.remove();
16             }
17 
18             public void handle() {
19                 UserInfo user = userContext.get();
20                 //注意倘若map中的value被定义为弱引用，则此处的user可能为null
21                 System.out.println(" i am:" + user.toString());
22                 //do sth
23                 try {
24                     TimeUnit.SECONDS.sleep(3);
25                 } catch (Exception e) {
26                     //...
27                 }
28             }
29         };
30         Thread t1 = new Thread(r);
31         t1.start();
32         Thread t2 = new Thread(r);
33         t2.start();
34     }
35 
36     private static void setUserInfo() {
37         UserInfo user = new UserInfo();// 假装是从db中获取的
38         userContext.set(user);
39         //跳出该方法后，userInfo的在外部的直接强引用就被回收了
40     }
41 }
42 
43 class UserInfo {
44     private String name;
45     private int age;
46     
47     //....
48 }
```

我们在A方法中设置了缓存 currentUserId，跳出A方法，currentUserId在外界的引用被断开，倘若此时value也被定义为弱引用，value就随时可能被回收。而我们又可以通过

**(key)Threadlocal  -->  threadLocals(ThreadLocalMap)  -->  entry  -->  value**

这样的调用关系来拿到缓存value。这样缓存的使用就不可控了。那么value一定不能设置为弱引用或及时回收么？并不是，其实我们只要在key回收时，顺手对value也做一个回收，但是这是GC完成的，再key消失时，联动对所有线程中关联的Map都进行一遍清理。（实现过于复杂）亦或者清理key（threadlocal）的强引用时，将value的强引用也一并被清理。可行，也是ThreadLocal推荐的方式，需要手动调用ThreadLocal.remove 方法。在调用remove方法后，ThreadLocalMap会对所有垃圾数据进行清理，还会压缩哈希表。为了解决ThreadLocalMap的value 延迟清理的情况，ThreadLocalMap在set get remove等方法中，都会对ThreadLocalMap存在的这种 垃圾数据进行一定程度的清理（注意这里要分各种情况，具体只能详细分析源码了，一篇博文很难说清）。

**(4)这样又会有一个新的问题，如果key 被回收了，但是value没有被回收，因此value就常驻内存了，那么value不就会导致内存泄露吗？**很不幸，这样的确是会导致内存的泄露。（这里简单提一下，java中的内存泄露是指，可以通过强引用关联到他，gc无法回收掉它。与此同时，业务按照正常逻辑又无法使用到它。也就是又用不到，又回收不掉，就称之为内存泄露）但是这种内存泄露出现的概率非常低。

它需要同时满足以下三个条件才可以：1、需要线程的生命周期永远不会结束。如果线程生命周期结束了，那么ThreadLocalMap就会被回收，里边出现的无其他关联的key value 也都会被回收。这种一般是守护线程或者线程池（线程复用出现）

2、ThreadLocal在设置为null时，没有手动调动remove方法

3、线程中的ThreadLocalMap在后续使用中，没有再调用任何get set remove方法，也就是线程没再使用ThreadLocal

概率低，是不是代表不太需要关注，当然不是。因为内存泄露不仅仅是减少了可用内存，还增加了GC负担，系统性能就会收到影响，这就说的远了。

其实ThreadLocal最大的问题，并不是泄露的问题，而是被滥用的问题，不规范使用的问题。很多人把ThreadLocal当成是线程的私有仓库，所有变量参数都往里边塞，导致写代码和维护时，非常不方便，出现问题也给维护人员造成很大的困扰。

接下来我们简单说下ThreadLocal的使用（*后边我会再写一篇，如何使用使用ThreadLocal，毕竟我们学习技术目的是能够驾驭它，而不仅仅是知其所以然*）：我们一般是将上下文信息，或者当前需要频繁使用的，与实际业务直接关系不大的系统数据方便携带。放置到thread的小书包中。（1）上下文信息如我们在controller层，将用户的上下文信息传入，如traceId（方便链路追踪），如用户token，后续可能调用其他鉴权接口等（2）解耦数据库连接等连接池信息，比如Springboot运行事务时，我们每次getconnection()，就只使用ThreadLocal中贮存好的这个连接，整个方法使用的是同一个数据库连接。以上场景不使用ThreadLocal可以吗？也可以，他并不是一定要使用。但是你这样就要把很多的参数传来传去，暴露很多的问题。甚至在很多第三方实现的框架中，他不支持你传这些参数，他就是要用通过ThreadLocal来回传值。

（3）为线程安全提供了方案，减少了锁竞争：如果说锁是从资源竞争的角度，解决了数据安全的问题。ThreadLocal则是在每个线程中，只保存（只隔离）出与自己当前业务相关的数据。注意他只是保证了数据的独立性，并不是独立创建了一份副本，(防盗连接：本文首发自https://github.com/jilodream/ )所以如果使用全局数据放置到value中时，一样可能会有数据安全问题。（当然这也是不推荐的用法）比如有一份UserCache的全局缓存，多线程使用时，我可以在全局中对UserCache进行加锁处理，也可以每个线程独立引用自己的UserInfo，线程之间互不干扰。结构就像这个样子：

全局加锁：

![tljingzheng](https://img2024.cnblogs.com/blog/704073/202509/704073-20250929154021459-1004970887.png)

线程各自引用：

![tlyinyong](https://img2024.cnblogs.com/blog/704073/202509/704073-20250929153958989-1539276905.png)

不知讲到这里大家还有没有最初的直觉了，为啥不设计一个全局的  Map。这样不是更简单，也更好定位问题：

细想一下，就会发现这样并不好：方案1，全局只有一个Map，value是当前线程的所有缓存数据。那么Object就是一个非常复杂的数据，每次对Object进行读取都要解析的特别复杂。方案2，全局定义的很多个Map，每个map是一个业务的缓存，比如User，就有userMap，token就有tokenMap。先不论Map本来就会有竞争的问题，对于管理大量的Map就是一件头痛的事情。

当然还是要根据具体业务来看，不能一概而论，并不能说任何时候使用ThreadLocal更好，使用全局Map更弱

本博客参考[surfshark](https://surfsharkcn.com)。转载请注明出处！
