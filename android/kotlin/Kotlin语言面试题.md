# Kotlin 语言面试题

> 覆盖 Kotlin 语言基础、空安全、函数与 Lambda、类与对象、委托、扩展、作用域函数、集合、泛型型变、互操作、协程核心等高频考点；每题给出标准纯文本描述答案，并补充底层编译产物、易错点与延伸。


### 1、Kotlin 相比 Java 有哪些主要区别？

Kotlin 与 Java 都编译为 JVM 字节码，可 100% 互操作、可混编，最终产物都是 .class 文件，运行在同一个 JVM 上。主要区别体现在：

**空安全**：Kotlin 用类型系统区分可空类型 T? 与非空类型 T，在编译期强制判空，从源头消除一大类 NPE；Java 只能靠 @Nullable/@NonNull 注解 + 静态检查工具（如 IDE、SpotBugs、NullAway）事后补救，运行期才暴露问题。

**简洁性**：Kotlin 有 data class（自动生成 equals/hashCode/copy）、默认参数（替代重载爆炸）、扩展函数、字符串模板、when 表达式、解构声明等，大幅减少样板代码。官方估算同一逻辑 Kotlin 代码量约为 Java 的 40%。

**函数式特性**：Kotlin 把函数当作一等公民，天然支持高阶函数、Lambda、带接收者的 Lambda（DSL 基石）；Java 8 之后才有 Lambda 和 Stream，但函数式接口受限于 SAM，且 Java 的 Lambda 只是语法糖，没有内联能力。

**协程**：Kotlin 用 suspend + 协程实现轻量异步，用同步写法写异步代码；Java 长期依赖线程 + 回调，JDK 21 才引入虚拟线程（Virtual Thread），而 Kotlin 协程早在 2018 年就稳定。

此外 Kotlin 没有受检异常（checked exception），调用更简洁。需要留意：Kotlin 特性虽多，但会引入额外的编译产物（如伴生对象生成内部类、顶层函数生成 XxxKt 类），包体积和编译时间略增，这是取舍。


### 2、val 和 var 有什么区别？

val 声明只读变量（引用不可重赋值，对应 Java 的 final），var 声明可变变量（引用可重新赋值）。Kotlin 有强大的类型推断，声明时通常可以不写类型。

**编译产物**：val 在字节码层面被编译为 final 字段，若同时是顶层/伴生对象常量，配合 const 可编译为真正的编译期常量（内联到调用点，无字段访问）。var 则编译为带 getter/setter 的普通字段（private backing field + 访问器）。

val 声明的对象**不一定不可变**。val 只保证引用本身不可变，若它指向的是可变对象（如 mutableListOf），对象内部内容仍可修改。例如 `val list = mutableListOf(1)` 仍然可以 `list.add(2)`，只是不能把 list 重新赋值为另一个集合。真正不可变要用只读集合（listOf、mapOf）或不可变类（data class 全 val + 不可变字段）。同理，`val map = HashMap<String, Int>()` 仍可 put。

推荐优先用 val：符合函数式编程的不可变思想，减少副作用，便于推理和并发安全（不可变对象天然线程安全）；val 对应 Java 的 final，有利于 JVM 优化（常量传播、逃逸分析）；还能让代码意图更清晰——看到 val 就知道这个变量不会被重新赋值。业界惯例是「能 val 就 val，除非确需重赋值才用 var」，这也是 Kotlin 官方编码规范的建议。


### 3、Kotlin 的空安全是怎么实现的？

Kotlin 通过**类型系统**区分可空类型 T? 和非空类型 T。非空类型在编译期保证永远不为 null，可空类型必须显式处理空值才能使用。这种可空性检查由编译器在编译期强制执行，运行时没有额外的可空包装，**零运行时开销**——这是它区别于 Java Optional 的关键：Optional 是一个包装对象，有装箱和对象分配成本，而 Kotlin 的 T? 只是编译器层面的标注，字节码里 Int? 仍是原始 int（可空时才是 Integer）。

空安全的三个核心操作符：

**?.（安全调用）**：对象可能为空时安全访问，若为 null 则整个表达式返回 null，否则执行访问，如 `s?.length`。编译产物是在调用前插入 `if (s == null) null else s.length()` 的判断。链式调用 `a?.b?.c` 只要中间任一环节为 null，整体返回 null，不会继续向下调用。

**?:（Elvis 操作符）**：左侧结果为 null 时提供默认值，常和安全调用连写，如 `val n = s?.length ?: 0`。注意它会在左侧为 null 时才对右侧求值（惰性），右侧可以是表达式甚至 throw/return。

**!!（非空断言）**：强制把可空类型转为非空类型，若对象为 null 会抛 NPE，等于主动放弃空安全，应尽量避免。字节码里 !! 编译为 `Intrinsics.checkNotNull` 调用，失败抛 KotlinNullPointerException（带源码位置信息）。


### 4、Kotlin 的 == 和 === 有什么区别？

Kotlin 的 **== 是结构相等**，等价于调用 equals()，比较对象内容是否相等；**=== 才是引用相等**，比较两个引用是否指向同一个对象。

**编译产物**：Kotlin 的 `a == b` 会编译为 `Intrinsics.areEqual(a, b)`，该函数内部先判断引用是否相同，再判空后调用 equals；因此 `a == null` 也能安全判断，不会 NPE。而 `a === b` 编译为 JVM 的 if_acmpeq 引用比较，对基本类型来说 === 与 == 等价（基本类型无对象身份）。

例如两个内容相同的字符串 a 和 b，`a == b` 为 true，`a === b` 是否为 true 取决于是否是同一对象（字符串常量池复用则可能为 true）。

和 Java 的 == 不一样：Kotlin 的 == 对应 Java 的 equals()（结构相等），而 Kotlin 的 === 才对应 Java 的 ==（引用相等，比较地址）。这是 Kotlin 的刻意设计，避免 Java 中 `str == "x"` 比较地址的经典坑。Java 中 `new String("a") == new String("a")` 为 false，而 Kotlin 中 `==` 结果为 true——这是两语言最需要警惕的语义差异。


### 5、Kotlin 的类型推断是什么？

Kotlin 编译器能根据初始化值、返回值、Lambda 参数等自动推断变量类型，`val s = "abc"` 自动推断为 String，无需显式声明。推断同样作用于泛型（`val list = listOf(1)` 推断为 List\<Int\>）和 Lambda 参数类型。类型推断让代码更简洁，但复杂场景（如推断歧义）仍需显式标注类型帮助编译器。


