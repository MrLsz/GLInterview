# Java 语言面试题

> 覆盖 Java 语言基础、核心类库、异常、泛型、反射、集合、并发基础、单例、锁机制、多线程等高频考点；每题给出深入、全面的标准纯文本描述答案。


### 1、Java 的基本数据类型有哪些？

Java 有八种基本数据类型：byte、short、int、long、float、double、char、boolean，它们直接存值、占用栈内存、性能好。


### 2、包装类和基本类型有什么区别？

Java 的八种基本类型（byte、short、int、long、float、double、char、boolean）都有对应的包装类 Byte、Short、Integer、Long、Float、Double、Character、Boolean。区别在于：基本类型直接存值、占用栈内存、性能好；包装类是引用类型，可存入集合、支持泛型、可为 null，并提供大量工具方法（如 Integer.parseInt）。


### 3、什么是自动装箱拆箱、什么是包装类的缓存池？

自动装箱是编译器把基本类型自动转为包装类（插入 Integer.valueOf），自动拆箱是包装类自动转为基本类型（插入 intValue），发生在赋值、方法参数、集合操作、运算等场景。装箱拆箱有两个坑：一是频繁装箱有性能开销（每次装箱可能 new 对象）；二是拆箱时对象为 null 会抛 NPE（如 Integer a = null; int b = a; 拆箱时报空指针）。

缓存池是包装类的关键优化。Integer 默认缓存 -128 到 127 之间的对象（上限可通过 -XX:AutoBoxCacheMax 调整），valueOf 命中缓存直接返回已有实例；因此在这个范围内两个 Integer 用 == 比较是 true（同一对象），超出范围是 false（不同对象）。Character 缓存 0~127，Boolean 缓存 true/false，Byte/Short/Long 也缓存 -128~127。这是"Integer 用 == 比较为什么有时 true 有时 false"的经典考点，记住"包装类比较值要用 equals 或先拆箱，只有明确要比较引用才用 =="。


### 4、String 为什么设计成不可变的？

String 内部用 final 的 char 数组（JDK 9 起改为 byte 数组配合 coder 标记编码，即紧凑字符串）存储，且不提供修改内容的方法，所以不可变。设计成不可变的原因：一是字符串常量池的复用需要不可变保证安全，多个引用共享同一个字符串实例；二是 String 常作为 HashMap 的 key，不可变保证哈希码稳定（否则哈希码变了会找不到桶）；三是不可变天然线程安全，可被多线程安全共享；四是安全性和可靠性，如类名、路径、网络参数等不被外部篡改。


### 5、String、StringBuilder、StringBuffer 有什么区别？

String 不可变导致每次拼接都产生新对象，频繁拼接性能差（产生大量中间对象）。StringBuilder 和 StringBuffer 都是可变字符序列，内部维护可扩容的 char/byte 数组，拼接不产生中间对象。区别是 StringBuilder 线程不安全但性能高，StringBuffer 的方法都加了 synchronized，线程安全但性能低。单线程（绝大多数场景）用 StringBuilder，多线程共享同一个缓冲区时才用 StringBuffer。补充：Java 编译器会把"+"拼接优化为 StringBuilder，但循环里的拼接仍应显式用 StringBuilder。


### 6、`==` 和 `equals()` 有什么区别？

`==` 是运算符：对基本类型比较值，对引用类型比较内存地址（是否同一个对象）。equals() 是 Object 的方法，默认实现也是比较地址（与 == 等价），但 String、Integer、Date 等类重写了它以比较内容。因此判断字符串内容相等必须用 equals 而非 ==，判断是否同一对象才用 ==。


### 7、为什么重写 equals 必须重写 hashCode？

重写 equals 必须重写 hashCode 的根因是哈希容器依赖一条契约：两个对象 equals 相等，则 hashCode 必须相等；但 hashCode 相等，equals 不一定相等（哈希碰撞）。违反契约的后果：在 HashMap/HashSet/Hashtable 中，两个 equals 相等的对象因 hashCode 不同会落入不同桶，导致同一个 key 存两份、get 取不到、contains 判断错误等诡异问题。所以重写 equals 时应基于相同的字段计算 hashCode（如 Objects.hash(field1, field2)）。补充：equals 本身要满足自反性、对称性、传递性、一致性，以及 equals(null) 返回 false 这五条约定。


### 8、final 关键字有什么作用？

final 是修饰符：修饰类表示类不可被继承（如 String）；修饰方法表示方法不可被重写；修饰变量表示变量不可变——基本类型值不变，引用类型引用不变（但对象内容可变）。final 变量是编译期常量时（static final 且值确定）会被内联进常量池。


### 9、finally 块有什么作用？

