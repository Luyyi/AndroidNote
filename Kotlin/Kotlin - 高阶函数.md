# Kotlin - 高阶函数

### 高阶函数的定义：参数或者返回值是函数的函数

- 函数作为参数

    - Java不允许将方法作为参数传递，通常使用“接口”包装方法
    
    - 在kotlin中，可以传入函数类型的参数，如：
        - 无参数无返回值的函数类型：() -> Unit
        - 单Int参数返回String的函数类型：(Int) -> String
         
    ``` kotlin
        fun func(funParam: (Int) -> String): String {
            return funParam(1)
        }
    ```

- 函数作为返回值

 ``` kotlin
        fun func(): (Int) -> String {
            return {
                it.toString()
            }
        }
 ```
    
</br>

### 函数引用 ::Method

- 高阶函数中，函数可以作为参数或返回值，这里的函数是作为对象存在的

- 函数引用`::Method`：将一个函数变成一个对象

如：
``` kotlin
    fun a(p: Int) {
        println("这是函数a：$p")
    }
    
    fun b(param: (Int) -> Unit) {
        param(1)
        param.invoke(2)
    }
```

如果直接调用b(a)，会报错`Function invocation 'a(...)' expected`
</br>
使用函数引用 b(::a)，则函数b的param(1)与param.invoke(2)都能调用成功
</br></br>
param(1) 实际上调用的是param.invoke(1)</br>
但不能直接调用a.invoke(1)，因为a是一个函数，而b(::a)已经把a变成了函数引用对象。

</br>

### 匿名函数
- 函数引用是将已声明的函数变成对象赋值给变量或作为高阶函数的参数

- 匿名函数是直接把未声明的函数赋值给变量或作为高阶函数的参数

- 匿名函数省略了函数名称

``` kotlin
    fun b(param: (Int) -> Unit) {
        param(1)
        param.invoke(2)
    }
    
    // 匿名函数作为高阶函数的参数
    b(
        fun(p: Int) {
            println("匿名函数：$p")
        }
    )
    
    // 匿名函数赋值给变量
    val a = fun(p: Int) {
        println("匿名函数：$p")
    }
    
    b(a) // a已经是对象，不需要再使用函数引用::a
```

</br>

### Lambda表达式
- 匿名函数的基础上，再省略fun与参数

- Lambda的返回值是最后一行

``` kotlin
    fun b(param: (Int) -> Unit) {
        param(1)
        param.invoke(2)
    }
    
    // Lambda
    b(
        { p ->
            println("Lambda：$p")
        }
    )
    
    // 如果Lambda是函数的最后一个参数
    // 可以把Lambda挪到括号外匿名函数赋值给变量
    b() { p ->
            println("Lambda：$p")
     }
    
    // 如果Lambda是函数的唯一一个参数，可以省略括号
     b { p ->
            println("Lambda：$p")
     }
     
    // 如果Lambda只有一个参数，可以省略参数
    // Kotlin中默认用it表示
    b { 
        println("Lambda：$it")
    }
```

</br>

### Lambda表达式写法的“进化”

`匿名函数、Lambda本质上都是函数类型的对象，跟函数引用是一个东西`

##### 1. 正常定义的函数

``` kotlin
    fun a(p: Int) {
        println("正常定义的函数：$p")
    }
```

##### 2.匿名函数

``` kotlin
    val a: (Int) -> Unit = fun(p: Int) {
        println("匿名函数：$p")
    }
```

##### 3.Lambda

``` kotlin
    val a: (Int) -> Unit = { p: Int ->
        println("Lambda：$p")
    }
```

##### 4.省略参数的Lambda

``` kotlin
    val a: (Int) -> Unit = { 
        println("省略参数的Lambda：$it")
    }
```

##### 5.省略函数类型的Lambda

``` kotlin
    val a = { p: Int ->
        println("省略参数的Lambda：$p")
    }
```

##### 6.省略参数及函数类型的Lambda

``` kotlin
    val a = {
        println("省略参数及函数类型的Lambda：$it")
    }
```

</br>

### 常用高阶函数
##### 1. run()

- 仅仅执行block()方法，并返回执行结果

- 作为独立的代码块写一些与上下文无关的代码

- 用run函数包裹一段代码，并使用其返回值

```kotlin
@kotlin.internal.InlineOnly
public inline fun <T, R> run(block: T.() -> R): R {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }
    return block()
}
```

##### 2. T.run() 

-  T的扩展函数，内部可使用T对象的上下文this，调用T对象的公用方法/属性


```kotlin
@kotlin.internal.InlineOnly
public inline fun <T, R> T.run(block: T.() -> R): R {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }
    return block()
}
```

##### 3. with()

- 传入一个T类对象，调用并返回T对象的扩展方法block()

- 作用等同于T.run()

```kotlin
@kotlin.internal.InlineOnly
public inline fun <T, R> with(receiver: T, block: T.() -> R): R {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }
    return receiver.block()
}
```

##### 4. T.apply()

- 调用block()并返回T对象本身

- 在T.run()的基础上，跑完一串代码还可以继续取得并操作T对象

```kotlin
@kotlin.internal.InlineOnly
public inline fun <T> T.apply(block: T.() -> Unit): T {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }
    block()
    return this
}
```

##### 5. T.also()

- 通过block()调用T类对象，最后返回T对象本身

- 调用的是block(this)，所以在also()内部通过it来获取对象

```kotlin
@kotlin.internal.InlineOnly
@SinceKotlin("1.1")
public inline fun <T> T.also(block: (T) -> Unit): T {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }
    block(this)
    return this
}
```

##### 6. T.let()

- 通过block()调用T类对象，最后返回block()的结果

- 调用的是block(this)，所以在let()内部通过it来获取对象

```kotlin
@kotlin.internal.InlineOnly
public inline fun <T, R> T.let(block: (T) -> R): R {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }
    return block(this)
}
```

##### 7. T.takeIf()

- 传入条件，符合该条件的返回T对象本身，不符合则返回null

```kotlin
@kotlin.internal.InlineOnly
@SinceKotlin("1.1")
public inline fun <T> T.takeIf(predicate: (T) -> Boolean): T? {
    contract {
        callsInPlace(predicate, InvocationKind.EXACTLY_ONCE)
    }
    return if (predicate(this)) this else null
}
```

##### 8. T.takeUnless()

- 传入条件，不符合该条件的返回T对象本身，符合则返回null

```kotlin
@kotlin.internal.InlineOnly
@SinceKotlin("1.1")
public inline fun <T> T.takeUnless(predicate: (T) -> Boolean): T? {
    contract {
        callsInPlace(predicate, InvocationKind.EXACTLY_ONCE)
    }
    return if (!predicate(this)) this else null
}
```

##### 9. repeat()

- 重复执行传入的action()

```kotlin
@kotlin.internal.InlineOnly
public inline fun repeat(times: Int, action: (Int) -> Unit) {
    contract { callsInPlace(action) }

    for (index in 0 until times) {
        action(index)
    }
}
```