### 6、字符串模板是什么、底层怎么实现？

**字符串模板**用 $ 符号在字符串中嵌入变量或表达式：`$name` 直接插入变量，`${expression}` 插入任意表达式结果（表达式需用花括号包裹）。它替代了 Java 中繁琐的字符串拼接，且比 String.format 更直观、类型安全。

**底层实现**：字符串模板在 JVM 上编译为 StringBuilder.append 的拼接（或多个 + 的 String 连接，编译器会优化），与 Java 手写 StringBuilder 等价，没有额外性能损耗。单个 `$name` 会编译为常量 + append；`${expr}` 会先对 expr 求值再 append。若模板中要输出字面 `$`，需转义为 `\$`。

易错点：普通字符串中的 `$` 后面若跟不是合法标识符的字符，编译器可能报错或按字面处理，表达式一定用 `${}` 包裹最稳妥。


### 7、Kotlin 的 when 和 Java 的 switch 有什么区别？

when 和 switch 的核心区别是：**when 是表达式（有返回值）**，switch 是语句；when 的每个分支返回值可作为表达式结果，常替代复杂的三元表达式或 if-else 链。

when 功能更强，体现在：

1. **支持任意类型判断**：不只是 int/枚举/字符串，任何类型都能作为 when 的主题。
2. **支持任意布尔条件分支**：分支可以是复杂条件表达式。
3. **支持 is 类型判断和 in 区间判断**：`is String ->`、`in 1..10 ->`，配合智能类型转换使用。
4. **支持无参数形式**：when 后面不跟变量，各分支直接写布尔条件，等价于 if-else 链。
5. **不需要 break**：匹配一个分支后自动结束，不会贯穿（fall-through），从根本上消除 switch 忘记 break 的 bug。
6. **可用逗号合并多个匹配值**：`1, 2, 3 ->`。
7. **else 作为默认分支**：若 when 作为表达式使用，else 通常是必须的（除非编译器能证明分支已穷尽，如对 sealed class 或枚举的穷尽匹配）。

**编译产物**：when 会根据分支情况编译为 if-else 链，或当主题是 int/String/枚举且分支是常量时优化为 JVM 的 tableswitch/lookupswitch 字节码，性能与 switch 相当。对 sealed class 的 when 编译为类型判断 + checkcast。注意：when 作为语句（不接收返回值）时，分支不必穷尽，else 可省略。


### 8、函数默认参数和命名参数是什么？

**默认参数**允许函数参数在声明时给出默认值，调用时可省略，如 `fun f(a: Int, b: Int = 0)`。**命名参数**允许调用时按参数名传参，如 `f(a = 1, b = 2)`，可以跳过中间参数、改变传参顺序，提高可读性。

**底层实现**：Kotlin 编译时会在字节码中生成一个 `$default` 合成方法，用整数的位掩码标记哪些参数使用了默认值，运行时按掩码填充默认值；真正的函数体方法只接收完整参数列表。

默认参数与继承有冲突——子类重写带默认参数的方法时不能重新指定默认值（默认值在编译时绑定到静态调用点）。此外命名参数配合默认参数可跳过中间参数，但可变参数（vararg）之后的位置参数无法用命名参数跳过。

它们解决了 Java 中构造函数和方法的「参数列表爆炸」问题：Java 为支持可选参数常要写多个重载方法或使用建造者模式（Builder），Kotlin 用默认参数一行就能搞定，配合命名参数只传想改的参数。Java 调用 Kotlin 的默认参数方法时，默认参数不生效，需要通过 `@JvmOverloads` 注解生成多个重载方法，让 Java 能省略默认参数。


### 9、什么是高阶函数？

高阶函数是把函数作为参数或返回值的函数。Kotlin 中函数是一等公民，可以像变量一样传递。高阶函数配合 Lambda 让 Kotlin 支持函数式编程风格，如集合的 map/filter/fold 等操作，代码更简洁、意图更清晰。

函数类型描述了函数的签名，如 `(Int, Int) -> Int` 表示接收两个 Int、返回 Int 的函数类型。

**函数类型与 JVM 的映射**：函数类型在 JVM 上编译为 Function 系列接口（kotlin.jvm.functions.Function0~Function22，按参数个数区分），如 `(Int, Int) -> Int` 对应 `Function2<Int, Int, Int>`，通过 invoke 方法调用。这是 Kotlin 函数类型能作为对象传递的底层原因。

Lambda 表达式是函数类型的字面量写法，本质上是一个实现了对应函数类型接口的对象，如 `{ a, b -> a + b }` 就是 `(Int, Int) -> Int` 的一个实例。Lambda 的最后一个表达式是返回值，用 it 表示单参数隐式名字。Lambda 可以捕获外部变量（闭包）——编译时捕获的变量会被装箱到 Ref 对象中共享，使闭包能读到外部变量的最新值。

Lambda 作为参数会创建额外对象（Function 实例 + 可能的捕获对象），高频调用场景可用 inline 优化；也可把 Java 的 SAM 接口（Runnable 等）直接传 Lambda，编译器会自动做 SAM 转换。


### 10、什么是带接收者的 Lambda？

带接收者的 Lambda（Lambda with Receiver）是 Kotlin 的一种特殊函数类型，形如 `A.() -> B`，表示这个 Lambda 在某个接收者对象 A 的作用域内执行，Lambda 体内可以直接访问接收者 A 的成员，this 指向接收者。

**编译产物**：`A.() -> B` 编译为 `Function1<A, B>`，接收者 A 作为第一个参数传入 invoke 方法。

典型应用是类型安全构建器（DSL）：Kotlin 协程的 suspend block、HTML DSL、Gradle 配置脚本，apply、with、run、also 等作用域函数以及函数式 DSL 的 block 参数都是带接收者的 Lambda。它让配置式代码读起来像声明式语言，是 Kotlin DSL 的基石。要注意接收者 this 在嵌套 Lambda 中会遮蔽外层 this，需用 `this@label` 显式指定。

和普通 Lambda 的区别：普通 Lambda 通过参数传入对象、用 it 访问；带接收者的 Lambda 直接把对象作为 this 接收者，在体内无需前缀即可访问其成员，写起来更像「在对象内部写代码」。用 apply 举例：`user.apply { name = "Tom" }` 中 `{ name = "Tom" }` 就是带接收者的 Lambda，name 直接赋值给 user 的成员。在字节码层面，带接收者的 Lambda `A.() -> B` 和普通 Lambda `(A) -> B` 是同一接口 `Function1<A, B>`，区别只在编译器的语法糖层面——编译器把接收者作为 this 处理。


