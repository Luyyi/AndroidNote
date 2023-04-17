# Kotlin 委托

`委托模式`是设计模式的一种，用聚合代替继承。意思是有两个对象A和B，让A做某个事情，A将这个事情委托给B做。

委托可以减少很多重复代码。

Kotlin通过关键字`by`实现委托，分为`类委托`和`属性委托`。

### 1. 类委托

一个类定义了一个方法，但这个方法实际上是调用了另一个类的对象的方法。

```kotlin

// 定义一个print接口
interface IPrint {
    fun print()
}

// 定义一个class A，实现IPrint
// 构造方法传入了一个实现了IPrint接口的对象b
// 通过by关键字把A类委托给了b
class A(private val b: IPrint): IPrint by b

fun main() {
    val b = object : IPrint{
        override fun print() {
            println("我是b的print()")
        }
    }
    val a = A(b)
    // 调用a.print()，实际上调用了b.print()
    a.print()
}
```

类A实现了IPrint接口，应该要定义好print()的实现。但是类A通过`by`关键字把这个实现交给了b对象来处理。

反编译成Java
```java
// IPrint接口
public interface IPrint {
    void print();
}

// class A
public final class A implements IPrint {
    //构造方法传进来的对象b
    private final IPrint b;

    public A(@NotNull IPrint b) {
        Intrinsics.checkNotNullParameter(b, "b");
        super();
        this.b = b;
    }

    // 继承IPrint，实现的print()
    public void print() {
        //实际上调用了b.print()
        this.b.print();
    }
}
```

### 2. 属性委托

一个类的某个属性，它的值的定义委托给代理类。代理类必须有getValue()的实现，如果是var可变属性，代理类还必须有setValue()的实现。

#### （1）属性委托的语法

```
val/var <属性名>: <类型> by <表达式>

表达式：委托的代理类
```

#### （2）定义一个被委托的代理类

```kotlin
class Delegate {
    operator fun getValue(thisRef: Any?, property: KProperty<*>): String {
        return "$thisRef, 这里委托了 ${property.name} 属性"
    }

    operator fun setValue(thisRef: Any?, property: KProperty<*>, value: String) {
        println("$thisRef 的 ${property.name} 属性赋值为 $value")
    }
}
```

委托属性实际上是把这个属性的set/get方法委托代理类的setValue/getValue。所以被委托的代理类必须有setValue/getValue。

- 对于val，不可变的变量，代理类必须要有getValue()。
- 对于var，可变的变量，代理类必须有setValue()和getValue()

#### （3）定义一个委托属性
```kotlin
class Test {
    // test委托给Delegate()
    var test: String by Delegate()
}

fun main() {
    val test = Test()
    println(test.test)
    test.test = "测试"
}
```

输出:
```
com.demo.kotlindemo.Test@53d8d10a, 这里委托了 test 属性
com.demo.kotlindemo.Test@53d8d10a 的 test 属性赋值为 测试
```

也就说说，test.test的set/get最终调用的是Delegate()的setValue/getValue

#### （4）Kotlin标准库声明的属性委托

Kotlin中声明了两个包含operator方法的接口。被委托类继承接口就可以，更方便使用。

##### ① ReadOnlyProperty

只提供getValue，val属性的委托代理类实现它。
```kotlin
public fun interface ReadOnlyProperty<in T, out V> {
    public operator fun getValue(thisRef: T, property: KProperty<*>): V
}
```

##### ② ReadWriteProperty

提供setValue和getValue，var属性的委托代理类实现它。

```kotlin
public interface ReadWriteProperty<in T, V> : ReadOnlyProperty<T, V> {
    public override operator fun getValue(thisRef: T, property: KProperty<*>): V
    public operator fun setValue(thisRef: T, property: KProperty<*>, value: V)
}
```

</br>
### 3. 一些Kotlin库定义好的委托

#### （1）Lazy

`by lazy{}` 是平时用得很多的属性委托，用于val的延迟赋值。

使用：
```kotlin
val test by lazy {
    println("延迟初始化")
    "我是Test"
}

println(test)
println("---")
println(test)
```

输出：
```
延迟初始化
我是Test
---
我是Test
```

##### 可以看到`lazy`的特性
1. val声明的时候不会进行赋值
2. 首次调用到这个对象时，会执行lamda表达式，并且将结果记录下来
3. 下次再调用这个对象，不会执行lamda，会直接返回记录的结果值

##### 可以通过传入参数mode，控制`lazy`的创建模式

```kotlin
public actual fun <T> lazy(mode: LazyThreadSafetyMode, initializer: () -> T): Lazy<T> =
    when (mode) {
        LazyThreadSafetyMode.SYNCHRONIZED -> SynchronizedLazyImpl(initializer)
        LazyThreadSafetyMode.PUBLICATION -> SafePublicationLazyImpl(initializer)
        LazyThreadSafetyMode.NONE -> UnsafeLazyImpl(initializer)
    }
```

- LazyThreadSafetyMode.SYNCHRONIZED
    - 默认的模式
    - 同步锁：只在一个线程中计算，所有的线程会看到同样的值
- LazyThreadSafetyMode.PUBLICATION
    - 初始化的lambda可以在同一时间被调用多次
    - 但是只使用第一次被调用的返回值
- LazyThreadSafetyMode.NONE
    - 线程不安全

    
#### （2）Observable
`Delegates.observable()` 当变量`set()`被调用时，会触发`onChange`回调。

使用：
```kotlin
var test by Delegates.observable(0) { kProperty: KProperty<*>, oldValue: Int, newValue: Int ->
    println("旧值为：$oldValue, 新值为：$newValue")
}

test = 1
test = 2
test++
--test
```

输出：
```
旧值为：0, 新值为：1
旧值为：1, 新值为：2
旧值为：2, 新值为：3
旧值为：3, 新值为：2
```

这样监听变量属性修改就很方便了。

另一个委托属性 `Delegates.vetoable()`与`observable`相比，可以加一个判断，决定是否修改为新值。

它们继承了同一个类`ObservableProperty`
- `vetoable()`覆写了`beforeChange()`
- `observable()`覆写了`afterChange()`
- 两个方法的调用时机，一个在赋值前，一个在赋值后