finally 是异常处理关键字，与 try-catch 搭配，无论是否抛异常、是否 return，finally 块都会执行（除非 JVM 退出 System.exit 或线程被杀），用于释放资源、关闭连接。注意 finally 中 return 会覆盖 try/catch 的返回值，是易错点。


### 10、finalize 方法有什么作用、为何弃用？

finalize 是 Object 的方法，垃圾回收器回收对象前可能调用，本意是对象回收前清理。但执行时机不确定、不可靠、可能复活对象，JDK 9 起已弃用，不应用于资源释放。

需要区分：final、finally、finalize 三者只是名字相似、语义完全无关——final 是修饰符，finally 是异常处理块，finalize 是待废弃的回收回调，这是面试常见的"概念辨析"送分题。


### 11、抽象类和接口有什么区别？

语法层面：抽象类用 abstract 修饰，可以有构造方法、成员变量、具体方法和抽象方法，单继承；接口 JDK 8 前只能有抽象方法和常量，JDK 8 后可定义 default 默认方法和 static 静态方法，JDK 9 后可有 private 方法，成员变量只能是 public static final 常量，支持多实现。

语义层面：抽象类是"是什么（is-a）"，表达类对类的继承关系，用于抽取子类的公共属性和行为（模板方法模式），强调代码复用；接口是"能做什么（can-do/has-a）"，是对能力的抽象，用于定义规范和契约（策略模式、回调），强调解耦和多态。


### 12、什么时候用抽象类、什么时候用接口？

设计原则：优先使用接口，因为一个类可以实现多个接口但只能继承一个类，接口让系统更灵活、耦合更低；当需要共享状态（成员变量）和通用实现时用抽象类，需要定义可扩展的行为规范时用接口。接口的 default 方法让"接口演进"成为可能——给接口加新方法不必破坏所有实现类，这是 JDK 8 引入 default 的初衷。


### 13、方法重载和方法重写有什么区别？

重载（Overload）发生在同一个类中，方法名相同但参数列表不同（参数类型、个数、顺序），与返回值、访问修饰符、异常无关，属于编译期行为，编译期就确定了调用哪个方法，也叫静态绑定。典型如多个构造方法、println 的多种重载。

重写（Override）发生在父子类之间，子类重新定义父类的方法，方法签名（方法名 + 参数列表）必须相同，返回类型可以是父类返回类型的子类（协变返回），访问权限不能比父类更严格（不能 private 重写 public），抛出的受检异常不能比父类更多，属于运行期行为。


### 14、Java 的多态是怎么实现的？

多态分编译期多态和运行期多态。编译期多态就是重载；运行期多态靠重写实现，通过父类引用指向子类对象，方法调用在运行期动态分派到实际对象类型的方法。运行期多态的实现依赖虚方法表（vtable）：每个类维护一张方法表，子类重写的方法替换父类表项，invokevirtual 指令按对象实际类型查表定位方法，这就是动态绑定。理解这一点能回答"为什么接口/父类引用调的是子类实现""为什么静态方法没有多态"。


### 15、Java 的四种访问修饰符各自的访问范围如何？

Java 有四种访问修饰符：public、protected、默认（不写，包级私有）、private，访问范围从大到小。public 对所有类可见。protected 对本类、同包、以及不同包的子类可见。默认（无修饰符）对本类和同包的其他类可见，对其他包（包括子类）不可见。private 仅本类可见。

关键易错点：protected 的可见范围是"同包 + 子类"，但不同包的子类只能通过继承关系访问父类的 protected 成员（在子类内部通过 this 或子类实例访问），不能通过父类实例随意访问（否则违背封装）；默认权限是包私有，跨包子类也不能访问。合理设置访问权限是封装性的体现，通常原则是"字段 private + getter/setter，对外 API 用 public，仅供子类或同包用 protected/默认"。


### 16、static 关键字可以修饰什么？

static 可以修饰变量、方法、代码块、内部类（不能修饰外部类和局部变量）。


### 17、静态变量、静态方法、静态代码块、静态内部类分别有什么特点？

静态变量属于类而非实例，所有实例共享一份，随类加载而初始化，用于全局共享状态和常量。静态方法属于类，可直接类名调用，不能访问实例成员（无 this）、不能用 super，常用于工具方法。静态代码块在类加载时执行一次（类初始化阶段，clinit 中），用于初始化静态变量，多个静态块按声明顺序执行。静态内部类不依赖外部类实例，可独立实例化，常用作单例的静态内部类实现（懒加载 + 线程安全）。

