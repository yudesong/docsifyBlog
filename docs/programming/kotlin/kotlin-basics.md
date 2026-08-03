---
title: Kotlin篇一之Kotlin基本知识
url: https://juejin.cn/post/7076061105528766472?searchId=202607171431032AEF48318D9C3D79D7FB
publishedTime: 2022-03-17T21:36:20+08:00
---

## Kotlin基本知识

本文主要着手于Kotlin与Java不同的点进行总结。

Kotlin是静态类型语言，所有表达式的类型在编译期已经确定。Kotlin源代码存放在后缀名为.kt的文件中，经编译器编译成.class文件。

## 一、函数和变量

### 1）函数

函数以关键字fun开始，函数名称紧随其后，如

```kotlin
fun max(a: Int, b: Int): Int {
    return if (a > b) a else b
}
```

在Kotlin中，if是表达式，而不是语句。语句和表达式的区别在于，表达式有值，能作为另一个表达式的一部分使用；语句总是包围着它的代码块中的顶层元素，并且没有自己的值。

如果函数体写在花括号中，称这个函数有代码块体。如果它直接返回一个表达式，称有表达式体，如下，

```kotlin
fun max(a: Int, b: Int): Int = if (a > b) a else b
```

### 2）变量

不可变量：使用val关键字，不可变引用，引用对象的值可变，对应Java中的final 可变变量：使用var关键字，可变引用，对应Java中的非final

### 3）字符串模板

使用$符连接变量，如下

```
println("Hello, $name!)
println("Hello, ${name[0]}!)
```

## 二、类和属性

值对象：只有数据没有其他代码的类。如下

```kotlin
class Person(val name: String)
```

### 1）属性

属性：字段及其访问器的组合。 只读属性：使用val关键字，生成一个字段和一个简单的getter。 可变属性：使用var关键字，生成一个字段、一个getter和一个setter。

在Java中调用Kotlin属性时，会在字段名加get或set前缀，如name生成getName的getter方法及setName的setter方法。但如果属性名称以is开头，getter不会增加任何的前缀，而setter名称中的is会被替换成set。

### 2）自定义访问器

如下，自定义属性的getter，isSquare为只读属性，该属性值可变但不可设置，

```kotlin
class Rectangle(val height: Int, val width: Int) {
    val isSquare: Boolean
        get() {
            return height == width
        }
}
```

针对isSquare通过声明一个没有参数的函数和带自定义的getter属性在性能上无差别，但针对类的特征，声明为属性可读性更强。

## 三、表示和处理选择：枚举和“when”

### 1)枚举

Kotlin使用enum class来声明枚举，还可以声明带属性的枚举类，如下，

```
enum class Color(val r:Int, val g: Int, val b:Int) {
    RED(255, 0, 0),
    ORANGE(255, 165, 0),
    YELLOW(255, 255, 0),
    GREEN(0, 255, 0),
    BLUE(0, 0, 255),
    INDIGO(75, 0, 130),
    VIOLET(238, 130, 238;

    fun rgb() = (r * 256 + g) * 256 + b
}
```

可以通过Color.BLUE.rgb()进行调用。

### 2）when

when是一个有返回值的表达式，代码块中的最后一个表达式就是结果，如getMnemonic返回了when表达式，

```kotlin
fun getMnemonic(color: Color) =
    when (color) {
        Color.RED -> "Richard"
        Color.ORANGE -> "Of"
        Color.YELLOW -> "York"
        Color.GREEN -> "Gave"
        Color.BLUE -> "Battle"
        Color.INDIGO -> "In"
        Color.VIOLET -> "Vain"
    }
```

如果匹配成功，只有对应的分支会执行。将多个值合并到一个分支时，用逗号分隔。

when的参数可以使用任何对象。

when可以不带参数，分支条件就是任意的布尔表达式，如下

```
fun mixOptimized(c1: Color, C2: Color) =
    when {
        (c1 == RED && c2 == YELLOW) ||
        (c1 == YELLOW && c2 == RED) -> ORANGE

        (c1 == YELLOW && c2 == BLUE) ||
        (c1 == BLUE && c2 == YELLOW) -> GREEN

        (c1 == BLUE && c2 == VIOLET) ||
        (c1 == VIOLET && c2 == BLUE) -> INDIGO

        else -> throw Exception("Dirty Color")
    }
```

