---
title: "Dart 语法特性速览"
tags:
  - dart
---

### 1. 变量声明与不可变性（`final`、`const`、`late`）
Dart 对变量不变性有严格要求，这是函数式编程思想的体现。

- **`final`**：**运行时常量**，只能被赋值一次（在运行时确定值）。`final now = DateTime.now();`
- **`const`**：**编译时常量**，值在编译期间就确定了，且会进行常量折叠（相同的值共用内存）。`const pi = 3.14;`
- **`late`**：**延迟初始化**。用于“声明时无法赋值，但使用时必不为空”的场景（比如 Flutter 中的 `_controller`），避免空检查的麻烦。注意：如果使用前没赋值会崩溃。

---

### 2. 空安全（Null Safety）相关运算符
这是 Dart 3.0 之后最核心的特性，用于优雅地处理可能为 `null` 的值。

| 运算符 / 关键字 | 作用 | 示例 |
| :--- | :--- | :--- |
| **`??`** | **如果左边为 null，则返回右边的值**（赋值给变量或作为返回值） | `String name = username ?? "Guest";` |
| **`??=`** | **如果变量为 null，则赋值为右边的值**（你刚问的，用于自身赋值） | `duration ??= Duration(seconds: 2);` |
| **`?.`** | **安全调用**：如果对象不为 null，才调用其方法或属性，否则返回 null | `user?.getAddress()?.city` |
| **`!`** | **非空断言**：明确告诉编译器“我确定这个值不是 null”，如果为 null 会抛出运行时异常 | `text = nullableString!;` |
| **`...?`** | **扩展运算符的空安全版**：展开 List 时，如果为 null 则忽略 | `items = [...?nullableList];` |

---

### 3. 集合（List/Set/Map）的简洁构建语法
在 Flutter 构建 UI 列表时，这些语法能让代码极度简洁，无需写多余的 `if` 或 `for` 循环。

- **`collection-if`**：在集合内部根据条件决定是否包含元素。
  ```dart
  // 常用于 Widget 子元素列表
  children: [
    Text('A'),
    if (isShow) Text('B'), // 条件为 false 时，B 自动不出现
  ]
  ```
- **`collection-for`**：在集合内部展开循环。
  ```dart
  children: [
    for (var item in items) Text(item.name),
  ]
  ```
- **扩展运算符 `...`**：将一个列表的所有元素展开插入另一个列表。配合 `...?` 处理 null 非常方便。

---

### 4. 运算符

#### 4.1  级联符号（`..` 和 `?..`）
这是 Dart 最独特的语法之一，让你**连续操作同一个对象**，无需反复写变量名，在配置对象时特别好用。

```dart
// 传统写法
var paint = Paint();
paint.color = Colors.blue;
paint.strokeWidth = 3.0;
paint.style = PaintingStyle.stroke;

// 级联写法
var paint = Paint()
  ..color = Colors.blue
  ..strokeWidth = 3.0
  ..style = PaintingStyle.stroke;
```
`?..` 是空安全版本，对象为 null 时不执行后续操作。


#### 4.2 级联操作符

级联操作符(Cascade notation)可以用来对一个对象进行多次操作，下面两段代码就是等价的：

级联操作符

```dart
var paint = Paint()
  ..color = Colors.black
  ..strokeCap = StrokeCap.round
  ..strokeWidth = 5.0;
```

非级联

```dart
var paint = Paint();
paint.color = Colors.black;
paint.strokeCap = StrokeCap.round;
paint.strokeWidth = 5.0;
```

#### 4.3 扩展操作符

扩展操作符(Spread operators)用于集合操作，其作用是将被修饰集合中的元素依次放入到新集合中。

```dart
var list = [1, 2, 3];
var list2 = [0, ...list];  // 0, 1, 2, 3
assert(list2.length == 4);
```

#### 4.4  ??=  是空合并赋值运算符

在 Dart 中，`??=` 是**空合并赋值运算符**（Null-aware assignment operator）。它的作用是：