关键点：static 成员与实例无关，因此静态方法不能调用非静态方法或访问非静态变量（反之可以）；静态内部类不持有外部类引用，不会导致外部类内存泄漏（这是它相比非静态内部类的重要优势）。理解"静态成员属于类、随类加载初始化"是回答初始化顺序、内存位置（方法区）等题目的基础。


### 18、Java 的异常体系是怎样的？

异常体系顶层是 Throwable，下分 Error 和 Exception。Error 是 JVM 层严重错误（OutOfMemoryError、StackOverflowError），程序通常无法处理、不应捕获。Exception 是程序可处理的异常，又分受检异常和非受检异常。


### 19、受检异常和非受检异常有什么区别？

受检异常（Checked Exception）是除 RuntimeException 及其子类外的 Exception，编译器强制要求 try-catch 或 throws 声明，否则编译不通过（如 IOException、SQLException、ClassNotFoundException）。非受检异常（Unchecked Exception）包括 RuntimeException 及其子类（NullPointerException、IndexOutOfBoundsException、ClassCastException、IllegalArgumentException），不强制处理，通常是编程错误，应通过代码修正而非捕获。


### 20、finally 中 return 会发生什么？

finally 的易错点：finally 中写了 return 会覆盖 try/catch 中的返回值，最终返回 finally 的值；即使 try 中 return，finally 仍会先执行（return 前执行 finally），若 finally 又 return 则覆盖原返回值；因此应避免在 finally 中 return，只放清理逻辑。补充：try-catch-finally 的返回值语义——若 try 有 return，会先算好返回值暂存，执行 finally，再真正返回，finally 里修改局部变量不影响已暂存的返回值（但修改引用对象的内容会影响）。


### 21、Java 泛型的类型擦除是什么？

类型擦除是 Java 泛型的实现机制：泛型信息只在编译期存在，编译后被擦除，类型参数被替换为它的上界（无上界则替换为 Object），并在需要处插入强制类型转换。因此运行时 List<String> 和 List<Integer> 是同一个类，无法用 instanceof 判断泛型参数（只能判断是否为 List），也不能 new 泛型数组（泛型数组编译报错），不能重载只靠泛型参数区分的方法。类型擦除是为了兼容泛型出现前的旧代码（二进制兼容），代价是运行时丢失泛型信息。


### 22、什么是泛型通配符？

通配符表示未知类型。上界通配符 <? extends T> 表示参数是 T 或其子类，只能读不能写（因为不知道具体子类型，写入不安全，但读取能保证是 T）；下界通配符 <? super T> 表示参数是 T 或其父类，只能写不能读（写 T 及其子类是安全的，读取只能拿到 Object）。这也对应了 PECS 口诀：集合用于提供数据（读）用 extends，用于消费数据（写）用 super，典型如 Collections.copy 的签名就是 src 用 extends、dest 用 super。


### 23、什么是反射、它能做什么？

反射是 Java 在运行期动态获取类信息并操作类的能力：通过 Class 对象，可以在运行时获取类的构造方法、字段、方法、注解，并动态创建对象、调用方法、读写字段，即使成员是私有的（通过 setAccessible 绕过访问检查）。

反射的典型应用：框架和中间件（Spring 的依赖注入、MyBatis 的结果映射、ORM 框架）、动态代理（JDK 动态代理基于反射）、注解解析、插件化、热部署、反序列化等。


### 24、反射有什么优缺点？

优点是灵活通用，能在运行时处理未知类型。缺点：性能较低，反射调用比直接调用慢（涉及安全检查、方法查找、类型包装，慢一到两个数量级），可缓存 Method/Constructor 对象优化；破坏封装性，可绕过访问权限访问私有成员（setAccessible）；有安全风险，可能被恶意利用。因此应谨慎使用，框架底层可用，业务热点代码避免，JDK 8 之后用 MethodHandle 或 Lambda 可在部分场景替代反射获得更好性能。


### 25、什么是注解？

注解（Annotation）是为代码附加元数据的方式，本身不改变程序逻辑，但可被编译器或运行时框架读取并处理。JDK 内置注解如 @Override（编译期校验重写）、@Deprecated、@SuppressWarnings、@FunctionalInterface。


### 26、元注解有哪些？

元注解是修饰注解的注解，主要有四种：@Target 指定注解可作用的位置（TYPE 类、METHOD 方法、FIELD 字段等）；@Retention 指定保留级别——SOURCE 源码期（编译后丢弃，如 @Override）、CLASS 编译期（保留在字节码，默认）、RUNTIME 运行期（可通过反射读取）；@Documented 表示写入 Javadoc；@Inherited 表示可被子类继承。


### 27、注解是怎么工作的？