### 3)智能转换：合并类型检查和转换

智能转换：如果你检查过一个变量时某种类型，后面就不再需要转换它，可以把它当作你检查过的类型使用。且这个属性必须是一个val属性，且不能有自定义的访问器。

## 四、迭代事物

### 1）迭代数字：区间和数列

使用..运算符表示区间，如1..10，包含起始和结束值。

使用downTo step运算符表示带步长的递减数列，如100 downTo 1 step 2，包含起始和结束值。

使用until运算符表示不包含结束值的区间，如0 until 100，等价于1..99。

### 2）map

..语法不仅可以创建数字区间，还可以创建字符区间；使用in可以迭代区间或集合，如

```kotlin
val binaryReps = TreeMap<Char, String>()

for (c in 'A'..'F') {
    val binary = Integer.toBinaryString(c.toInt())
    binaryReps[c] = binary
}

for ((letter, binary) in binaryReps) {
    println("$letter = $binary")
}
```

binaryReps\[c\] = binary 等价于Java中binaryReps.put(c, binary)

在迭代集合的同时跟踪当前项的下标，不需要创建一个单独的变量来存储下标并手动增加它，如

```erlang
val list = arrayListOf("10", "11", "1001")
for ((index, element) in list.withIndex()) {
    println("&index: $element")
}
```

### 3)使用"in"检查集合和区间的成员

使用in运算符检查一个值是否在区间中，!n检查这个值是否不在区间中。

## 五、异常

Kotlin中的throw结构是一个表达式，能作为另一个表达式的一部分使用。

Kotlin没有受检异常。

## 六、函数的定义与调用

### 1）命名参数

调用函数时，显示地标明函数参数的名称，可读性更强。

### 2）默认参数

声明函数时，指定参数的默认值，可以避免创建重载的函数。可以省略排在末尾的参数。如果使用命名参数，可以省略中间的一些参数，也可以任意顺序只给定需要的参数。

参数的默认值是被编译到被调用的函数中，而不是调用的地方。

使用@JvmOverloads注解，指示编译器生成Java重载函数，从最后一个开始省略每个参数。

### 3）消除静态工具类：顶层函数和属性

项目中常常有以Util作为后缀名的静态工具类，Kotlin可以使用顶层函数来代替。

顶层函数：将函数放在代码文件顶层，不用从属于任何的类。顶层函数会被编译为类的静态函数。

要改变包含顶层函数的生成的类的名称，需要为这个文件添加@JvmName的注解，将其放到这个文件的开头，位于包名的前面，如

```
@file:JvmName("StringFunctions")
package strings
fun joinToString(...):String{...}
```

顶层属性：放在文件顶层的属性，会被存储到一个静态的字段中。

## 七、扩展函数和属性

扩展函数：定义在类的外面的类的成员函数。

把你要扩展的类或者接口的名称，放在即将添加的函数前面。这个类的名称被称为接收者类型；用来调用这个扩展函数的那个对象，叫做接收者对象。如下

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9218d35d50bf43b69ff82a9c46a524fe~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?) 在扩展函数中，可以直接访问被扩展的类的其他方法和属性，就好像是在这个类自己的方法中访问它们一样。和在类内部定义的方法不同的是，扩展函数不能访问私有的或者受保护的成员。

### 1）导入扩展函数

使用关键字as来修改导入的类或者函数的名称：

```kotlin
import strings.lastChar as last
val c = "Kotlin".last()
```

### 2）从Java中调用扩展函数

扩展函数是静态函数，它把调用对象作为了它的第一个参数。假设StringUtil.kt中有一个扩展函数lastChar，在Java中如下调用，

```
char c = StringUtil.lastChar("Java")
```

### 3)扩展函数不可重写

因为Kotlin把扩展函数当作静态函数对待，因此扩展函数不可被重写。

如果成员函数和扩展函数有相同的签名，成员函数优先。

### 4）扩展属性

和扩展函数一样，扩展属性也像接收者的一个普通的成员属性一样，必须定义getter函数，因为没有支持字段，因此没有默认的getter的实现，不可以初始化，因为没有地方存值，如

```kotlin
val String.lastChar: Char
    get() = get(length - 1)
```

从Java中访问扩展属性时，显示地调用它的getter函数，如StringUtilKt.getLastChar("Java")。

## 八、可变参数和中缀表达式

### 1）可变参数