### 11、inline、noinline、crossinline、reified 分别是什么？

**inline 内联函数**会把函数体和其中的 Lambda 直接复制到调用处，而不是创建函数对象并调用。代价是代码体积膨胀（每处调用都复制一份），所以只对频繁调用的小函数或带 Lambda 的函数使用 inline。

**noinline** 修饰 inline 函数的某个函数类型参数，表示该参数不被内联，仍作为普通函数对象存在。用于该 Lambda 需要被保存下来、作为返回值、存到字段或传给其他非内联函数等无法内联的场景。

**crossinline** 修饰 inline 函数的函数类型参数，禁止该 Lambda 中出现非局部返回（return）。因为内联后 Lambda 体被复制到调用处，其 return 会直接结束外层函数；用 crossinline 可防止在 Lambda 内直接 return 外层函数，但允许嵌套的局部 return（如 return@label）。

**reified** 是 inline 函数的专属能力：因为函数体内联后类型信息被保留，泛型类型参数 T 可以被具体化，从而在函数体内使用 `T::class`、`is T` 等运行时类型操作（普通泛型由于类型擦除无法做到）。典型如 Gson 反序列化扩展函数 `inline fun <reified T> fromJson(json: String): T`。

为什么要用 inline：inline 的好处是消除 Lambda 作为参数时创建的匿名类对象、减少函数调用开销；配合 reified 实现具体化类型（普通函数无法做到，因为函数体内联后类型信息被保留，泛型类型参数 T 才可在运行时使用 `T::class`、`is T` 等类型操作）。代价是代码体积膨胀。

**易错点**：inline 函数不能访问其所在类的 private 成员（内联后跨类边界会破坏封装）；public 的 inline 函数不能调用非 public 的 API；递归函数不能 inline；reified 只能与 inline 搭配，非 inline 函数用 reified 会编译报错。


### 12、什么是 data class？

**data class 数据类**自动生成 equals、hashCode、toString、copy、componentN（解构）等方法，用于承载数据的类，如 `data class User(val name: String, val age: Int)`，声明时至少要有一个主构造参数。生成的 equals/hashCode 基于主构造参数，copy 支持选择性复制修改某个字段，componentN 支持解构声明 `val (name, age) = user`。注意：数据类要求主构造参数至少一个 val/var；自动生成方法只覆盖主构造参数，body 中声明的属性不参与 equals/hashCode。


### 13、什么是 object（单例）？

**object 声明单例**，一个 object 声明就同时是类定义和唯一实例，天然线程安全的懒加载单例（编译为 INSTANCE 静态字段，类加载时初始化），如 `object Singleton`。object 也可用于伴生对象、匿名对象表达式（匿名对象可实现多个接口、跨线程返回时类型受限）。


### 14、什么是 companion object？

**companion object 伴生对象**是类内部的 object，相当于 Java 的静态成员容器，但本质上是一个真实的对象实例，可有自己的名字、实现接口、继承类。Kotlin 没有 static 关键字，类级别的成员（工厂方法、常量、静态变量）都放在 companion object 里。编译后：类内生成一个静态字段指向 Companion 实例，成员既可通过 Companion 访问，用 @JvmStatic 还可生成真正的静态方法供 Java 直接调用。


### 15、什么是 sealed class？

**sealed class 密封类**限制子类必须在同一个文件（或模块）内定义，编译器知道所有子类，因此 when 表达式可以穷尽所有分支而无需 else，配合智能转换实现封闭的类型体系，常用于表示有限的状态集合（如网络请求的成功/失败/加载中）。sealed interface 与之类似，子类可以是 class。新增子类时编译器会提示所有 when 分支需要更新，从编译期防止漏处理。


### 16、Kotlin 的属性委托是什么？

**属性委托**用 by 关键字把属性的 get/set 委托给另一个对象处理，如 `val x by lazy { ... }`、`val y by Delegates.observable(0) { _, old, new -> }`。自定义委托需实现 getValue/setValue 操作符方法（或实现 ReadOnlyProperty/ReadWriteProperty 接口），是 Kotlin 强大的元编程能力。常见委托还有 Delegates.vetoable（可否决赋值）、map 委托（`val name by map`）。

最常见的两个委托是 by lazy 和 lateinit：

**by lazy 懒加载**：只有第一次访问属性时才执行初始化 Lambda 并缓存结果，之后直接返回缓存值；默认线程安全（LazyThreadSafetyMode.SYNCHRONIZED 模式，用双重检查锁实现，代价是性能开销，可用 NONE 模式关闭线程安全换取性能）。只能用于 val（只读属性）。适合初始化开销大、且不一定用到的属性。

**lateinit 延迟初始化**：用于 var（可变）属性，声明时先不初始化，使用前手动赋值后才能访问，访问未初始化的 lateinit 属性会抛 UninitializedPropertyAccessException。它只能用于非空、非基本类型的 var 属性（基本类型无 null 表示，无法标记未初始化；如 Android 的 viewBinding、依赖注入字段）。编译时 lateinit 属性先存 null，getter 中插入判空检查，访问未初始化时抛异常。可用 `::prop.isInitialized` 检查是否已初始化。

区别：**lazy 是 val + 首次访问自动初始化（惰性），lateinit 是 var + 手动延迟赋值（非惰性）**。选择标准：确定首次访问时初始化且之后不变用 lazy，声明时无法初始化但保证使用前赋值、且需要可变的用 lateinit。


### 17、Kotlin 的扩展函数是什么？

扩展函数是给已有类（包括第三方类、甚至不能修改的类）添加新函数的能力，声明形式为 `fun ReceiverType.functionName(...)`，调用时像调用该类的成员函数一样，如 `"abc".isBlank()`。

**本质是静态分发**：编译后扩展函数被编译为一个普通静态函数，接收者作为第一个参数传入，调用点被改写为调用这个静态函数。顶层扩展函数编译到名为 `XxxKt` 的类中（以文件名+Kt 命名，可用 @file:JvmName 改名）。因此扩展函数并没有真正修改原类、不能访问私有成员、也不会被子类真正重写——调用哪个扩展函数取决于变量的静态类型而非运行时类型，这是和成员函数多态的重要区别。

扩展属性同理，本质是静态的 get/set 方法（扩展属性没有 backing field，不能直接初始化）。