注解的工作方式取决于保留级别：SOURCE 级在编译期被编译器/注解处理器处理（如 Lombok、编译期代码生成）；CLASS 级可被编译期注解处理器（APT，AbstractProcessor）读取并生成代码；RUNTIME 级保留到运行期，框架用反射读取注解配置（如 Spring 的 @Autowired、@RequestMapping、@Value）。自定义注解通常配合注解处理器或反射使用，典型流程是"定义注解 → 标记代码 → 处理器读取并执行逻辑"。


### 28、Java 集合框架分为哪几类？

集合框架由两大接口派生：Collection 和 Map。Collection 又分 List、Set、Queue。


### 29、List、Set、Map 分别有什么特点和常用实现？

List 是有序、可重复的集合，支持按索引访问。ArrayList 底层是动态数组，随机访问 O(1)，尾部增删快，中间增删 O(n)（要移动元素）；LinkedList 底层是双向链表，头尾和中间增删 O(1)，随机访问 O(n)。Set 是无序、不可重复的集合。HashSet 基于 HashMap，O(1) 增删查；TreeSet 基于红黑树，有序（自然序或比较器序）、O(log n)；LinkedHashSet 基于 LinkedHashMap，维护插入顺序。Map 是键值对集合，key 不可重复。HashMap 哈希表无序；LinkedHashMap 维护插入或访问顺序；TreeMap 红黑树按 key 排序。

选型建议：频繁随机访问选 ArrayList，频繁增删选 LinkedList；去重无序选 HashSet，需要排序选 TreeSet；键值映射选 HashMap，需要有序遍历选 LinkedHashMap，需要按 key 排序选 TreeMap。线程安全场景用 ConcurrentHashMap（分段/桶锁高并发）、CopyOnWriteArrayList（读多写少）、Collections.synchronizedXxx（低并发简单包装）。


### 30、ArrayList 的底层结构和扩容机制是怎样的？

ArrayList 底层是 Object 数组 elementData，通过 size 记录元素个数（size 与数组容量是两回事）。JDK 8 起默认构造时 elementData 是空数组，第一次 add 时才真正分配容量 10（延迟初始化，节省内存）。

扩容时机是 add 时 size 达到容量上限。扩容流程：计算新容量为旧容量的 1.5 倍（newCapacity = oldCapacity + (oldCapacity >> 1)），若 1.5 倍仍不足以容纳新增元素（如 addAll 大批量），则直接使用所需最小容量作为新容量；然后 Arrays.copyOf 创建新数组并复制旧元素，把引用指向新数组。扩容涉及数组复制，开销较大（复制 O(n) 且旧数组需等待 GC）。

优化建议：若已知元素数量，构造时指定初始容量（new ArrayList<>(n)）避免频繁扩容。对比 LinkedList：链表结构不需扩容，但每次插入都要 new 节点对象、内存不连续、缓存不友好。理解扩容机制能回答"为什么 ArrayList 要预置容量""为什么说 ArrayList 中间插入慢"等问题。


### 31、HashMap 的底层数据结构和工作原理是怎样的？

HashMap 底层是数组 + 链表 + 红黑树。数组的每个位置叫桶（bucket），存放链表或红黑树的头节点。JDK 8 中当单个桶的链表长度超过阈值 8 且数组容量大于等于 64 时，链表树化为红黑树；当树节点减少到 6 以下时退化回链表，以平衡查找与内存开销。

put 过程：先对 key 的 hashCode 做扰动（高 16 位与低 16 位异或，减少低位相同导致的碰撞），再对数组长度减一取模（等价于 hash & (n-1)，因为容量是 2 的幂）得到桶下标；桶为空直接放，否则遍历链表或树，找到 key 相等（hash 相等且 equals 相等）则覆盖 value，否则追加节点；插入后若 size 超过阈值（容量 × 负载因子 0.75）则扩容为 2 倍并重新散列。get 过程：计算下标定位桶，沿链表或树查找 key 相等的节点。

JDK 8 相对 JDK 7 的改进：一是链表改红黑树，把哈希碰撞最坏 O(n) 优化为 O(log n)，防止哈希碰撞攻击（构造大量同哈希 key 拖垮性能）；二是扩容时头插法改为尾插法，修复了 JDK 7 多线程扩容时头插导致环形链表、get 死循环的经典 bug（但 HashMap 本身仍非线程安全）；三是扩容时机和哈希算法有调整。补充：容量必须是 2 的幂是为了用位运算代替取模并让元素分布更均匀。


### 32、ConcurrentHashMap 是如何实现线程安全的？

ConcurrentHashMap 是线程安全的 HashMap，支持高并发读写，性能远高于 Hashtable（全程锁整表）和 Collections.synchronizedMap（也是锁整表）。

