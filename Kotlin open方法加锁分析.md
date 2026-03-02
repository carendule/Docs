# Kotlin open方法加锁分析


##### User:


kotlin中open fun定义的方法，如果给这个open fun 方法加上 @Synchronized 的注解，它还本身和重写了它的子方法都会是加锁的方法吗？



##### Assistant:


这是一个非常经典且重要的并发编程问题，特别是在 Android (AOSP) 开发中经常遇到。

直接回答你的问题：**不，锁不会被继承。**

具体情况如下：

1.  **基类的方法（Base Method）：** 是**加锁**的。
2.  **重写的子类方法（Overridden Method）：** 默认是**不加锁**的。

### 详细原理解析

在 Kotlin 中，`@Synchronized` 注解的作用是在编译生成 Java 字节码时，给该方法加上 `synchronized` 关键字。

然而，在 JVM（Java 虚拟机）规范中，`synchronized` 属于**实现细节（Implementation Detail）**，而不属于**方法签名（Method Signature）**的一部分。

这意味着：
*   **父类**：当你调用父类的这个 `open fun` 时，进入该方法前会获取对象锁（Monitor）。
*   **子类**：当你重写（Override）这个方法时，新的方法仅仅是继承了函数名和参数列表，但**没有继承同步特性**。子类的方法体在默认情况下是裸奔的（非线程安全）。

### 代码演示

让我们看一个具体的例子：

```kotlin
open class Base {
    @Synchronized
    open fun doWork() {
        println("Base: Start ${Thread.currentThread().name}")
        Thread.sleep(1000) // 模拟耗时操作
        println("Base: End ${Thread.currentThread().name}")
    }
}

class Child : Base() {
    // ⚠️ 注意：这里没有加 @Synchronized
    override fun doWork() {
        println("Child: Start ${Thread.currentThread().name}")
        super.doWork() // 调用父类方法
        println("Child: End ${Thread.currentThread().name}")
    }
}
```

#### 此时的行为分析：

1.  如果多线程同时调用 **`Base().doWork()`**：
    *   **结果**：它是线程安全的，线程会排队执行。

2.  如果多线程同时调用 **`Child().doWork()`**：
    *   **结果**：它是**非**线程安全的。多个线程可以同时进入 `Child` 的 `doWork` 方法体。
    *   **特殊情况**：当代码执行到 `super.doWork()` 这一行时，线程会尝试获取锁。但是，在 `super.doWork()` 之前和之后的 `Child` 类自己的逻辑，完全没有锁保护。

### 应该怎么做？

如果你希望子类重写的方法也具备原子性（全方法同步），你须显式地在子类方法上也加上注解：

```kotlin
class Child : Base() {
    @Synchronized // ✅ 必须手动再加一次
    override fun doWork() {
        // 现在的整个方法体都是线程安全的
        super.doWork() 
    }
}
```

### 专家建议 (AOSP 视角)

在阅读或编写 AOSP 源码时，你会发现 Framework 层大量使用了同步锁（如 `ActivityManagerService` 或 `WindowManagerService`）。

*   **设计原则**：尽量避免在 `open` 方法上直接使用 `synchronized` / `@Synchronized`。
*   **推荐做法**：使用内部锁对象（Review AOSP 源码时常见 `synchronized(mLock)` ）。这样做的好处是锁的范围更可控，且子类重写时，你能清楚地知道是在哪个对象上加锁，避免因为继承关系导致的“部分同步”产生的并发 Bug。

**总结：** 只要发生了重写（Override），子类就像是写了一个全新的方法，父类的 `@Synchronized` 对子类方法体本身**失效**，必须由开发者决定是否在子类再次添加同步机制。



##### User:


如果我想让open方法和它的子类都自动有加锁功能，应该怎么办



##### Assistant:


如果想实现“强制所有子类的业务逻辑都自动运行在锁内部”，依靠语言特性（单纯加注解）是做不到的。

最好的解决方案是使用 **模版方法模式 (Template Method Pattern)**。

这种设计模式的核心思想是：**父类控制“流程和锁”，子类只负责“填空业务逻辑”**。

### 核心方案：Final + Protected Open

你需要将原来的方法拆分为两个：
1.  **公共入口方法 (Public Entry)**：加上 `final` 和 `@Synchronized`。这是外界调用的唯一入口，`final` 禁止子类破坏锁机制。
2.  **业务钩子方法 (Hook Method)**：声明为 `protected open`。这是子类真正写代码的地方。

#### 代码示例

```kotlin
abstract class BaseEngine {

    // 1. 【公共入口】：加锁 + 禁止重写 (final)
    // 外界只能调用这个方法，它保证了锁一定存在
    @Synchronized
    fun execute() {
        println("Core: 获取锁，准备执行... Thread:${Thread.currentThread().name}")
        
        try {
            // 在锁的保护下，调用子类的逻辑
            onExecute() 
        } finally {
            // 这里还可以做一些统一的清理工作
        }
        
        println("Core: 执行完毕，释放锁。")
    }

    // 2. 【子类实现】：开放给子类，但对外隐藏 (protected)
    // 子类只需要重写这个方法，不用管锁的事情
    protected open fun onExecute() {
        // 默认实现（可选）
        println("Core: 基类默认逻辑")
    }
}

// 子类
class CustomEngine : BaseEngine() {
    
    // 子类只能重写这个 onExecute
    // 不需要加 @Synchronized，因为它一定会被父类的 execute() 包裹在锁里
    override fun onExecute() {
        println("Custom: 正在处理子类业务... Thread:${Thread.currentThread().name}")
        // 模拟耗时
        Thread.sleep(500) 
    }
}
```