扩展函数的**限制**：

1. 不能访问接收者的私有/受保护成员。
2. 和成员函数同名时**成员函数优先**（扩展函数被遮蔽）。
3. 是静态分发，没有运行期多态。
4. 扩展函数的作用域受 import 控制，跨包使用需显式 import。
5. 扩展函数的接收者若为可空类型（`fun String?.isNullOrBlank()`），函数体内需处理空值。

扩展函数常用于为第三方库添加便捷方法、组织工具函数、实现 DSL，但不应用于真正需要多态的场景。


### 18、作用域函数 let、run、with、apply、also 有什么区别？

作用域函数用于在对象上执行一段代码块，区别在于两点：**上下文对象是 it 还是 this，返回值是 Lambda 结果还是对象本身**。

| 函数 | 上下文对象 | 返回值 | 典型用途 |
| --- | --- | --- | --- |
| let | it | Lambda 结果 | 非空判断后处理（?.let）、变量改名、转换 |
| run | this | Lambda 结果 | 配置对象并计算返回值 |
| with | this | Lambda 结果 | 对同一对象连续多次操作（非扩展函数，接收者作参数） |
| apply | this | 对象本身 | 建造者式链式配置，初始化后返回对象 |
| also | it | 对象本身 | 附加副作用（日志、校验）不改变返回值 |

按上下文对象分：let 和 also 用 it（可作为参数改名）；run、with、apply 用 this（接收者，可省略）。按返回值分：let、run、with 返回 Lambda 结果（用于转换）；apply、also 返回对象本身（用于链式或副作用）。

**底层实现**：这五个都是内联的标准库函数。let 是 `inline fun <T, R> T.let(block: (T) -> R): R = block(this)`，apply 是 `inline fun <T> T.apply(block: T.() -> Unit): T { block(); return this }`，等等。因为是 inline，Lambda 不会产生额外对象，调用链性能友好。

选择建议：需要非空处理 + 转换用 let；对象初始化并返回它用 apply；对某对象做多步操作并得到结果用 run 或 with（with 用于把对象当参数、适合接收者是表达式时）；需要附加打印日志/校验又不想改返回值用 also。易错点：嵌套作用域函数时 it/this 会被遮蔽，必要时用参数改名（`?.let { user -> }`）或 this@label 消除歧义。


### 19、只读集合和可变集合有什么区别？

Kotlin 把集合分为只读接口和可变接口：只读集合（List、Set、Map）只提供读取操作，可变集合（MutableList、MutableSet、MutableMap）继承只读接口并增加写操作。**注意只读不等于不可变**——它只是接口层面不提供写方法，底层仍可能是可变对象（如 listOf 返回的实现仍可被强转为 MutableList）。生产代码推荐用只读接口暴露数据，防止外部误改，但若要绝对不可变需用不可变集合实现或防御性拷贝。


### 20、Sequence 和集合的流式操作有什么不同？

**Sequence 惰性求值**：集合的流式操作是急切的（eager），每个操作符（map/filter）立即执行并产生中间集合；而 Sequence 的每个元素会依次走完整条操作链（水平评估），直到终端操作（toList、first、count 等）才触发执行，中间不产生中间集合。因此数据量大、操作链长时，Sequence 更省内存、性能更好，尤其适合「过滤后取前几个」这类可提前终止的场景（`list.asSequence().filter{...}.first()` 找到第一个就停，而集合版会过滤全部）。

底层：Sequence 是惰性迭代器，通过 asSequence() 把集合转为 Sequence，中间操作只是包装迭代器、不真正遍历。数据量小、链短时两者差别不大，用集合即可（集合操作内联优化、无 Sequence 包装开销）。它与 Java Stream 类似但更轻量、可在 Android 上使用（Java 8 Stream 在旧 Android 需兼容库）。


### 21、out 协变和 in 逆变是什么？

型变描述的是泛型类型之间的子类型关系是否随类型参数传递。

**协变 out**：如果 A 是 B 的子类型，那么 `Producer<A>` 是 `Producer<B>` 的子类型，子类型关系保持不变。**逆变 in**：子类型关系反转，如果 A 是 B 的子类型，那么 `Consumer<B>` 是 `Consumer<A>` 的子类型。

**out 协变**用于只产出（返回）T 的类型，如 `List<out T>` 只读集合，因此 `List<String>` 可赋值给 `List<Any>`（类型安全，因为只会从中读，读出的 String 是 Any 的子类）；**in 逆变**用于只消费（接收）T 的类型，如 `Comparator<in T>`，`Comparator<Any>` 可赋值给 `Comparator<String>`（能比较 Any 就能比较 String）。

型变遵循 PECS 原则（Producer-Extends, Consumer-Super）：生产者（输出）用 out，消费者（输入）用 in。编译器会用型变校验防止「协变类型作参数、逆变类型作返回值」这类破坏类型安全的用法，极端场景可用 @UnsafeVariance 绕过（但需自担风险）。

**声明处型变 vs 使用处型变**：声明处型变是在定义类/接口时就声明 out/in，如 Kotlin 标准库的 `interface List<out T>`、`Comparable<in T>`；使用处型变是在使用时用类型投影限定，即 Java 的通配符——`MutableList<out T>` 对应 Java 的 `? extends T`（协变，只能读），`MutableList<in T>` 对应 Java 的 `? super T`（逆变，只能写）。Kotlin 的 star projection（`MutableList<*>`）对应 Java 的 `?` 无界通配符。声明处型变让 API 定义更简洁（Java 需在每次使用处写通配符），是 Kotlin 泛型优于 Java 泛型的体现之一。


### 22、运算符重载是什么、怎么使用？

**运算符重载**允许自定义类型支持 +、-、==、in、[] 等运算符，实现方式是定义对应的约定函数：plus 对应 +、minus 对应 -、times 对应 *、div 对应 /、contains 对应 in、get/set 对应 []、compareTo 对应比较运算符、invoke 对应 () 调用、rangeTo 对应 ..、unaryMinus 对应一元负号等。这些函数用 operator 修饰，如 `operator fun plus(other: T): T`。典型例子是复数的加法、自定义集合的索引访问。注意 == 和 === 不能被重载（== 已绑定 equals），只有算术、比较、下标等可重载。

运算符重载是 Kotlin 语法糖，本质仍是普通函数调用，编译后不变——约定函数就是普通方法，编译器把运算符表达式翻译为方法调用，只是让代码更简洁、更具可读性，常用于定义 DSL 和数学/领域模型。易错点：运算符重载要保持语义一致（如 plus 不应有副作用），否则代码难读。