使用vararg修饰符修饰可变参数，在参数前放置 \* 展开运算符。

### 2）键值对的处理：中缀调用和解构声明

使用mapOf函数来创建map

```
val map = mapOf(1 to "one", 7 to "seven", 53 to "fifty-three")
```

to是一种特殊的函数调用，叫中缀调用，函数名直接放在目标对象名称和参数之间。

中缀调用可以和只有一个参数的函数一起使用，要允许使用中缀符号调用函数，需要使用infix修饰符来标记它，如对to的声明，

```kotlin
infix fun Any.to(other: Any) = Pair(this, other)
```

val (number, name) = 1 to "one"表示结构，将1解构到nunmber，"one"解构到name。

## 九、字符串和正则表达式

Java中的Split方法不适用于一个点号，如"12.345-6.A".split(".")，"."会被当做正则表达式，表示任何字符的正则表达式，上述结果返回空数组。

Kotlin提供了一些名为split的具有不同参数的重载的扩展函数，用来承载正则表达式的值需要一个Regex类型。

三重引号的字符串不需要对任何字符进行转义，可以包含任何字符，包括换行符，可以简单地把包含换行符的文本嵌入到程序中。如Windows风格的路径"C:\\Users\\yole\\kotlin-book"可以写成"""C:\\Users\\yole\\kotlin-book"""。

向字符串内容添加前缀，标记边距的结尾，然后调用trimMargin来删除每行中的前缀和前面的空格。

## 十、局部函数

局部函数可以保持代码整洁及避免重复，如下 带重复的代码

```kotlin
class User(val id: Int, va; name: String, val address: String)

fun saveUser(user: User) {
    if (user.name.isEmpty()) {
    }
    if (user.address.isEmpty()) {
    }
    ......
}
```

提取逻辑到扩展函数

```kotlin
class User(val id: Int, va; name: String, val address: String)

fun User.validateBeforeSave() {
    fun validate(value: String, fieldName: String) {
        if (value.isEmpty()) {
        }
    }
    validate(name, "Name")
    validate(address, "Address")
}

fun saveUser(user: User) {
    user.validateBeforeSave()
    ......
}
```

## 十一、类继承结构

### 1）Kotlin中的接口

Kotlin使用interface关键字声明一个Kotlin接口，可以包含抽象方法的定义和非抽象方法的实现，不能包含任何状态。接口的方法可以有一个默认实现，只需要一个方法体。如

```kotlin
interface Clickable {
    fun click()
    fun showOff() = println("I'm clickable!")
}
```

如果实现这个接口，需要为click提供一个实现，可以重新定义showOff方法或者不重新定义。

Kotlin在类名后面使用冒号代替了Java中的extends和implements关键字，强制使用override关键字。

如果同样的继承成员有不止一个实现，必须提供一个显示的实现，如super.showOff()。

Kotlin把每个带默认方法的接口编译成一个普通接口和一个将方法体作为静态函数的类的结合体。

### 2）open、final和abstract修饰符：默认为final

对基类进行修改会导致子类不正确的行为，要么为继承做好设计并记录文档，要么禁止这么做，所以Kotlin中类和方法默认为final。

如果你重写了一个基类或者接口的成员，重写了的成员同样默认是open的。

智能转换只能在进行类型检查后没有改变过的变量上起作用，因为属性默认是final的，可以在大多数属性上使用智能转换。

抽象成员始终是open的，不需要显示地使用open修饰符。

抽象类中的非抽象函数并不是默认open的，但是可以标注为open。

### 3）可见性修饰符：默认为public

Kotlin没有将包用作可见性控制。

Kotlin引入新的修饰符internal，表示只在模块内部可见。一个模块就是一组一起编译的Kotlin文件，一个Intellij IDEA模块、一个Eclipse项目、一个Maven或Gradle项目或者一组使用Ant任务进行编译的文件。

包可见性控制的封装很容易被破坏，因为外部代码可以将类定义到与你代码相同的包中，但internal提供了对模块实现细节的真正封装。

Kotlin允许在顶层声明中使用private可见性，包括类和属性，只在声明它们的文件中可见。

protected成员只在类和它的子类可见，类的扩展函数不能访问它的private和protected成员。

Kotlin中一个外部类不能看到其内部（或者嵌套）类中的private成员。

### 4）内部类和嵌套类：默认是嵌套类