### 为什么这样做是最佳实践？

#### 1. 强制性安全 (Enforced Safety)
子类开发者不需要记忆“我是否需要加锁”。因为他们根本无法重写最外层的 `execute()` 方法（因为是 `final` 的）。只要通过 `execute()` 调用，锁就是强制生效的。

#### 2. 符合 AOSP/Android 系统服务设计标准
在 AOSP 源码中，这是非常经典的设计。例如 Android 的 `View` 系统或 `Activity` 生命周期管理，很多时候都是 Framework 层处理好状态管理（包括锁），然后回调 `onXXX()` 方法。

虽然 Framework 不一定全用 `synchronized` 修饰方法，但逻辑是一样的：

```java
// 伪代码：AOSP 风格
public void performAction() {
    synchronized (mLock) {
        // 核心检查
        if (!checkState()) return;
        
        // 调用子类或具体的实现
        onPerformActionLocked(); 
    }
}

// 命名通常会加上 Locked 后缀，提示这个方法是在锁里被调用的
protected abstract void onPerformActionLocked();
```

### 补充：Kotlin 专属的优雅写法

如果你不想写两个方法（一个 `fun` 一个 `onFun`），在 Kotlin 中还可以利用 **高阶函数** 配合内联锁来实现类似的效果，但继承结构上还是推荐上面的模版模式。

如果是为了代码整洁，且你的类不是为了继承设计的，可以使用 `block`：

```kotlin
// 这种方式适合工具类，不适合复杂的继承体系
fun <T> runInLock(action: () -> T): T {
    synchronized(this) {
        return action()
    }
}
```

**总结：**
不要试图让锁“继承”，而是通过 **`final @Synchronized` 父类方法** 包裹 **`protected open` 子类方法**，这是实现“自动加锁”唯一且最稳健的架构方案。



##### User:


如果我给这个abstract类加锁，那它的子类方法会有锁吗



##### Assistant:


这是一个非常关键的误区，答案依然是：**不会**。

而且这里还有一个更底层的语法限制：**你甚至不能给 `abstract` 方法加 `@Synchronized` 注解**。

我们需要分两种情况来彻底粉碎这个幻想：

### 情况一：试图给抽象方法（Abstract Method）加锁

如果你尝试这样写代码：

```kotlin
abstract class Base {
    // ❌ 编译报错！
    // Abstract methods cannot be synchronized
    @Synchronized 
    abstract fun doSomething()
}
```

**为什么不行？**
*   `abstract` 这意味着“**没有代码体**”。
*   `synchronized` 意味着“**在代码体执行前后**加锁”。
*   你不能给一个不存在的代码体加锁。JVM 无法对一个单纯的“接口定义”插入加锁指令。

所以，你不能强制要求子类在实现这个抽象方法时自动加锁。

---

### 情况二：给抽象类里的具体方法（Concrete Method）加锁

如果你在抽象类里写了一个带实现的具体方法，并加上了锁：

```kotlin
abstract class Base {
    // ✅ 这是一个具体方法，可以加锁
    @Synchronized
    open fun commonLogic() {
        print("Base Logic")
    }
}

class Child : Base() {
    // ⚠️ 只要你重写了它，锁就消失了
    override fun commonLogic() {
        print("Child Logic (No Lock Here)")
        super.commonLogic() // 只有执行到这一句时，才会获取锁
    }
}
```

**结果**：
和之前的普通类继承一样，**子类重写的方法体本身没有锁**。

---

### 所谓的“抽象类锁”到底锁的是什么？

即使你在抽象类里写了 `synchronized(this)`，你需要明白：

1.  **抽象类不能被实例化**。
2.  当你 `new Child()` 时，内存里只有**一个对象**（Child 的实例）。
3.  父类（抽象类）的代码和子类的代码，操作的都是这**同一个对象实例**。

**关键点来了：**
锁是保存在**对象头（Object Header）**里的。
*   如果父类方法声明了 `synchronized`，它是告诉 JVM：“执行这块父类代码前，检查一下当前对象的锁。”
*   如果子类方法没声明 `synchronized`，它是告诉 JVM：“直接执行，不用检查锁。”

**虽然也就是同一个对象，但子类方法“忘了”去检查锁，所以锁就没生效。**

### 解决方案：怎样才能“一劳永逸”？

既然不能给 `abstract` 方法加锁，想要强制子类有锁，依然只能用我上一条回答提到的 **模版方法模式（Template Method Pattern）**。

这是**唯一**能让抽象类的设计意图（必须加锁）强制覆盖到子类的方法：

```kotlin
// AOSP 风格的标准范式
abstract class SafeBase {

    // 1. 定义一个非 abstract 的、final 的入口
    // ⭐️ 锁加在这里！
    @Synchronized 
    fun safeDoAction() {
        // 这里的代码是在锁里的
        println("Holding lock on ${this}...")
        
        // 调用下面的抽象方法
        doActionInternal()
    }

    // 2. 定义抽象方法，强制子类去实现
    // 子类不需要管锁，因为只能通过上面的 braveDoAction 进来
    protected abstract fun doActionInternal()
}

class Child : SafeBase() {
    
    // 子类必须实现这个，而且 implementation 这里自动就处于锁环境中了
    override fun doActionInternal() {
        println("I am running inside the lock automatically!")
    }
}
```

**总结：**
*   不能给 `abstract` 方法加锁（编译不过）。
*   给抽象类里的普通方法加锁，子类重写后依然会丢失锁。
*   **不要试图“继承锁”，要学会“包裹锁”**（使用 Final Template Method）。