- 如果左侧变量当前为 `null`，则将右侧表达式的值赋给它；
- 如果左侧变量不为 `null`，则保持原值不变。

在你给出的代码里：

```dart
duration ??= const Duration(seconds: 2);
```

等价于：

```dart
if (duration == null) {
  duration = const Duration(seconds: 2);
}
```

也就是说，当调用方没有显式传入 `duration` 参数（或传入 `null`）时，自动使用默认的 2 秒时长。

---

**补充：**  
`??=` 是 `??` 的赋值版本。单独的 `??` 用于表达式求值（如 `a ?? b` 表示若 `a` 非空则取 `a`，否则取 `b`），而 `??=` 专门用于变量赋值。两者都是 Dart 处理空值时的常用语法糖。

---

### 5. 箭头函数（`=>`）
当函数体**只有一行表达式**时，可以用 `=>` 简写，省去花括号和 `return`。

```dart
// 普通写法
int add(int a, int b) { return a + b; }
// 箭头写法
int add(int a, int b) => a + b;
```
注意：`=>` 后面必须紧跟一个表达式，不能写语句（如 `if` 或 `for`）。

---

### 6. 函数参数的灵活性（命名参数与位置参数）
Dart 是少数同时支持命名参数和位置参数的语言，极大提高了 API 可读性。

#### 6.1 关于方法

Dart 中方法参数的花样比较多，包含默认值、可选位置参数、命名参数等。

- **命名参数（推荐）**：用 `{}` 包裹，调用时需指定参数名，顺序随意。配合 `required` 强制必须传入。
  ```dart
  void showToast({required String message, Duration duration = const Duration(seconds: 2)}) {}
  // 调用：showToast(message: "Hi", duration: Duration(seconds: 3));
  ```
- **位置参数**：用 `[]` 包裹，表示可选，按顺序传入。
  ```dart
  void log(String msg, [String? tag]) {} // 调用：log("error", "network");
  ```

参数默认值要与后面两者连用，可选位置参数是方法参数中用 **[]** 包裹起来的部分，命名参数是 **{}** 包裹起来的部分。

```dart
void main() {
  print(func());
  print(func("Sohpia"));

  print(fun());
  print(fun(name: "Sohpia"));
}

String func([String name = "World"]) {
  return "func $name";
}

String fun({String name = "World"}) {
  return "fun $name";
}
```

命名参数默认情况应该是可选的，所以允许为空，或提供默认值；若不允许为空，且在调用时必须传入值，可以显式使用 `required` 进行标记。

```dart
static ToastRecord show({required String message, Duration? duration}) {
  duration ??= const Duration(seconds: 2);
  final record = ToastRecord(message: message, duration: duration);

  _singleton._toastWidget.showToast(record: record);

  return record;
}
```
  
#### 6.2  扩展方法、扩展变量、别名

Dart 也支持扩展方法、变量，使用 `extension` 关键词。

```dart
extension FileExtension on FileSystemEntity {
  String get name {
    return path.split(Platform.pathSeparator).last;
  }
}

typedef NotNullCallback<T extends Object> = void Function(T it);
extension KtOperator<T extends Object> on T? {
  void let(NotNullCallback<T> callback) {
    if (this != null) {
      callback.call(this!);
    }
  }
}
```

Dart 也支持未命名扩展，未命名扩展仅在本 library 中可见，这样可以规避  API 冲突问题。

```dart
extension on String {
  bool get isBlank => trim().isEmpty;
}
```

使用 `typedef` 关键词可以给一个类型起别名，在可以使一个复杂的方法声明变得更易读。  
  
---

### 7. 类相关

>Dart 中所有的 class 都是隐式接口，这也就意味着所有的 class 都可以被实现。

#### 7.1  mixin/with

Dart 语法中有一个独特的 mixin 类，被该关键词修饰的类中的所有变量、函数都可以被其他类通过 with 的方式进行 **混入**。但是通过这种形式，实际上达到了多继承的目的。

mixin 类中也可以有抽象方法。

=== "with 多个类"