JDK 7 采用分段锁（Segment）机制：内部维护一个 Segment 数组，每个 Segment 是一个独立的锁（继承 ReentrantLock），把整个哈希表分成多段，不同段的操作可并发进行，只有操作同一段时才竞争锁，默认 16 段、并发度 16。Segment 内部类似一个小 HashMap。

JDK 8 放弃分段锁，改为 CAS + synchronized 的细粒度锁：读操作完全无锁，通过 volatile 保证可见性；写操作用 CAS 完成空桶的初始化插入、无竞争时的节点插入；只有发生哈希碰撞（同一桶已有节点）时才对该桶的头节点加 synchronized 锁；扩容时用 CAS 配合数据迁移。这样锁粒度从"段"细化到"桶"，并发度更高、内存占用更小。

补充易错点：ConcurrentHashMap 的 key 和 value 都不允许为 null（与 HashMap 不同），因为并发场景下 null 有歧义（无法区分"不存在"和"值为 null"）。它是并发容器设计的代表，也常被问到与 Hashtable 的区别。


### 33、深拷贝和浅拷贝有什么区别、如何实现深拷贝？

浅拷贝只复制对象本身和它的基本类型字段，引用类型字段仍指向原对象的同一块内存，新旧对象共享引用，修改其中一个的引用内容会相互影响。深拷贝把引用类型字段也递归复制，新旧对象完全独立，互不影响。

实现深拷贝的常见方式：

- 手动递归复制每个引用字段
- 实现 Cloneable 接口重写 clone，对引用字段逐个 clone（注意默认的 Object.clone 是浅拷贝）
- 利用序列化，把对象序列化到字节流再反序列化，得到全新对象（要求对象及其引用字段都实现 Serializable）
- 用第三方库（Jackson/Gson/Fastjson）的 JSON 序列化做深拷贝


### 34、Java 创建线程有哪几种方式？

创建线程的方式：继承 Thread 类重写 run；实现 Runnable 接口作为参数传给 Thread；实现 Callable 接口配合 FutureTask（有返回值、可抛异常）。推荐实现接口方式，因为 Java 单继承、接口更灵活，且能更好配合线程池（线程池 ExecutorService 也只接受 Runnable/Callable）。实际开发中优先用线程池而非手动 new Thread，避免线程创建销毁开销和线程数失控。


### 35、线程有哪几种状态？

线程六种状态：

- NEW：新建未 start
- RUNNABLE：就绪或运行中，Java 把就绪和运行合并
- BLOCKED：等待获取 synchronized 锁
- WAITING：无限期等待，wait/join/LockSupport.park 未带超时
- TIMED_WAITING：限时等待，sleep/带超时的 wait/join
- TERMINATED：终止


### 36、wait() 和 sleep() 有什么区别？

wait 和 sleep 的区别：wait 是 Object 的方法，必须在同步块（持有锁）中调用，调用后释放锁，需要 notify/notifyAll 唤醒，否则一直等待；sleep 是 Thread 静态方法，不需要同步，调用后不释放锁，到时间自动醒来。二者都会让出 CPU 进入等待，但锁的释放行为是核心区别（sleep 不释放锁，wait 释放锁）。补充：notify 唤醒一个等待线程、notifyAll 唤醒所有，被唤醒的线程需重新竞争锁才能继续执行。


### 37、volatile 和 synchronized 有什么区别？

volatile 是轻量级同步机制，保证可见性和有序性（禁止指令重排），但不保证原子性。它通过内存屏障实现：写 volatile 变量后插 StoreStore/StoreLoad 屏障、读前插 LoadLoad/LoadStore 屏障，强制变量读写直接操作主内存。

synchronized 是重量级锁机制，保证原子性、可见性、有序性三者兼备。可修饰方法（实例方法锁 this、静态方法锁 Class 对象）或代码块（锁指定对象），获取锁的线程独占临界区，未获取的线程阻塞。现代 JVM 对 synchronized 做了大量优化（偏向锁、轻量级锁、锁粗化、锁消除），性能已大幅提升。

区别总结：volatile 只能修饰变量，synchronized 修饰方法或代码块；volatile 不保证原子性，synchronized 保证；volatile 不阻塞线程（自旋/直接读写），synchronized 会阻塞；volatile 性能更好但能力有限。

应用场景：volatile 适用于"一个线程写、多个线程读"的简单标志位（如停止标志 boolean flag）、双重检查锁单例的实例字段（防重排）；synchronized 适用于复合操作原子性（计数器自增）、多变量一致性更新等场景。选型：简单可见性用 volatile，需要原子性/互斥用 synchronized 或 Lock/原子类（AtomicInteger 用 CAS 实现，性能优于 synchronized 的场景）。


