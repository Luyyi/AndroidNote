# Kotlin - 密封类

### 密封类 Sealed Class

- 密封类用于表示受限制的类层次结构

- 某种意义上，是枚举的扩展

- 枚举常量仅作为单个实例存在，而密封类的一个子类可以有包含状态的多个实例

- 密封类是一个抽象类，不能用data等修饰符，也不用加open关键字，不能被实例化

</br>

#### 密封类用于表示受限制的类层次结构

- 用于表示类层次结构： 

    - 继承，它的子类可以是任意的类

- 受限制：

    - 密封类的子类，只能写在密封类的内部，或者写在父类的同一个文件里，但子类的子类可以写在别的地方

```kotlin
sealed class Expr
data class Const(val number: Double) : Expr()
data class Sum(val e1: Expr, val e2: Expr) : Expr()
object NotANumber : Expr()

fun eval(expr: Expr): Double = when (expr) {
    is Const -> expr.number
    is Sum -> eval(expr.e1) + eval(expr.e2)
    NotANumber -> Double.NaN
}
```

</br>

#### 密封类是枚举的扩展

- 枚举常量仅作为单个实例存在

    如：
    
    ``` kotlin
    enum class GenderEnum{
        Male,
        Female
    }
    ```
    
    验证：
    
    ```kotlin
    val male = GenderEnum.Male
    val male0 = GenderEnum.Male
    println("${male == male0}")  // 输出true
    ```
    
    </br>

- 密封类的一个子类则可以有包含状态的多个实例
    
    ```kotlin
    sealed class GenderSealed {
        class Male(val value: String) : GenderSealed()
        object Female : GenderSealed()
    }
    ```
    
    验证：
    
    ```kotlin
    val male1 = GenderSealed.Male("1")
    val male2 = GenderSealed.Male("2")
    println("${male1 == male2}")  // false
    
    val female1 = GenderSealed.Female
    val female2 = GenderSealed.Female
    println("${female1 == female2}")  // true
    ```
    
    </br>
    
### 密封类是怎么做限制的
  
  将密封类Expr反编译：
  
```java
public abstract class Expr {
   private Expr() {
   }

   // $FF: synthetic method
   public Expr(DefaultConstructorMarker $constructor_marker) {
      this();
   }
}
```

- sealed 修饰的密封类是个抽象类

- 构造函数被私有化了

- 公开的构造函数需要传参`DefaultConstructorMarker`

</br>

子类 data class Const：

```java
public final class Const extends Expr {
   private final double number;

   public final double getNumber() {
      return this.number;
   }

   public Const(double number) {
      super((DefaultConstructorMarker)null);
      this.number = number;
   }

   ...
}
```

子类 object NotANumber：

```java
public final class NotANumber extends Expr {
   @NotNull
   public static final NotANumber INSTANCE;

   private NotANumber() {
      super((DefaultConstructorMarker)null);
   }

   static {
      NotANumber var0 = new NotANumber();
      INSTANCE = var0;
   }
}
```

可以看到：

- 密封类的子类可以是任意类

- 密封类的子类反编译后都是`final class`

- 私有化的构造方法，都调用了`super((DefaultConstructorMarker)null)`

</br>

DefaultConstructorMarker：

```java
package kotlin.jvm.internal;

public final class DefaultConstructorMarker {
    private DefaultConstructorMarker() {}
}
```

- DefaultConstructorMarker是kotlin.jvm.internal里面的

- Kotlin限制internal不能被外部访问

- 所以密封类的子类，只能写在密封类的内部，或者写在父类的同一个文件里


</br>

### 枚举 vs 抽象类 vs 密封类

##### 枚举

- 枚举常量都是单例

- 枚举常量只能使用相同的类型作为参数

##### 抽象类

- 可以有不同的类型的子类继承它，但子类不受限

##### 密封类

- 结合了枚举的受限性和抽象类的灵活性

如用在网络请求状态上：

```kotlin

```