```dart
class Maestro extends Person with Musical, Aggressive, Demented {
  Maestro(String maestroName) {
    name = maestroName;
    canConduct = true;
  }
}
```

=== "抽象方法"

```dart
mixin Musician {
  void playInstrument(String instrumentName); // Abstract method.
    
  void playPiano() {
    playInstrument('Piano');
  }
  void playFlute() {
    playInstrument('Flute');
  }
}
    
class Virtuoso with Musician { 
  void playInstrument(String instrumentName) { // Subclass must define.
    print('Plays the $instrumentName beautifully');
  }  
}
```

=== "on ：mixin 也可以有超类"

```dart hl_lines="7"
class Musician {
  musicianMethod() {
    print('Playing music!');
  }
}
    
// 这个 Mixin 只能混入到 Musician 及其子类中    
mixin MusicalPerformer on Musician {
  perfomerMethod() {
    print('Performing music!');
    super.musicianMethod();
  }
}
    
class SingerDancer extends Musician with MusicalPerformer { }
    main() {
      SingerDance().performerMethod();
    }
```

=== "mixin class ：既是 mixin 又是 class"

```dart
mixin class Musician {
  // ...
}

class Novice with Musician { // Use Musician as a mixin
  // ...
}

class Novice extends Musician { // Use Musician as a class
  // ...
}
```

#### 7.2  构造方法

除开常规的构造方法之外，有额外两种构造方法：

1. [命名构造方法](https://dart.cn/codelabs/dart-cheatsheet#named-constructors)
2. 通过 `factory` 关键词修饰的[工厂构造方法](https://dart.cn/codelabs/dart-cheatsheet#factory-constructors)

命名构造方法在实现时需要需要给所有未初始化的变量赋值，而工厂构造方法则需要构造出对应的对象。  

工厂构造方法能够返回其子类、null 对象，甚至抛出异常。

```dart
class Color {
  int red;
  int green;
  int blue;
  
  Color(this.red, this.green, this.blue);

  Color.black(): red = 0, green = 0, blue = 0;
  factory Color.black() => Color(0, 0, 0);
}
```

### 8  错误处理

Dart 的错误处理可以用这些关键词来进行描述：

常规的 try、catch、finally，以及独特的on、rethrow。

`on` 用来处理特定类型的异常，`rethrow` 则用来重新抛出异常。

```dart
try {
  breedMoreLlamas();
} on OutOfLlamasException {
  // A specific exception
  buyMoreLlamas();
} on Exception catch (e) {
  // Anything else that is an exception
  print('Unknown exception: $e');
} catch (e) {
  // No specified type, handles all
  print('Something really unknown: $e');
  rethrow;
}
```

`catch` 块支持一个参数`(e)`和两个参数`(e, s)`，e 表示 exception，s 表示 StackTrace。

```dart
try {
  // ···
} on Exception catch (e) {
  print('Exception details:\n $e');
} catch (e, s) {
  print('Exception details:\n $e');
  print('Stack trace:\n $s');
}
```

### 9. Dart 3.0+ 新秀：模式匹配（Patterns）与记录（Records）
这是近年最大的语法革新，让数据组合和逻辑分支更强大。

- **记录（Records）**：类似匿名对象，可以轻松返回多个值。
  ```dart
  (String name, int age) getPerson() => ("Alice", 18);
  var (name, age) = getPerson(); // 结构化解构赋值
  ```
- **Switch 表达式**：Switch 不再只是语句，可以直接返回数值，且支持复杂模式匹配。
  ```dart
  String status = switch (code) {
    200 => 'OK',
    404 => 'Not Found',
    _ => 'Unknown',
  };
  ```

---

### 小总结与建议
- **写 Flutter UI 时**，多用 **`collection-if`** 和 **`for`** 代替 `add` 或 `remove`，代码更清晰。
- **处理网络请求解析数据时**，多用 **`?`** 和 **`??`** 链式调用，避免手写 `if (data != null)`。
- **配置对象时**，多用 **`..`** 级联，比 Builder 模式更符合 Dart 风格。