### 23、中缀函数是什么、怎么使用？

**中缀函数**用 infix 修饰，允许用「a 函数 b」的形式调用（省略点和括号），函数必须只有一个参数、且是成员函数或扩展函数，如 `infix fun Int.to(other: Int)`、`1 to "a"`（Map 的键值对构造）。中缀函数的优先级低于算术运算符，和常规调用等价。

中缀函数是 Kotlin 语法糖，本质仍是普通函数调用，编译后不变，只是让代码更简洁、更具可读性。易错点：中缀函数参数只能一个，命名要清晰表意。


### 24、Kotlin 和 Java 互操作有哪些注意点？

互操作是 Kotlin 的核心优势，但有几个注意点：

1. **平台类型**：Java 返回值在 Kotlin 中被视为可空也可能非空的平台类型（T!），不做强校验，需自行判断空值；Java 用 @Nullable/@NotNull 注解可让 Kotlin 识别空性。
2. **空安全**：Kotlin 对 Java 代码的 null 不设防，调用 Java 方法返回 null 时若未判空会 NPE。
3. **受检异常**：Java 的 checked exception 到 Kotlin 变成非受检，不必 try-catch；反向要让 Java 感知 Kotlin 抛出的异常需用 @Throws 注解。
4. **伴生对象/顶层函数**：Kotlin 顶层函数编译到 XxxKt 类（可用 @file:JvmName 改名）；伴生对象成员在 Java 侧需通过 Companion 访问。
5. **SAM 转换**：Kotlin 的 Lambda 可自动转换为 Java 的函数式接口（Runnable、Callback 等），但 Java 的 SAM 接口到 Kotlin 不一定能传 Lambda（需 Kotlin 接口或用 typealias）。
6. **关键字冲突**：Kotlin 关键字（如 when、object、is）在 Java 中叫同名方法时，需用反引号 `when` 转义。

有三个注解专门解决 Java 侧调用 Kotlin 代码的兼容问题：

- **@JvmStatic**：把伴生对象或 object 的方法编译为真正的静态方法，让 Java 能直接类名调用而不用写 Companion。
- **@JvmOverloads**：为有默认参数的函数生成多个重载方法，让 Java 能省略默认参数。
- **@JvmField**：把属性暴露为 Java 的 public 字段而非 getter/setter（避免 Kotlin 属性的 getter/setter 命名与 Java 字段期望不一致，常用于常量、Parcelable 等）。

这些注解都是为了抹平两种语言在字节码层面的差异。此外还有 @JvmName 改名、@Throws 声明异常、@JvmInline 等，需要时按需使用。


### 25、suspend 关键字是什么？

**suspend 挂起函数**：suspend 修饰的函数可以在执行过程中挂起而不阻塞线程，挂起后线程被释放去做别的事，恢复时从挂起点继续执行。挂起函数只能被协程或其他挂起函数调用。**底层是 CPS（Continuation-Passing Style）变换 + 状态机**：编译器把挂起函数编译成一个带 Continuation 参数的状态机对象，每个挂起点对应状态机的一个状态；挂起就是返回 COROUTINE_SUSPENDED 信号并保存 Continuation，恢复就是通过 resumeWith 回调重新进入状态机。因此协程的挂起不阻塞线程，一个线程可承载成千上万个协程（区别于线程：挂起释放的是逻辑执行流而非物理线程）。协程需要调度器（Dispatchers.Main/IO/Default）决定在哪个线程执行。


### 26、什么是结构化并发？

**结构化并发**是协程的父子层级模型：在某个 CoroutineScope 中启动的子协程挂在父 Job 之下，父协程结束时等待所有子协程完成，父协程取消时子协程一并取消。这样作用域（scope）划定了生命周期边界，不会出现「协程泄漏」，取消和异常处理都有明确归属（子协程异常默认向上传播取消父协程，可用 SupervisorJob 阻止传播）。典型如 viewModelScope、lifecycleScope 随生命周期自动取消。


### 27、协程的取消机制是怎样的？

**协程的取消是协作式的**：cancel() 只给 Job 打上取消标记，不会强行中断正在执行的代码，协程代码必须在挂起点（delay、yield、withContext）或显式检查 isActive 时才响应取消并抛出 CancellationException。CancellationException 被视为正常流程，不会触发异常处理器。因此纯 CPU 的 while 循环没有挂起点时 cancel 无效，需配合 isActive 或 ensureActive 检查。此外 withContext(NonCancellable) 可在取消后执行必要的清理逻辑；finally 块中调用挂起函数需谨慎。


### 28、协程的启动方式 launch 和 async 有什么区别？

launch 和 async 都是 CoroutineScope 的扩展函数，用来启动一个协程，核心区别在于**返回值**：launch 返回 Job（无结果，适合"发射后不管"的副作用型任务），async 返回 Deferred\<T\>（有结果，适合需要返回值的并发计算）。Deferred 继承自 Job，因此 async 也受结构化并发与取消约束，通过 await() 挂起等待并取得结果。

| 维度 | launch | async |
| --- | --- | --- |
| 返回值 | Job（不携带结果） | Deferred\<T\>（携带结果，await() 取值） |
| 用途 | 副作用型任务（更新 UI、写日志、不关心返回值） | 并行计算后聚合结果 |
| 类比 | Java 的 `new Thread().start()` | Java 的 Future / CompletableFuture |

**并行求值的典型写法**：两个耗时请求用 async 并行，再 await 合并，总耗时取二者较长者而非相加：

```kotlin
coroutineScope {
    val a = async { fetchA() }   // 两个 async 并行执行
    val b = async { fetchB() }
    val sum = a.await() + b.await()  // await 时才挂起等待结果
}
```

要点与易错点：
- async 默认是**立即调度**（CoroutineStart.DEFAULT），调用即开始执行；传 `start = CoroutineStart.LAZY` 可改为惰性，只有调用 start() 或 await() 时才真正执行。
- **易错点 1**：async 里抛异常默认会取消父协程，除非用 supervisorScope 隔离或用 try-catch 包住 await；若 async 启动了却从不 await，异常会被静默吞掉，极易漏掉失败。
- **易错点 2**：忘了 await 意味着"结果没被消费"，此时 async 协程虽执行完，但若父作用域早已结束，异常无人处理。
- 底层：二者都通过 CoroutineScope 启动 AbstractCoroutine 的子类——launch 是 StandaloneCoroutine，async 是 DeferredCoroutine（额外实现 Deferred 接口、持有结果 state），最终都挂到当前 Job 下成为子协程。


