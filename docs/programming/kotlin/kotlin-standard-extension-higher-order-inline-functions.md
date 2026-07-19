---
title: Kotlin系列四：标准函数、扩展函数、高阶函数、内联函数
url: https://juejin.cn/post/7011780736109789215?searchId=202607171431032AEF48318D9C3D79D7FB
publishedTime: 2021-09-25T16:11:07+08:00
---

小知识，大挑战！本文正在参与“ [程序员必备小知识](https://juejin.cn/post/7008476801634680869 "https://juejin.cn/post/7008476801634680869") ”创作活动。

## 一 标准函数

Kotlin 的标准函数指的是 Standara.kt 文件中定义的函数，任何Kotlin 代码都可以自由调用所有的标准函数。

for example,定义一个自己的标准函数：

```kotlin
kotlin 代码解读复制代码fun <T> T.myApply(block: T.() -> Unit): T {
    block() // 在当前对象上执行传入的 Lambda 表达式
    return this // 返回对象本身
}

data class Person(var name: String = "", var age: Int = 0)

fun main() {
    val person = Person().myApply {
        name = "John"
        age = 30
    }

    println(person) // 输出：Person(name=John, age=30)
}
```

自定义的标准函数可以根据你的需求写在不同的地方，具体取决于你希望在哪些地方使用这个函数。

1. **顶层函数：** 如果你希望这个函数在整个项目中都能使用，可以将它定义为顶层函数。这样你可以在任何地方直接调用它。
2. **扩展函数：** 如果你希望这个函数能够像标准库中的函数一样直接在特定类型的实例上调用，可以将它定义为扩展函数。这样你就可以通过点符号在对象上调用这个函数。

下面是两种不同情况下的示例：

1. **定义为顶层函数：**

```kotlin
kotlin 代码解读复制代码fun <T> myApply(value: T, block: T.() -> Unit): T {
    value.block()
    return value
}

data class Person(var name: String = "", var age: Int = 0)

fun main() {
    val person = myApply(Person()) {
        name = "John"
        age = 30
    }

    println(person) // 输出：Person(name=John, age=30)
}
```

在这个示例中， `myApply` 函数是一个顶层函数，你可以在任何地方直接调用它。

1. **定义为扩展函数：**

```kotlin
kotlin 代码解读复制代码fun <T> T.myApply(block: T.() -> Unit): T {
    block()
    return this
}

data class Person(var name: String = "", var age: Int = 0)

fun main() {
    val person = Person().myApply {
        name = "John"
        age = 30
    }

    println(person) // 输出：Person(name=John, age=30)
}
```

在这个示例中， `myApply` 函数是一个扩展函数，你可以像使用标准库的扩展函数一样在对象上调用它。

无论你选择哪种方式，都可以根据需要将这个自定义的标准函数放在合适的位置，以便在项目中使用。

## 1.1 作用域函数

Kotlin 标准库中有一些函数，它们的唯一目的是在对象的上下文中执行代码块。当对一个对象调用这样的函数并提供一个 Lambda表达式时，它会形成一个临时作用域。在此作用域中，可以访问该对象而无需其名称。这些函数称为作用域函数。共有以下五种：let、run、with、apply 以及 also。

### 1.1.1 let

函数式 { 参数: 参数类型 -> 函数体 } ，它的参数就是传入本体，我们可以在函数体内对本体做任何事情。

```kotlin
kotlin 代码解读复制代码fun doStudy(study: Study?){
    study?.let {
        it.doHomework()
        it.readBooks()
    }
}
```

### 1.1.2 with

with 函数接收两个参数，第一个参数可以是一个任意类型的对象，第二个参数是Lambda表达式。with 函数会在Lambda 表达式中提供第一个参数对象的上下文，并使用Lambda 表达式的最后一行代码作为返回值返回。

语法：

```kotlin
kotlin 代码解读复制代码with(obj){
            // 这里是 ojb 的上下文
            "value" // with 函数的返回值
        }
```

例子，例如创建一个水果列表，将水果列表全部打印出来：

```kotlin
kotlin 代码解读复制代码fun printFruits(){
        val list = listOf("Apple", "Banana", "Orange", "Pear", "Grape")
        val buffer = StringBuilder()
        buffer.append("Start eating fruits. \n")
        for (fruit in list) {
            buffer.append(fruit).append("\n")
        }
        buffer.append("Ate all fruits.")
        val result = buffer.toString()
        println(result)
    }
```

观察上面的代码我们可以看到多次调用了builder 对象的方法，其实这个时候就可以使用with 函数来让代码变得更加简单：

```kotlin
kotlin 代码解读复制代码fun printFruits(){
    val list = listOf("Apple", "Banana", "Orange", "Pear", "Grape")
    val result = with(StringBuilder()) {
        append("Start eating fruits. \n")
        for (fruit in list) {
            append(fruit).append("\n")
        }
        append("Ate all fruits.")
        toString()
    }
    println(result)
}
```

在第一个参数传入了什么对象，那么Lambda 表达式内就会拥有这个对象的所有变量和函数，就相当于在对象内部调用函数，所以我们直接调用了StringBuilder 对象的append 函数。

### 1.1.3 run

run函数和with函数几乎一模一样，with 函数是内置函数形式调用with(obj){} ，run 函数是 obj.run{} ，一个是通过传入对象，一个是通过对象调用，作用相同，也是Lambda表达式内包含上下文环境，最后一句代码为返回值，run 函数只有一个参数就是Lambda 表达式。

```kotlin
kotlin 代码解读复制代码fun printFruits(){
    val list = listOf("Apple", "Banana", "Orange", "Pear", "Grape")
    val result = StringBuilder().run {
        append("Start eating fruits. \n")
        for (fruit in list) {
            append(fruit).append("\n")
        }
        append("Ate all fruits.")
        toString()
    }
    println(result)
}
```

### 1.1.4 apply

apply 函数和 run 函数基本相同，不同的地方在于，apply 会返回对象本身，Lambda 表达式内不存在返回值，也是在Lambda 表达式中提供对象的上下文，结构为 obj.apply{}。

```kotlin
kotlin 代码解读复制代码fun printFruits(){
    val list = listOf("Apple", "Banana", "Orange", "Pear", "Grape")
    val result = StringBuilder().apply {
        append("Start eating fruits. \n")
        for (fruit in list) {
            append(fruit).append("\n")
        }
        append("Ate all fruits.")
    }
    println(result.toString())
}
```

### 1.1.5 also

let 及 also 将上下文对象作为 lambda 表达式参数。如果没有指定参数名，对象可以用隐式默认名称 it 访问。

```kotlin
kotlin 代码解读复制代码fun getRandomInt(): Int {
    return Random.nextInt(100).also {
        writeToLog("getRandomInt() generated value $it")
    }
}

val i = getRandomInt()
```

### 1.1.6 takeIf 与 takeUnless

除了作用域函数外，标准库还包含函数 takeIf 及 takeUnless。这俩函数使你可以将对象状态检查嵌入到调用链中。

当以提供的谓词在对象上进行调用时，若该对象与谓词匹配，则 takeIf 返回此对象。否则返回 null。因此，takeIf 是单个对象的过滤函数。反之，takeUnless如果不匹配谓词，则返回对象，如果匹配则返回 null。该对象作为 lambda 表达式参数（it）来访问。

```kotlin
kotlin 代码解读复制代码val number = Random.nextInt(100)

val evenOrNull = number.takeIf { it % 2 == 0 }
val oddOrNull = number.takeUnless { it % 2 == 0 }
println("even: $evenOrNull, odd: $oddOrNull")
```

当在 takeIf 及 takeUnless 之后链式调用其他函数，不要忘记执行空检查或安全调用（?.），因为他们的返回值是可为空的。

```kotlin
kotlin 代码解读复制代码val str = "Hello"
val caps = str.takeIf { it.isNotEmpty() }?.toUpperCase()
//val caps = str.takeIf { it.isNotEmpty() }.toUpperCase() // 编译错误
println(caps)
```

takeIf 及 takeUnless 与作用域函数一起特别有用。 一个很好的例子是用 let 链接它们，以便在与给定谓词匹配的对象上运行代码块。 为此，请在对象上调用 takeIf，然后通过安全调用（?.）调用 let。对于与谓词不匹配的对象，takeIf 返回 null，并且不调用 let。

```kotlin
kotlin 代码解读复制代码fun displaySubstringPosition(input: String, sub: String) {
    input.indexOf(sub).takeIf { it >= 0 }?.let {
        println("The substring $sub is found in $input.")
        println("Its start position is $it.")
    }
}
displaySubstringPosition("010000011", "11")
displaySubstringPosition("010000011", "12")
```

## 1.2 小结

函数选择：为了帮助你选择合适的作用域函数，我们提供了它们之间的主要区别表。

| 函数  | 对象引用 | 返回值            | 是否是扩展函数             |
| ----- | -------- | ----------------- | -------------------------- |
| let   | it       | Lambda 表达式结果 | 是                         |
| run   | this     | Lambda 表达式结果 | 是                         |
| run   | \-       | Lambda 表达式结果 | 不是：调用无需上下文对象   |
| with  | this     | Lambda 表达式结果 | 不是：把上下文对象当做参数 |
| apply | this     | 上下文对象        | 是                         |
| also  | it       | 上下文对象        | 是                         |

以下是根据预期目的选择作用域函数的简短指南：

- 对一个非空（non-null）对象执行 lambda 表达式：let
- 将表达式作为变量引入为局部作用域中：let
- 对象配置：apply
- 对象配置并且计算结果：run
- 在需要表达式的地方运行语句：非扩展的 run
- 附加效果：also
- 一个对象的一组函数调用：with

apply 及 also 返回上下文对象。let、run 及 with 返回 lambda 表达式结果。

## 二 扩展函数

## 2.1 扩展函数基本使用

Kotlin 能够扩展一个类的新功能而无需继承该类或者使用像装饰者这样的设计模式。 这通过叫做 扩展 的特殊声明完成。 例如，你可以为一个你不能修改的、来自第三方库中的类编写一个新的函数。 这个新增的函数就像那个原始类本来就有的函数一样，可以用普通的方法调用。 这种机制称为扩展函数 。此外，也有扩展属性 ， 允许你为一个已经存在的类添加新的属性。

建议向哪个类中添加扩展函数，就定义一个同名的Kotlin文件，便于以后查找。当然扩展函数可以定义在任何一个现有类中，并不一定非要创建新文件。不过通常而言，最好定义成顶层方法，这样可以让扩展函数拥有全局的访问域。

语法结构：

```kotlin
kotlin 代码解读复制代码fun ClassName.methodName(param1: Int, param2: Int): Int {
    //相关逻辑
    return 0
}
```

声明一个扩展函数，我们需要用一个 接收者类型 也就是被扩展的类型来作为他的前缀。 下面代码为 MutableList 添加一个swap 函数：

```kotlin
kotlin 代码解读复制代码fun MutableList<Int>.swap(index1: Int, index2: Int) {
    val tmp = this[index1] // “this”对应该列表
    this[index1] = this[index2]
    this[index2] = tmp
}
```

现在，我们可以对任意 MutableList 调用该函数了：

```kotlin
kotlin 代码解读复制代码val list = mutableListOf(1, 2, 3)
list.swap(0, 2) // “swap()”内部的“this”会保存“list”的值
```

当然，这个函数对任何 MutableList 起作用，我们可以泛化它：

```kotlin
kotlin 代码解读复制代码fun <T> MutableList<T>.swap(index1: Int, index2: Int) {
    val tmp = this[index1] // “this”对应该列表
    this[index1] = this[index2]
    this[index2] = tmp
}
```

注意：

- 扩展函数在不修改某个类源码的情况下,动态地添加新的函数.className.
- 扩展函数不能访问原有类的私有属性
- 底层实际上是用写了个静态函数来实现的,将类的实例传入这个静态函数,然后搞一些操作

## 2.2 运算符重载

概念：同一运算符在不同的环境所表现的效果不同，如”+“在两个Int值之间表示两者的数值相加，在两个字符串之间表示，将字符串拼接，同时kotlin允许我们将任意两个类型的对象进行”+“运算，或者其他运算符操作。

语法结构：如下，其中operator 为运算符重载的关键字

```kotlin
kotlin 代码解读复制代码class A {
    operator fun plus(a: A): A {
        //相关逻辑
    }
}
```

**常见的语法糖表达式和实际调用函数对照表：**

| 表达式                               | 函数名         |
| ------------------------------------ | -------------- |
| a \* b                               | a.times(b)     |
| a / b                                | a.div(b)       |
| a % b                                | a.rem(b)       |
| a + b                                | a.plus(b)      |
| a - b                                | a.minus(b)     |
| a++                                  | a.inc()        |
| a--                                  | a.dec()        |
| !a                                   | a.not()        |
| a == b                               | a.equals(b)    |
| ”a > b“、”a < b“、”a >= b“、”a >= b“ | a.compareTo(b) |
| a..b                                 | a.rangeTo(b)   |
| a\[b\]                               | a.get(b)       |
| a\[b\] = c                           | a.set(b, c)    |
| a in b                               | b.contains(a)  |

### 2.3 最佳实践：扩展函数和运算符重载的合体

```kotlin
kotlin 代码解读复制代码operator fun ClassName.plus(param1: ClassName): ClassName {
    //相关逻辑
    return result
}
```

举例：

```kotlin
kotlin 代码解读复制代码fun main() {
     "hello " * 6
}
operator fun String.times(int:Int){
    val builder = StringBuilder()
    repeat(int){
        builder.append(this)
    }

    println(builder.toString())

}
```

## 三 Kotlin高阶函数

## 3.1 基本定义

函数类型定义：

```kotlin
kotlin 代码解读复制代码(String, Int) -> Unit
```

高阶函数定义：参数有函数类型或者返回值是函数类型的函数。

```kotlin
kotlin 代码解读复制代码fun a(funParam: (Int) -> String): String {
  return funParam(1)
}
```

要传一个函数类型的参数，或者把一个函数类型的对象赋值给变量有三种方法

```kotlin
kotlin 代码解读复制代码fun num1AndNum2(num1: Int, num2: Int, operation: (Int, Int) -> Int): Int {
    val result = operation(num1, num2)
    return result
}
```

## 3.2 三种用法

### 3.2.1 双冒号::method

在 Kotlin 里，一个函数名的左边加上双冒号，它就不表示这个函数本身了，而表示一个对象，或者说一个指向对象的引用，但，这个对象可不是函数本身，而是一个和这个函数具有相同功能的对象。下面的例子表示将plus()函数作为参数传递给num1AndNum2()函数。

```kotlin
kotlin 代码解读复制代码fun plus(num1: Int, num2: Int) : Int {
    return num1 + num2
}
val a = num1AndNum2(10, 8, ::plus)
```

### 3.2.2 匿名函数

```kotlin
kotlin 代码解读复制代码val a = num1AndNum2(10, 8, fun(num1: Int, num2: Int) = num1 + num2)
```

### 3.2.3 Lambda 表达式(常用)

```kotlin
kotlin 代码解读复制代码val a = num1AndNum2(10, 8) {
    n1, n2 -> n1 + n2
}
```

Kotlin高阶函数的实现原理是将Lambda表达式在底层转为如上第二种匿名类的实现方式。每调用一次Lambda都会创建一个新的匿名类实例，会造成额外的内存和性能开销。由此Kotlin提供了内联函数功能。

## 四 内联函数inline

Kotlin编译器会将内联函数中的代码在编译时自动替换到调用它的地方，这样就不存在运行时的开销了。一般会把高阶函数声明为内联函数，即在定义高阶函数时加上inline关键字声明，这是一种良好的编程习惯，绝大多数高阶函数是可以直接声明成内联函数的。

```kotlin
kotlin 代码解读复制代码inline fun num1AndNum2(num1: Int, num2: Int, operation: (Int, Int) -> Int): Int {
    val result = operation(num1, num2)
    return result
}
```

## 4.1 noinline

如果一个内联高阶函数中含有多个函数类型参数，其中有一个函数类型参数不想内联，可在该参数前加上noinline关键字。

```kotlin
kotlin 代码解读复制代码inline fun inlineFun(block1:() -> Unit, noinline block2: () -> Unit) {
}
```

为什么Kotlin还提供一个noline关键字来排除内联功能?

因为内联的函数类型参数在编译的时候会被代码替换，因此它没有真正的参数属性。非内联的函数类型参数可以自由地传递给其他函数，因为它就是一个真实的参数，而内联的参数类型只允许传递给另外一个内联函数，这是它最大的局限性。

内联函数和非内联函数的另一个区别是内联函数所引用的Lambda表达式中可以使用return关键字进行函数返回，而非内联函数只能局部返回。（注意，这个有点拗口，因为在Java8语言中，lambda表达式是可以明确使用return关键字的，而Kotlin语言中，lambda表达式却不能使用return关键字。而上面内联函数所引用的Lambda表达式中可以使用return关键字进行的函数返回是表示退出该lambda函数逻辑，局部返回，不再执行Lambda表示式的剩余内容。）

## 4.2 crossinline （禁止非局部返回）

如果在内联高阶函数中创建了另外的Lambda或者匿名类的实现，并且在这些实现中调用函数的参数，就会提示错误。如

```kotlin
kotlin 代码解读复制代码inline fun runRunnable(block: () -> Unit) {
    val runnable = Runnable {
        block() // 提示错误
    }
}
```

因为内联函数所引用的Lambda表达式允许使用return进行函数返回，而高阶函数的匿名类实现中不允许使用return造成了冲突。

为什么高阶函数的匿名类实现中不允许使用return？匿名类中调用的函数类型参数是不可能进行外层调用函数返回的（匿名类啊，你连名字都没有你怎么在外部调用？），最多只能对匿名类中的函数调用进行返回。所以高阶函数的匿名类实现中不允许使用return。

crossinline用于保证内联函数Lambda表达式中一定不会使用return进行函数返回。

```kotlin
kotlin 代码解读复制代码inline fun runRunnable(crossinline block: () -> Unit) {
    val runnable = Runnable {
        block()
    }
}
```

声明后runRunnable函数的Lambda中无法使用return进行函数返回了，仍然可以使用return@runRunnable进行局部返回

reified 关键字 reified 配合 inline可以将泛型实例化

```kotlin
kotlin 代码解读复制代码inline fun <reified T> startActivity(context: Context, block: Intent.() -> Unit) {
    val intent = Intent(context, T::class.java)
    intent.block()
    context.startActivity(intent)
}
```