### 38、单例模式有哪几种实现方式？

单例模式保证一个类只有一个实例，并提供全局访问点。常见实现方式有五种：

**1. 饿汉式**：类加载时就创建实例（static final 字段），线程安全（类加载机制保证），但不管用不用都会初始化，浪费内存；若实例创建依赖参数或需要延迟则不合适。

```java
public class Singleton {
    private static final Singleton INSTANCE = new Singleton();
    private Singleton() {}
    public static Singleton getInstance() { return INSTANCE; }
}
```

**2. 懒汉式（线程不安全）**：首次调用时才 new，但多线程下可能创建多个实例，需加锁。

```java
public class Singleton {
    private static Singleton instance;
    private Singleton() {}
    public static Singleton getInstance() {
        if (instance == null) {
            instance = new Singleton();
        }
        return instance;
    }
}
```

**3. 双重检查锁（DCL，Double-Checked Locking）**：懒汉式的线程安全优化版，外层 `if (instance == null)` 判空避免每次加锁，内层 synchronized 加锁后再次判空避免重复创建。关键点是 instance 必须用 **volatile** 修饰：因为 `instance = new Singleton()` 不是原子操作（分配内存 → 初始化对象 → 把引用赋值给 instance 这三步可能被指令重排），若重排导致 instance 先指向未初始化完的对象，其他线程看到非 null 就直接返回了半初始化对象，造成错误；volatile 禁止重排，保证安全。

```java
public class Singleton {
    private static volatile Singleton instance;
    private Singleton() {}
    public static Singleton getInstance() {
        if (instance == null) {                 // 第一次检查
            synchronized (Singleton.class) {
                if (instance == null) {         // 第二次检查
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```

**4. 静态内部类**：利用 JVM 类加载的懒加载 + 线程安全特性，把实例放在静态内部类中，只有首次调用 getInstance 访问内部类时才触发加载并初始化实例，既懒加载又线程安全、无锁，是最推荐的写法之一。

```java
public class Singleton {
    private Singleton() {}
    private static class Holder {
        private static final Singleton INSTANCE = new Singleton();
    }
    public static Singleton getInstance() { return Holder.INSTANCE; }
}
```

**5. 枚举单例**：`enum Singleton { INSTANCE; }`，JVM 保证枚举实例唯一，天然防反射（反射无法 new 枚举）、防序列化（反序列化仍是同一实例），是最简洁优雅且最安全的方式，也是《Effective Java》推荐的实现。

```java
public enum Singleton {
    INSTANCE;
    public void doSomething() { /* ... */ }
}
```


### 39、反射和序列化能破坏单例吗？

反射和序列化能破坏单例：能。反射可通过 `getDeclaredConstructor().setAccessible(true)` 绕过私有构造器 new 出第二个实例；序列化会把对象序列化后反序列化生成新对象破坏单例。防御方式如下，但枚举单例两者都不怕（JVM 层面保证），所以最安全。

反射破坏与防御：

```java
// 破坏：绕过私有构造器创建新实例
Constructor<Singleton> c = Singleton.class.getDeclaredConstructor();
c.setAccessible(true);
Singleton s = c.newInstance();   // 得到第二个实例，破坏单例

// 防御：私有构造器内判断实例已存在则抛异常
private Singleton() {
    if (instance != null) {
        throw new RuntimeException("单例已存在，禁止反射创建");
    }
}
```

序列化防御（实现 readResolve 返回已有实例，而非反序列化出的新对象）：

```java
private Object readResolve() {
    return getInstance();
}
```


### 40、synchronized 的底层实现和锁升级机制是怎样的？

synchronized 是 JVM 层面的同步关键字，底层依赖**对象头里的 Mark Word** 和 **Monitor（管程）** 实现。每个 Java 对象在内存布局里都有对象头，其中 Mark Word 记录了哈希码、GC 分代年龄、锁状态标志等信息；重量级锁对应一个 Monitor 对象（基于操作系统互斥量 Mutex），持有锁的线程占用 Monitor，其他线程被挂起阻塞。

偏向锁、轻量级锁、重量级锁分别是什么：

**偏向锁**：假定锁总是被同一个线程反复获取。第一次获取时用 CAS 把线程 ID 写进 Mark Word，之后同一线程再次进入只需比对线程 ID，无需 CAS，省去竞争开销，适合"单线程反复获取同一把锁"的场景。若其他线程竞争，偏向锁撤销升级。注意 JDK 15 起偏向锁被废弃（JEP 374），因为现代应用锁竞争场景下偏向锁的撤销成本高于收益，默认不再开启。