### 29、CoroutineScope 和 GlobalScope 有什么区别？

CoroutineScope（协程作用域）是协程的组织容器，持有一个 CoroutineContext（尤其是 Job），用来统一管理作用域内所有协程的生命周期、取消与异常。GlobalScope 是一个特殊的全局顶层作用域，没有父 Job，独立于任何组件生命周期。

| 维度 | CoroutineScope（自定义/组件提供） | GlobalScope |
| --- | --- | --- |
| 生命周期 | 跟随持有它的组件（Activity、ViewModel），组件销毁时 cancel 即取消所有子协程 | 与应用进程同生命周期，永不自动取消 |
| 是否推荐 | 推荐，结构化并发的基础 | 官方明确不推荐（Kotlin 文档标注"不鼓励使用"） |
| 典型场景 | viewModelScope、lifecycleScope、自定义 `CoroutineScope(Dispatchers.IO + Job())` | 几乎不用，仅测试或极少数进程级长驻任务 |

**GlobalScope 的问题**：用它启动的协程不挂任何父 Job，即使界面销毁、ViewModel 清理，协程仍继续运行，造成内存泄漏、资源浪费、无法统一取消；异常也不会自动向上传播到任何处理器。因此 Android 上应优先用 viewModelScope / lifecycleScope，或自定义 CoroutineScope 并在组件销毁时调用 `scope.cancel()` 统一回收。

**易错点**：把 CoroutineScope 当成"类"去继承实现是反模式——正确做法是"持有"一个作用域（`class Foo { private val scope = CoroutineScope(SupervisorJob() + Dispatchers.Default) }`），并在不再需要时 cancel。取消作用域会级联取消其 Job 及所有子孙协程，这是清理协程泄漏的核心手段。


### 30、Dispatchers.Main、IO、Default 有什么区别？

Dispatcher（调度器）决定协程在哪个线程（或线程池）上执行，是 CoroutineContext 的核心元素之一。三大内置调度器：

- **Dispatchers.Main**：主线程调度器（Android 上是 UI 线程），用于 UI 更新与交互，是唯一能碰 UI 的调度器；在纯 JVM 上需引入 kotlinx-coroutines-android / Swing 对应库才有实现。
- **Dispatchers.IO**：为 IO 密集型阻塞操作（网络、磁盘、数据库、文件读写）设计的弹性线程池，默认上限 64 线程，内部与 Default 共享线程并做"线程饥饿"下的动态扩容；阻塞任务应切到 IO，避免占用计算线程。
- **Dispatchers.Default**：CPU 密集型计算（JSON 解析、排序、图像处理、大集合 map/filter），默认线程数 = CPU 核心数（至少 2），用共享线程池，超出核心数会排队。

选择原则：阻塞 IO → IO；纯计算 → Default；触碰 UI → Main；测试用 runTest 或 Unconfined。

**withContext 切换**：协程内用 `withContext(Dispatchers.IO) { ... }` 临时切换调度器执行阻塞代码，完成后自动切回原调度器，是替代"手动切线程 + 回调"的惯用法。注意 withContext 本身是挂起函数，不会阻塞调用它的线程。

**易错点**：
- Dispatchers.IO 与 Default 在 JVM 上共享线程池（IO 复用 Default 的线程池做弹性扩容），但语义要严格区分——CPU 密集计算放进 IO、阻塞 IO 放进 Default 都会拖垮对方。
- withContext 切线程不等于创建新线程，它只是把挂起函数的恢复点重新投递到目标调度器的队列，开销远小于线程切换。
- 协程的"线程"由调度器决定，挂起与恢复可能在不同线程执行，因此**不能在协程里假设线程连续性**（切线程后访问 ThreadLocal 会失效、thread confinement 被打破）。


### 31、协程的异常是怎么传播和处理的？

协程异常处理遵循结构化并发的规则：子协程异常默认**向上传播**，最终取消父协程及其所有兄弟子协程；根协程（顶层协程）的未捕获异常交给其 CoroutineExceptionHandler 处理，若没有则抛到当前线程的默认异常处理器（Android 上通常 crash）。

处理方式按协程类型区分：

**launch 类（不关心结果）**：子协程在挂起点之后抛出的异常无法被外层 try-catch 捕获，要用 CoroutineExceptionHandler（安装到根作用域的上下文）统一兜底，或用 SupervisorJob 阻止异常向上传播。

```kotlin
val handler = CoroutineExceptionHandler { _, e -> log(e) }
val scope = CoroutineScope(SupervisorJob() + Dispatchers.Main + handler)
scope.launch { throw RuntimeException() }  // 由 handler 兜底，不 crash 也不取消兄弟
```

**async 类（关心结果）**：异常被 Deferred 保存，在调用 await() 时重新抛出，因此用 try-catch 包住 await() 即可捕获；但若 async 从不 await，异常会被静默吞掉。

**SupervisorJob 与 supervisorScope**：SupervisorJob 让异常**不向上传播**，一个子协程失败不影响兄弟协程；普通 Job 下子协程失败会取消整个作用域。supervisorScope 是作用域版的 SupervisorJob，用于局部"隔离失败"。

**易错点**：
- CancellationException 是"正常"取消，不触发异常处理器，也不被当作失败向上传播。
- CoroutineExceptionHandler 只对**根协程**（直接在 scope 上 launch 的协程）生效，装在子协程 context 里无效，且对 async 无效。
- try-catch 只能捕获"同步抛出的异常"，对 launch 内挂起点之后抛出的异常无能为力——这是最常见的误区。


### 32、挂起函数和阻塞函数有什么区别？

**阻塞函数**（如 Thread.sleep、同步 IO）会占用当前线程，线程在阻塞期间无法做任何其他事，只能干等；高并发只能靠创建大量线程，线程栈内存开销大（默认约 1MB/线程）、上下文切换成本高。

**挂起函数**（suspend 修饰）在遇到挂起点时**让出线程而不阻塞它**，线程被释放去执行别的任务，等结果就绪后通过 Continuation 恢复执行（可能换线程）。挂起释放的是"逻辑执行流"，不是物理线程。