Kotlin的嵌套类不能访问外部类的实例。

Kotlin嵌套类默认静态，如果内部类需要持有外部类引用需要使用inner修饰符。

Kotlin中引用外部类实例与Java不同，需要啊使用this@Outer从Inner类去访问Outer。

### 5）密封类：定义受限的类继承结构

密封类：为父类添加一个sealed修饰符，所有的直接子类必须嵌套在父类中，密封类不能在类外部拥有子类。密封类默认是open类、抽象类、构造器私有。

密封类用法如下，作为密封类的表达式，

```kotlin
sealed class Expr {
    class Num(val value: Int) : Expr()
    class Sum(val left: Expr, val right: Expr) : Expr()
}

fun eval(e: Expr): Int =
    when (e) {
        is Expr.Num -> e.value
        is Expr.Sum -> eval(e.right) + eval(e.left)
    }
```

上述密封类将所有可能的类作为嵌套类列出，when表达式涵盖了所有可能的情况，所以不再需要else分支。

## 十二、声明带非默认构造方法或属性的类

主构造方法：在类体外部声明。

从构造方法：在类体内部声明。

### 1）初始化类

主构造方法表明了构造参数以及定义使用这些参数初始化的属性。

主构造方法不能包含初始化代码，则需要使用初始化语句块。

可以像函数参数一样为构造方法参数声明一个默认值，如

```kotlin
class User(val nickname: String, val isSubscribed: Boolean = true)
```

可以显示地为某些构造方法参数标明名称，如

```
val carol = User("Carol", isSubscribed = false)
```

如果所有的构造方法参数都有默认值，编译器会生成一个额外的不带参数的构造方法来使用所有的默认值。

必须显示地调用父类的构造方法，即使它没有任何参数。

### 2）构造方法：用不同的方式来初始化类

类里有很多构造器，该类被创建的路径有很多条，如果某一个路径初始化了a, b, c 三个属性，另一个路径初始化了b, c, d 三个属性，a 和 d 在这两条路径里面没有被完全覆盖，由于java里面属性能自动初始化为0或null，而kotlin不能自动初始化，所以所有属性没有被完全覆盖时会出现问题。

为解决上述问题，Kotlin初始化所有的副构造器都要调用主构造，所有的字段都需要初始化。如果类没有主构造方法，那么每个从构造方法必须初始化基类或者委托给另一个这样做了的构造方法。

### 3）实现在接口中声明的属性

接口可以包含抽象属性声明。

```kotlin
class PrivateUser(override val nickname: String) : User

class SubscribingUser(val email: String) : User {
    override val nickname: String
        get() = email.substringBefore('@')
}

class FacebookUser(val accountId: Int) : User {
    override val nickname = getFacebookName(accountId)
}
```

nickname在SubscribingUser中有一个自定义getter在每次访问时计算substringBefore，而在FacebookUser中有一个支持字段来存储在初始化时计算得到的数据。

除了抽象属性声明外，接口还可以包含具有getter和setter的属性，只要它们没有引用一个支持字段（支持字段需要在接口中存储状态，而这是不允许的）。如

```kotlin
interface User {
    val email: String
    val nickname: String
        get() = email.substringBefore('@')
}
```

该属性没有支持字段，结果值在每次访问时通过计算得到。

email属性必须在子类中重写，nickname可以被继承。

### 4）通过getter或setter访问支持字段

属性有两种：1）存储值的属性；2）具有自定义访问器在每次访问时计算值的属性。结合这两种实现的例子如

```javascript
class User(val name: String) {
    var address: String = "unspecified"
        set(value: String) {
            println("""
                Address was changed for $name:
                "$field" -> "$value".""".trimIndent())
            field = value
        }
}
```

在setter函数体中，使用了特殊的标识符field来访问支持字段的值。在getter中，只能读取值；而在setter中，既能读取它也能修改它。

访问属性的方式不依赖于它是否含有支持字段。如果显示地引用或者使用默认的访问器实现，编译器会为属性生成支持字段。如果提供了一个自定义的访问器实现并且没有使用field，支持字段将不会被生成。

### 5）修改访问器的可见性

访问器的可见性默认与属性的可见性相同，如果需要可以通过在get和set关键字前放置可见性修饰符的方式来修改它。如，

```kotlin
class LengthCounter {
    var counter: Int = 0
        private set

    fun addWord(word: String) {
        counter += word.length
    }
}
```