**轻量级锁**：当有线程竞争但竞争不激烈、持锁时间短时，用轻量级锁。线程在栈帧中创建锁记录（Lock Record），通过 CAS 把 Mark Word 替换为指向锁记录的指针，成功后进入临界区；其他线程竞争时自旋（忙等）尝试获取，不立即阻塞，避免线程挂起的开销，适合"临界区短、竞争不激烈"的场景。自旋失败次数过多则升级。

**重量级锁**：竞争激烈、持锁时间长时升级为重量级锁，依赖 Monitor 和操作系统互斥量，未获取到锁的线程被挂起阻塞（进入等待队列），由操作系统调度，涉及用户态/内核态切换，开销大但能避免 CPU 空转。

因为直接上重量级锁开销大（涉及用户态/内核态切换），JDK 6 起引入了锁升级（锁膨胀）机制，从低到高依次是**无锁 → 偏向锁 → 轻量级锁 → 重量级锁**（只升不降，一般不会降级）：默认偏向锁（假定锁总被同一线程反复获取），当出现竞争、偏向锁被撤销时升级为轻量级锁（竞争不激烈、持锁时间短，线程自旋等待），当竞争激烈、自旋失败次数过多或持锁时间长时升级为重量级锁（依赖 Monitor 和操作系统互斥量，线程挂起阻塞）。

除锁升级外还有两项优化：**锁消除**（JIT 编译器通过逃逸分析判断锁不可能被竞争或对象不可能逃逸，直接去掉锁）、**锁粗化**（相邻的多个加锁解锁合并为一次更大范围的锁，减少锁开销）。

易错点：锁升级是单向的（一般只升不降）；synchronized 是可重入锁（同一线程可重复获取，Monitor 内部有重入计数器）；锁的状态、线程 ID 等都会占用 Mark Word 的比特位，因此会与 hashCode、GC 年龄互斥占用。


### 41、线程池的核心参数、工作流程和拒绝策略是怎样的？

线程池是为了复用线程、避免频繁创建销毁线程的开销、控制并发数量而引入的。核心类是 ThreadPoolExecutor，有 7 个构造参数：

1. **corePoolSize 核心线程数**：常驻线程数量，即使空闲也保留（除非设置 allowCoreThreadTimeOut）。
2. **maximumPoolSize 最大线程数**：线程池能容纳的最大线程数。
3. **keepAliveTime 存活时间**：非核心线程空闲超过该时间被回收。
4. **unit 时间单位**。
5. **workQueue 工作队列**：存放待执行任务的阻塞队列，常用 ArrayBlockingQueue（有界）、LinkedBlockingQueue（默认无界 Integer.MAX_VALUE）、SynchronousQueue（不存任务，直接交接）。
6. **threadFactory 线程工厂**：创建线程、命名、设为守护线程等。
7. **handler 拒绝策略**：线程和队列都满时的处理方式。

易错点：阿里开发手册禁止用 Executors 工厂方法创建线程池，因为 newFixedThreadPool 和 newSingleThreadExecutor 用的是无界 LinkedBlockingQueue、newCachedThreadPool 的 maximumPoolSize 是 Integer.MAX_VALUE，在任务量大时会导致 OOM（队列无限堆积或线程无限创建）；应显式 new ThreadPoolExecutor 并根据业务设定有界队列和合理参数。还要注意线程池的线程数设置需结合 CPU 密集（N+1）和 IO 密集（2N 或按阻塞系数）区分。

工作流程：提交任务后，先判断核心线程是否满——未满则创建核心线程执行，已满则把任务放入工作队列；队列满后再判断最大线程是否满——未满则创建非核心线程执行，已满则执行拒绝策略。简记「核心线程 → 队列 → 非核心线程 → 拒绝策略」。

四种拒绝策略：

- AbortPolicy：默认，抛 RejectedExecutionException
- CallerRunsPolicy：由提交任务的调用线程自己执行，能减缓提交速度
- DiscardPolicy：直接丢弃
- DiscardOldestPolicy：丢弃队列最旧的任务再尝试提交


### 42、ThreadLocal 的原理是什么、会导致内存泄漏吗？

ThreadLocal 提供线程局部变量，每个线程访问到的都是自己独立的副本，互不干扰，常用来隔离线程内的状态（如数据库连接、事务、用户上下文、SimpleDateFormat）。

原理：每个 Thread 对象内部维护一个 ThreadLocalMap（ThreadLocal 的静态内部类，本质是自定义的哈希表），key 是 ThreadLocal 实例、value 是存的值。调用 ThreadLocal.set 时，先拿到当前线程的 ThreadLocalMap，再以当前 ThreadLocal 为 key 存值；get 同理。因为每个线程都有自己的 ThreadLocalMap，所以变量天然线程隔离，无需同步。