| 维度 | 阻塞（Thread.sleep） | 挂起（delay） |
| --- | --- | --- |
| 线程占用 | 占用当前线程，线程空转等待 | 不占用，线程可干别的 |
| 资源开销 | 一个任务一个线程，栈内存 + 切换开销大 | 一个线程可承载大量协程 |
| 恢复 | 无需恢复，醒来继续 | 靠 Continuation 状态机恢复，可能换线程 |

**底层**：挂起函数编译为 CPS 状态机（见 suspend 题），每个挂起点把状态与 Continuation 保存下来并返回 COROUTINE_SUSPENDED，所以"挂起"只是"返回 + 保存回调"，线程并不阻塞。这正是协程"同步写法、异步执行"的本质。

**易错点**：挂起 ≠ 一定切线程。delay 会挂起，但 `suspend fun foo() = 1` 没有挂起点，实际同步执行完；切不切线程由调度器决定。反过来，挂起函数内部若调用了阻塞代码（如 Thread.sleep）仍会阻塞当前线程——真正非阻塞要靠 delay、挂起式 IO 库（如 OkHttp 的 suspend 扩展）等。


### 33、Flow 的冷流和热流（StateFlow/SharedFlow）有什么区别？

Flow 是 Kotlin 协程的响应式流 API，用于处理一串异步产生的数据。按"是否有独立的生产者"分为冷流和热流：

**冷流（Flow）**：每个收集器 collect 时才启动独立的生产者，数据从头开始发射；多个收集器互不共享、互不干扰，收集器停止时上游也停止。类似 Java 8 Stream / RxJava 的 Cold Observable。

**热流（StateFlow / SharedFlow）**：生产者在收集器之外独立存在，数据持续发射，收集器只能收到"订阅之后"的数据，多个收集器共享同一份数据源，类似 LiveData / RxJava 的 Hot Observable。

| 维度 | Flow（冷流） | StateFlow（热流） | SharedFlow（热流） |
| --- | --- | --- | --- |
| 触发 | collect 时才生产 | 独立于收集器持续存在 | 独立于收集器持续存在 |
| 共享 | 每收集器独立一份 | 全收集器共享同一状态 | 全收集器共享同一数据 |
| 数据 | 从头发射 | 只保留最新值，新收集器拿当前值 | 可配 replay / extraBufferCapacity，可丢事件 |
| 典型场景 | 数据库/网络数据流、链式转换 | 单一状态源（替代 LiveData，UI 状态） | 事件流（一次性事件、广播，如点击、导航） |

**StateFlow**：始终持有当前值（value 属性），天然去重（conflate，新值等于旧值则不发射），永远只有一个最新值，适合 UI 状态；冷流转热用 `flow.stateIn(scope, SharingStarted.WhileSubscribed(5000), initial)`。

**SharedFlow**：可配置 replay（重放最近 N 个给新订阅者）、extraBufferCapacity（缓冲区）、onBufferOverflow（缓冲策略），默认无缓存，适合事件流——注意事件用 replay=0，否则界面重建时会把旧事件重放导致重复消费。

**易错点**：StateFlow 因 conflate 会丢中间状态（快速连发只保留最新）；事件应选 SharedFlow(replay=0) 而非 StateFlow，否则"已消费"状态无法表达；Flow 是冷的，没有 collect 上游操作不会执行（惰性）。


### 34、协程为什么比线程轻量？底层调度原理是什么？

协程是**用户态**的轻量并发单元，线程是**内核态**的调度实体。轻量的本质在于"挂起不阻塞线程"以及"协程调度不经过操作系统"。

| 维度 | 线程（内核态） | 协程（用户态） |
| --- | --- | --- |
| 调度者 | 操作系统内核调度器（抢占式、时间片） | 应用层的 Dispatcher（协作式、靠挂起点让出） |
| 切换开销 | 用户态↔内核态切换 + 保存/恢复寄存器上下文 | 仅函数调用级别，保存 Continuation 状态机 |
| 内存 | 每个线程独立栈，默认约 1MB | 协程只需一个小对象 + 状态机字段，几 KB 级 |
| 数量 | 受内存与系统上限限制（几百~几千） | 一个线程可承载成千上万个协程 |
| 阻塞 | 阻塞即占死线程 | 挂起让出线程，不阻塞 |

**核心原理**：
- 线程由 OS 内核调度，阻塞（如 IO 等待）时线程被挂起，需要保存完整上下文、陷入内核态，切换成本高；线程栈是预分配的固定内存。
- 协程由 Kotlin 标准库的 Dispatcher 在用户态调度，挂起只是"保存状态 + 返回"，把 Continuation 交给调度器，线程立刻去跑别的协程，不涉及内核调度；恢复时调度器把 Continuation 投递回线程队列继续执行。

**为什么能"同步写、异步跑"**：编译器把挂起函数转成 CPS 状态机（见 suspend 题），每个挂起点拆成独立状态，挂起时返回 COROUTINE_SUSPENDED 并持有 Continuation，真正的线程被释放；等异步结果就绪，调度器调用 resumeWith 重新驱动状态机从断点继续，对外呈现为"顺序执行的同步代码"。

**易错点**：协程轻量 ≠ 不用线程。协程最终仍运行在线程上，Dispatcher 底层是线程池；"一个线程跑一万个协程"是因为大部分协程处于挂起等待、不占用线程，而不是一个线程同时执行一万个协程。CPU 密集的协程不会更省 CPU，省的是"等待期间的线程占用"。


### 35、协程的 Job 状态机有哪些状态？是怎么流转的？

Job 是协程的句柄，代表一个可取消的、有生命周期的任务，其状态机由六个状态组成（JobState）：

- **New（新建）**：协程创建但尚未启动（如 CoroutineStart.LAZY 的协程）。
- **Active（活跃）**：协程已启动、正在执行或挂起等待中。
- **Completing（完成中）**：协程主体执行完毕，但还在等待子协程全部完成（结构化并发要求父协程等所有子协程）。
- **Cancelling（取消中）**：已发起取消，正在等待子协程完成取消（可执行 finally 清理）。
- **Cancelled（已取消）**：取消完成（含子协程）。
- **Completed（已完成）**：正常完成（含子协程）。

状态流转：

```
New ── start ──> Active ── 主体完成 ──> Completing ── 子协程完成 ──> Completed
                    │                              │
                    └── cancel() ──> Cancelling ──> Cancelled
```