counter属性不能在类外部修改。

## 十三、数据类

Kotlin编译器能针对数据类和类委托将生成样板代码放在幕后，本文只讲数据类，类委托以后再讲。

“Any”是java.lang.Object的模拟：Kotlin中所有类的父类。

Kotlin中的is检查是Java中的instanceof的模拟，用来检查一个值是否为一个指定的类型。

数据类：在类前面添加data修饰符，自动生成通用方法equals、hashCode、toString。如下，

```kotlin
data class Client(val name: String, val postalCode: Int)
```

没有在主构造方法中声明的属性将不会加入到相等性检查和哈希值计算中去。

## 十四、“object”关键字：将声明一个类与创建一个实例结合起来

object定义一个类并同时创建一个实例，三个使用场景如下，

a）对象声明，定义单例的一种方式；

b）伴生对象，可以持有工厂方法和其他与这个类相关，调用时并不依赖类实例的方法。它们的成员可以通过类名来访问；

c）对象表达式，用来代替Java的匿名内部类。

### 1）对象声明：创建单例

单例类Singleton的创建如下图，为饿汉式单例，

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/39f02109281e48ac89babfec67cda83d~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?) 上述单例与下图Java单例形式等价，

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f015c73d9f82475291fb29314b580af1~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?) Kotlin中一个对象声明可以包含属性、方法、初始化语句块等的声明。唯一不允许的就是构造方法（包括主构造方法和从构造方法）。对象声明在定义时就立即创建了，不需要在代码的其他地方调用构造方法。

对象声明同样可以继承自类和接口。可以在类中声明对象。

### 2）伴生对象：工厂方法和静态成员

Kotlin中的类不能拥有静态成员，伴生对象（与包级别函数和属性一起）替代了Java静态方法和字段定义。在类中定义的对象使用companion关键字来标记，就获得了直接通过容器类名称来访问这个对象的方法和属性的能力，不再需要显示地指明对象的名称。如下，

```kotlin
class A {
    companion object {
        fun bar() {
        }
    }
}
```

调用如A.bar()。

伴生对象可以访问类中的所有private成员，包括private构造方法，是实现工厂模式的理想选择，如下用工厂方法创建新用户，确保每一个email都与一个唯一的User实例对应，

```kotlin
class User private constructor(val nickname: String) {
    companion object {
        fun newSubscribingUser(email: String) =
            User(email.substringBefore('@'))

        fun newFacebookUser(accountId: Int) =
            User(getFacebookName(accountId))
    }
}
```

伴生对象是一个声明在类中的普通对象，它可以有名字，实现一个接口或者有扩展函数或属性。如果你省略了伴生对象的名字，默认的名字将会分配为Companion。

伴生对象也可以实现接口。

在Java中调用时，在对应的成员上使用@JvmStatic注解，在Java中生成静态成员。@JvmField的作用是不为Kotlin属性在JVM上生成getter和setter。

类C做扩展函数时，如果C有一个伴生对象，且在C.Companion上定义了一个扩展函数func，可以通过C.func()来调用。

### 3）对象表达式：改变写法的匿名内部类

object用来声明匿名对象时，替代了Java中的匿名内部类的用法，并增加了如实现多个接口的能力和修改在创建对象的作用域中定义的变量的能力等。匿名内部类如下，

```kotlin
window.addMouseListener(
    object : MouseAdapter() {
        override fun mouseClicked(e: MouseEvent) {
        }

        override fun mouseEntered(e: MouseEvent) {
```

对象表达式声明了一个类并创建了该类的一个实例。

与Java匿名内部类只能扩展一个类或实现一个接口不同，Kotlin的匿名对象可以实现多个接口或者不实现接口。

与对象声明不同，匿名对象不是单例的。每次对象表达式被执行都会创建一个新的对象实例。

在对象表达式中的代码可以访问创建它的函数中的变量，访问没有被显示在final变量，还可以在对象表达式中修改变量的值，如

```kotlin
fun countClicks(window: Window) {
    var clickCount = 0

    window.addMouseListener(object : MouseAdapter() {
        override fun mouseClicked(e: MouseEvent) {
            clickCount++
        }
    })
}
```

对象表达式在需要在匿名对象中重写多个方法时是最有用的。只需要实现一个单方法的接口时用lambda或SAM。