内存泄漏问题：ThreadLocalMap 的 Entry 继承了 WeakReference，key（ThreadLocal 实例）是弱引用，value 是强引用。当 ThreadLocal 实例外部没有强引用时，key 会被 GC 回收变成 null，但 value 仍被 Entry 强引用，Entry 又引用 value，而 value 只能通过 key 访问——key 没了导致 value 无法被访问也无法被回收，造成内存泄漏。尤其在**线程池**场景下，核心线程长期存活不销毁，ThreadLocalMap 一直存在，泄漏的 value 越积越多，可能 OOM。

规避：使用完 ThreadLocal 后务必调用 remove() 清除（配合 try-finally 保证）；ThreadLocal 本身也在 get/set/remove 时对 key 为 null 的过期 Entry 做清理（expungeStaleEntry），但不够及时，不能依赖。延伸：InheritableThreadLocal 能让子线程继承父线程的值，但线程池下复用线程会导致值错乱，可用阿里开源的 TransmittableThreadLocal（TTL）解决跨线程池传递上下文的问题。


### 43、什么是 CAS、它有什么问题？

CAS（Compare And Swap，比较并交换）是乐观锁的核心机制，由 CPU 提供原子指令支持。它包含三个操作数：内存地址 V、期望值 A、新值 B，只有当 V 当前值等于 A 时才把 V 更新为 B，否则什么都不做，整个过程是原子的。Java 通过 sun.misc.Unsafe 的 compareAndSwapInt 等本地方法调用 CPU 的 CAS 指令实现。

CAS 的三个问题：一是 **ABA 问题**——值从 A 变成 B 又变回 A，CAS 检查时发现仍是 A，误以为没变，实际已经变了，可用版本号/时间戳解决（AtomicStampedReference、AtomicMarkableReference）；二是 **自旋开销**——CAS 失败会循环重试，竞争激烈时 CPU 空转开销大；三是 **只能保证单个变量的原子性**——多变量原子操作需用 synchronized 或锁。


### 44、原子类是怎么实现的？

原子类（java.util.concurrent.atomic）用 CAS + volatile 实现无锁线程安全：AtomicInteger、AtomicLong、AtomicBoolean 保证单变量原子操作；AtomicReference 支持引用类型；LongAdder 用分段累加降低竞争，高并发累加性能优于 AtomicLong；AtomicIntegerFieldUpdater 等更新已有对象的字段。它们是「用无锁替代 synchronized」的典型。


### 45、死锁产生的必要条件是什么、如何避免和排查？

死锁是指两个或多个线程互相持有对方需要的锁资源并等待对方释放，导致都无法继续执行、永久阻塞的状态。

死锁产生的四个必要条件（缺一不可）：**互斥**（资源一次只能被一个线程持有）、**占有且等待**（线程持有资源的同时等待其他资源）、**不可剥夺**（已持有的资源不能被强制抢占，只能主动释放）、**循环等待**（线程之间形成首尾相接的资源等待环）。

避免死锁的思路是破坏上述任一条件：互斥一般无法破坏（资源本性）；可破坏「占有且等待」——一次性申请所有资源；可破坏「不可剥夺」——占用资源的线程申请不到新资源时主动释放已占资源；最常用的是破坏「循环等待」——**按固定顺序加锁**（所有线程按相同顺序获取锁，避免交叉），或用 **tryLock 带超时**（ReentrantLock.tryLock(timeout) 获取不到就放弃并释放已持有的锁），避免无限等待。

排查死锁：一是用 jstack 工具 dump 线程栈，会直接输出 "Found one Java-level deadlock" 及涉及的线程和锁信息；二是用 ThreadMXBean 的 findDeadlockedThreads() 程序化检测；三是监控线程阻塞和锁等待情况。预防方面，尽量减少锁的粒度和嵌套、避免在持锁时调用外部方法（可能间接获取其他锁）、使用无锁或乐观锁替代。

> 附：高频速记 —— == 比地址、equals 比内容、重写 equals 必重写 hashCode；String 不可变、拼接用 StringBuilder；final 修饰不可变、finally 必执行、finalize 已弃用；重载看参数、重写看签名；HashMap 数组+链表+红黑树、JDK8 尾插防死循环；ArrayList 扩容 1.5 倍；volatile 管可见、synchronized 管原子；泛型编译期擦除、通配符 extends 读 super 写；单例 DCL 要 volatile、枚举单例最安全；锁升级 偏向→轻量→重量（只升不降）；线程池流程 核心线程→队列→非核心→拒绝；ThreadLocal 用完必 remove 防泄漏；CAS 有 ABA 问题、原子类无锁；死锁四条件、按顺序加锁破循环等待。