关键点：
- 中间态 Completing / Cancelling 的存在，正是因为结构化并发：父协程要"等子协程跑完"才算真正结束，所以有了"主体完成但子协程未完"的中间态。
- 可通过 isActive（Active 时为 true）、isCompleted（Cancelled/Completed 为 true）、isCancelled（Cancelling/Cancelled 为 true）等属性查询。
- 状态变更会触发 completion 回调（invokeOnCompletion），常用于释放资源；状态一经进入终态（Cancelled/Completed）不可回退。

**易错点**：
- 不要在 Job 完成回调里做耗时/挂起操作；completion 回调在 completion 线程上执行。
- Job 取消后若协程不配合（无挂起点），状态会停在 Cancelling 直到协程响应取消——这是"取消是协作式"在状态机层面的体现。
- 内部 Job 子类（JobSupport）用 state 字段 + CAS 管理状态转换，保证并发安全；这与 AQS 用 state + CAS 类似。


### 36、CoroutineContext 的实现原理是什么？

CoroutineContext 是协程的上下文，承载 Job、Dispatcher、CoroutineName、CoroutineExceptionHandler 等元素，决定协程在哪个线程跑、怎么取消、异常给谁处理。它的实现核心是**"组合键查找"**。

**接口设计**：CoroutineContext 内部是一个 Element 集合，每个 Element 本身也是一个 CoroutineContext。核心方法：

```kotlin
interface CoroutineContext {
    operator fun <E : Element> get(key: Key<E>): E?   // 按 Key 取元素
    operator fun plus(context: CoroutineContext): CoroutineContext  // 合并两个上下文
    fun <R> fold(initial: R, operation: (R, Element) -> R): R  // 遍历所有元素
}
```

**原理要点**：
- **Key 唯一**：每个元素类型对应一个伴生对象 Key（如 Job.Key、CoroutineDispatcher.Key），上下文里同类型元素最多一个——这是"用 Key 精确查找"的基础。
- **Element 链 / 组合**：多个元素用 plus 合并时，底层组合成 CombinedContext（链表结构，左孩子是元素、右孩子是剩余上下文），get 时沿链遍历匹配 Key。`Dispatchers.Main + Job()` 就是一个"Dispatcher 元素 + Job 元素"的组合上下文。
- **plus 的覆盖规则**：右侧上下文的元素若与左侧同 Key，会**覆盖**左侧（右侧优先），例如 `Dispatchers.IO + Dispatchers.Main` 结果是 Main 生效。
- **继承与传播**：启动子协程时，子协程默认继承父协程上下文（newContext = parentContext + 子协程额外元素），但 Job 例外——子协程会拿到一个**新的 Job**（child Job），从而建立父子关系。

**易错点**：
- 用 `context[Job]`、`context[CoroutineDispatcher]` 取元素时，Key 是元素的伴生对象，而不是元素类型本身；自定义上下文元素必须提供 `companion object Key : CoroutineContext.Key<MyElement>`。
- CoroutineContext 是不可变的，plus 返回新上下文而不是原地修改。
- 子协程的 Job 是新对象，所以判断"某个协程是否被取消"要看自己上下文的 Job，而不是父的 Job。


### 37、协程调度器 Dispatcher 是如何调度协程的？

Dispatcher 决定协程在哪个线程执行，其本质是**"把协程的续体（Continuation）投递到某个线程队列"**。核心接口是 CoroutineDispatcher，继承自 ContinuationInterceptor：

```kotlin
abstract class CoroutineDispatcher {
    abstract fun dispatch(context: CoroutineContext, block: Runnable)
}
```

**调度原理**：协程要恢复执行时（挂起结束、被唤醒），调度器把"续体 Runnable"通过 dispatch 投递到目标线程的队列；目标线程的循环不断从队列取任务执行。dispatch 只负责"投递"，不负责执行。

内置调度器的实现机制：

- **Dispatchers.Main**：Android 上基于主线程的 Looper/Handler 实现，dispatch 就是向主线程 Handler post 一个 Runnable，保证 UI 操作在主线程执行。
- **Dispatchers.Default**：基于 JVM 共享线程池（CoroutineScheduler），线程数 = CPU 核心数，用无锁队列 + 任务窃取（work-stealing）平衡负载，适合 CPU 密集。
- **Dispatchers.IO**：复用 Default 的线程池但做弹性扩容（默认上限 64），当检测到线程饥饿（Default 线程全被阻塞）时动态加线程，避免阻塞任务饿死计算任务；有超时回收机制。
- **Dispatchers.Unconfined**：不切换线程，直接在调用线程执行到第一个挂起点，恢复后继续在 resume 的线程执行（线程不固定），仅测试用。

**挂起恢复的完整链路**：协程挂起 → 异步任务完成 → 调用 Continuation.resumeWith 恢复 → 该续体被包装成 Runnable 交给 Dispatcher.dispatch → 投递到目标线程队列 → 目标线程取出并驱动状态机继续执行。这就是"挂起不阻塞、恢复换线程"的底层机制。

**易错点**：
- dispatch 不保证立即执行，它只是入队；所以 withContext 切线程后，代码恢复执行的"时机"由队列调度决定。
- Unconfined 让线程不固定，容易出现"前半段在主线程、后半段在子线程"的隐蔽问题，生产代码禁用。
- 自定义调度器要正确处理取消（拦截器要配合 ensureActive），且避免在 dispatch 里做重活（会阻塞投递线程）。


> 附：高频速记 —— val 引用不可变、var 可重赋值；?. 安全调用、?: 兜底、!! 断言抛 NPE；== 比内容（Intrinsics.areEqual）、=== 比引用；when 是表达式不用 break；let/also 用 it，run/with/apply 用 this；let/run/with 返回结果，apply/also 返回对象；inline 内联免对象、reified 具体化类型；data class 自动生成方法、object 单例、sealed 穷尽分支；lazy 是 val 懒加载、lateinit 是 var 延迟赋值；扩展函数静态分发；out 协变只读、in 逆变只写；协程挂起不阻塞线程、取消是协作式；launch 返回 Job、async 返回 Deferred 用 await 取结果；GlobalScope 不推荐、用 viewModelScope/lifecycleScope；Main 主线程、IO 阻塞、Default 计算；SupervisorJob 隔离异常；Flow 冷流、StateFlow 单一状态、SharedFlow 事件流；协程是用户态调度、挂起不占线程；Job 六状态 New/Active/Completing/Cancelling/Cancelled/Completed；CoroutineContext 按 Key 取元素、plus 右覆盖左；Dispatcher 把续体 Runnable 投递到线程队列。
