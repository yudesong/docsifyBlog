---
title: Swift 语言基础与进阶
publishedTime: 2026-08-04T14:00:00+08:00
---

Swift 是 Apple 在 2014 年 WWDC 推出的现代化编程语言，用于 iOS / macOS / watchOS / tvOS / visionOS 开发。本文系统梳理 Swift 从基础语法到高级特性，覆盖面试常考与工程实战要点。

## 一、Swift 语言概述

### 1. 语言特性

| 特性 | 说明 |
|------|------|
| 类型安全 | 编译期检查类型，减少运行时错误 |
| 类型推断 | 声明时可省略类型，编译器自动推断 |
| 内存自动管理 | ARC（Automatic Reference Counting） |
| 多范式 | 面向对象 + 函数式 + 协议导向 |
| Optional | 显式处理 nil，避免空指针崩溃 |
| 值类型优先 | `struct` / `enum` 是值类型，`class` 是引用类型 |
| 泛型强大 | 协议带关联类型、条件 conforms |
| 错误处理 | `throws` / `try` / `catch` |

### 2. Swift vs Objective-C

| 维度 | Swift | Objective-C |
|------|-------|-------------|
| 语法 | 现代简洁 | C 超集 + Smalltalk 风格 |
| 类型系统 | 强类型 + 类型推断 | 动态类型 + runtime |
| 内存管理 | ARC + 值类型 | ARC + 引用类型 |
| 方法调用 | 直接调用（编译期绑定） | 消息传递（runtime） |
| 泛型 | 原生支持 | 无原生泛型 |
| 协议 | 可带默认实现 | 仅方法声明 |
| 可空性 | Optional 类型 | `nullable` 注解 |

### 3. 版本演进关键节点

| 版本 | 关键特性 |
|------|---------|
| Swift 1.0 | 基础语法、Optional |
| Swift 2.0 | 协议扩展、错误处理 |
| Swift 3.0 | API 设计指南、命名规范化 |
| Swift 4.0 | Codable、智能 KeyPath |
| Swift 5.0 | ABI 稳定、Result 类型 |
| Swift 5.1 | Opaque Types、Property Wrappers |
| Swift 5.5 | async/await、Actor 并发模型 |
| Swift 5.7 | `any` 关键字、正则表达式 |
| Swift 5.9 | Macros、`if` / `switch` 表达式 |

---

## 二、基础语法

### 1. 变量与常量

```swift
// 常量（不可变）
let pi: Double = 3.14159
let name = "Swift"           // 类型推断为 String

// 变量（可变）
var count: Int = 0
var items = [1, 2, 3]        // 推断为 [Int]
```

**值类型 vs 引用类型**：

```swift
// struct 是值类型
struct Point {
    var x: Double
    var y: Double
}

var p1 = Point(x: 1, y: 2)
var p2 = p1      // 拷贝
p2.x = 10
print(p1.x)      // 1（p1 不受影响）

// class 是引用类型
class Box {
    var value: Int
    init(value: Int) { self.value = value }
}

let b1 = Box(value: 1)
let b2 = b1      // 共享引用
b2.value = 10
print(b1.value)  // 10（b1 也变了）
```

### 2. 数据类型

```swift
// 整数
let i: Int = 42
let u: UInt = 42

// 浮点
let f: Float = 3.14
let d: Double = 3.141592653589793

// 布尔
let isActive: Bool = true

// 字符串
let str: String = "Hello"
let multi = """
多行
字符串
"""

// 字符
let char: Character = "A"
```

### 3. 字符串操作

```swift
var greeting = "Hello, Swift"

// 拼接
greeting += "!"
let name = "World"
let formatted = "Hello, \(name)"   // 字符串插值

// 长度
print(greeting.count)              // 字符数

// 子串
let index = greeting.firstIndex(of: ",")!
let part = greeting[..<index]      // "Hello"

// 遍历
for ch in greeting {
    print(ch)
}

// 查找与替换
let replaced = greeting.replacingOccurrences(of: "Swift", with: "World")
```

### 4. 集合类型

```swift
// 数组
var numbers: [Int] = [1, 2, 3]
numbers.append(4)
numbers.insert(0, at: 0)
print(numbers.first)        // Optional(0)
print(numbers[0...2])       // [0, 1, 2]

// 字典
var scores: [String: Int] = ["Alice": 90, "Bob": 85]
scores["Charlie"] = 78
print(scores["Alice"] ?? 0) // 90

// 集合
var primes: Set<Int> = [2, 3, 5, 7]
primes.insert(11)
primes.contains(3)           // true

// 区间
let range = 1...5            // 闭区间 [1, 5]
let halfRange = 1..<5        // 半开区间 [1, 5)
```

### 5. 控制流

```swift
// if-else
let score = 85
if score >= 90 {
    print("A")
} else if score >= 80 {
    print("B")
} else {
    print("C")
}

// guard：提前退出
func greet(_ name: String?) {
    guard let n = name, !n.isEmpty else {
        print("无名")
        return
    }
    print("Hello, \(n)")
}

// switch（强大：支持模式匹配）
let point = (2, 0)
switch point {
case (0, 0):
    print("原点")
case (_, 0):
    print("x 轴")
case (0, _):
    print("y 轴")
case let (x, y) where x == y:
    print("对角线")
default:
    print("其他")
}

// for-in
for i in 1...5 {
    print(i)
}
for item in [1, 2, 3] where item > 1 {
    print(item)
}

// while
var n = 5
while n > 0 {
    print(n)
    n -= 1
}
```

---

## 三、Optional 与解包

### 1. Optional 本质

`Optional` 是枚举：

```swift
enum Optional<Wrapped> {
    case none
    case some(Wrapped)
}
```

`String?` 等价于 `Optional<String>`，`nil` 等价于 `.none`。

### 2. 解包方式

```swift
let name: String? = "Swift"

// 1. 强制解包（危险，nil 时崩溃）
let n1: String = name!

// 2. if-let 绑定
if let n = name {
    print(n)
}

// 3. guard-let（推荐）
func process() {
    guard let n = name else { return }
    print(n)
}

// 4. 空合并运算符
let n2: String = name ?? "default"

// 5. 可选链
let length = name?.count        // Int?

// 6. 多重可选绑定
let a: String? = "a"
let b: Int? = 10
if let a = a, let b = b, b > 5 {
    print("\(a), \(b)")
}
```

### 3. 可选链

```swift
struct Person {
    var address: Address?
}
struct Address {
    var city: String
}

let p = Person(address: Address(city: "Shanghai"))
let city = p.address?.city       // Optional("Shanghai")
let count = p.address?.city.count // Optional(8)
```

链上任意一环为 nil，整个表达式返回 nil，不会崩溃。

---

## 四、函数与闭包

### 1. 函数

```swift
// 基础
func add(_ a: Int, _ b: Int) -> Int {
    return a + b
}
add(2, 3)  // 5（参数标签省略）

// 多返回值（元组）
func minMax(_ nums: [Int]) -> (min: Int, max: Int)? {
    guard !nums.isEmpty else { return nil }
    return (nums.min()!, nums.max()!)
}

// 默认参数
func greet(_ name: String, with greeting: String = "Hello") -> String {
    return "\(greeting), \(name)"
}
greet("Swift")              // "Hello, Swift"
greet("Swift", with: "Hi")  // "Hi, Swift"

// 可变参数
func sum(_ numbers: Int...) -> Int {
    return numbers.reduce(0, +)
}
sum(1, 2, 3, 4)  // 10

// 输入输出参数
func swap(_ a: inout Int, _ b: inout Int) {
    let t = a; a = b; b = t
}
var x = 1, y = 2
swap(&x, &y)
```

### 2. 闭包

```swift
// 闭包表达式
let add: (Int, Int) -> Int = { (a, b) in
    return a + b
}

// 类型推断
let add2: (Int, Int) -> Int = { $0 + $1 }

// 尾随闭包
[1, 2, 3].map { $0 * 2 }    // [2, 4, 6]

// 多尾随闭包（Swift 5.3+）
Button(action: {
    print("tapped")
}) {
    Text("Click")
}

// 逃逸闭包（@escaping）
func fetchData(completion: @escaping (Result<Data, Error>) -> Void) {
    DispatchQueue.global().async {
        completion(.success(Data()))
    }
}

// 自动闭包（@autoclosure）
func logIfTrue(_ condition: @autoclosure () -> Bool) {
    if condition() {
        print("true")
    }
}
logIfTrue(1 < 2)  // 传入表达式自动包成闭包
```

### 3. 捕获语义

```swift
var counter = 0
let closure = {
    counter += 1     // 捕获引用
}
closure()
print(counter)  // 1

// 闭包与 self 的循环引用（典型）
class NetworkLoader {
    var onComplete: ((Data) -> Void)?

    func load() {
        onComplete = { data in
            self.process(data)   // ❌ 强引用 self
        }
    }

    func process(_ data: Data) {}
}

// 解决：capture list
class NetworkLoader2 {
    var onComplete: ((Data) -> Void)?

    func load() {
        onComplete = { [weak self] data in
            self?.process(data)  // ✅ weak 引用
        }
    }

    func process(_ data: Data) {}
}
```

---

## 五、枚举与结构体

### 1. 枚举

```swift
// 基础枚举
enum Direction {
    case up, down, left, right
}

let dir: Direction = .up
switch dir {
case .up:    print("↑")
case .down:  print("↓")
default:     break
}

// 关联值
enum Result<Value> {
    case success(Value)
    case failure(Error)
}

let r: Result<Int> = .success(42)
switch r {
case .success(let value):
    print(value)
case .failure(let err):
    print(err)
}

// 原始值
enum HTTPStatus: Int {
    case ok = 200
    case notFound = 404
    case serverError = 500
}
print(HTTPStatus.ok.rawValue)  // 200

// 递归枚举
indirect enum Expression {
    case number(Int)
    case add(Expression, Expression)
    case multiply(Expression, Expression)
}

let expr: Expression = .add(.number(2), .multiply(.number(3), .number(4)))
```

### 2. 结构体

```swift
struct Rectangle {
    var width: Double
    var height: Double

    // 计算属性
    var area: Double {
        return width * height
    }

    // 方法
    func describe() -> String {
        return "Rectangle \(width)×\(height)"
    }

    // 修改方法（mutating）
    mutating func scale(by factor: Double) {
        width *= factor
        height *= factor
    }

    // 初始化器（struct 自动合成成员初始化器）
    // Rectangle(width:height:)
}

var rect = Rectangle(width: 10, height: 5)
print(rect.area)         // 50.0
rect.scale(by: 2)
print(rect.area)         // 200.0
```

### 3. 何时用 struct / class

| 场景 | 推荐 | 理由 |
|------|------|------|
| 数据模型 | struct | 值语义、线程安全 |
| UI 视图 | class | UIKit 继承体系 |
| 网络响应 | struct | 不可变、易测试 |
| 单例 / 管理器 | class | 引用共享 |
| 大数据拷贝 | class | 避免多次拷贝 |

---

## 六、类与继承

### 1. 类定义

```swift
class Animal {
    var name: String

    // 初始化器
    init(name: String) {
        self.name = name
    }

    // 反初始化器
    deinit {
        print("\(name) 被释放")
    }

    func speak() -> String {
        return "..."
    }
}

class Dog: Animal {
    var breed: String

    init(name: String, breed: String) {
        self.breed = breed
        super.init(name: name)
    }

    // 重写
    override func speak() -> String {
        return "汪汪"
    }
}

let d = Dog(name: "旺财", breed: "柴犬")
print(d.speak())  // 汪汪
```

### 2. 属性类型

```swift
class Temperature {
    // 存储属性
    var celsius: Double = 0

    // 计算属性
    var fahrenheit: Double {
        get { return celsius * 9 / 5 + 32 }
        set { celsius = (newValue - 32) * 5 / 9 }
    }

    // 类型属性
    static let boilingPoint = 100.0

    // 属性观察者
    var state: String = "" {
        willSet { print("即将变为 \(newValue)") }
        didSet { print("从 \(oldValue) 变为 \(state)") }
    }
}
```

### 3. ARC 内存管理

```swift
class Person {
    let name: String
    var apartment: Apartment?       // 强引用
    init(name: String) { self.name = name }
}

class Apartment {
    let unit: String
    weak var tenant: Person?        // 弱引用，避免循环引用
    init(unit: String) { self.unit = unit }
}

// 强引用循环示例（未解决前）
// person.apartment = apartment
// apartment.tenant = person   // 互相强引用 → 永不释放

// 解决方案：weak / unowned
// weak：可为 nil，对象释放后自动置 nil
// unowned：不可为 nil，假设对象一定存在（更危险）
```

---

## 七、协议与面向协议编程

### 1. 协议定义

```swift
protocol Drawable {
    var area: Double { get }
    func draw()
    // 可选方法（通过扩展提供默认实现）
    func describe() -> String
}

extension Drawable {
    func describe() -> String {
        return "Drawable, area = \(area)"
    }
}

struct Circle: Drawable {
    var radius: Double
    var area: Double { return .pi * radius * radius }

    func draw() {
        print("绘制圆")
    }
}

let c = Circle(radius: 5)
c.draw()             // 绘制圆
print(c.describe())  // Drawable, area = 78.5...
```

### 2. 协议扩展

```swift
extension Collection where Element: Numeric {
    func sum() -> Element {
        return reduce(0, +)
    }
}

let nums = [1, 2, 3, 4]
print(nums.sum())  // 10
```

### 3. 关联类型

```swift
protocol Container {
    associatedtype Item
    var count: Int { get }
    mutating func append(_ item: Item)
    subscript(i: Int) -> Item { get }
}

struct IntStack: Container {
    typealias Item = Int       // 可省略，自动推断
    private var items: [Int] = []

    var count: Int { return items.count }
    mutating func append(_ item: Int) { items.append(item) }
    subscript(i: Int) -> Int { return items[i] }
}
```

### 4. 面向协议编程（POP）

```swift
// 协议 + 扩展 = 默认实现 + 多态
protocol Greetable {
    var name: String { get }
}

extension Greetable {
    func greet() -> String {
        return "Hello, \(name)"
    }
}

struct User: Greetable {
    let name: String
}

let user = User(name: "Swift")
print(user.greet())  // Hello, Swift
```

通过协议提供默认实现，避免继承带来的耦合。

---

## 八、泛型

### 1. 泛型函数

```swift
func swapValues<T>(_ a: inout T, _ b: inout T) {
    let t = a; a = b; b = t
}

var x = 1, y = 2
swapValues(&x, &y)

var s1 = "A", s2 = "B"
swapValues(&s1, &s2)
```

### 2. 泛型类型

```swift
struct Stack<T> {
    private var items: [T] = []
    mutating func push(_ item: T) { items.append(item) }
    mutating func pop() -> T? { return items.popLast() }
}

var intStack = Stack<Int>()
intStack.push(1)
intStack.push(2)

var stringStack = Stack<String>()
stringStack.push("hello")
```

### 3. 类型约束

```swift
// T 必须 conforms Comparable
func findMax<T: Comparable>(_ values: [T]) -> T? {
    return values.max()
}

// T 必须 conforms Equatable，U 必须 conforms Numeric
func equals<T: Equatable, U: Numeric>(_ a: T, _ b: T, _ x: U, _ y: U) -> Bool {
    return a == b && x == y
}
```

### 4. 关联类型协议的泛型使用

```swift
// any Container（Swift 5.7+）
let containers: [any Container] = [
    IntStack(),
    // 其他 Container 实现
]

// 泛型 some 关键字（Swift 5.7+）
func makeContainer() -> some Container {
    return IntStack()
}
```

---

## 九、错误处理

### 1. 定义错误

```swift
enum NetworkError: Error {
    case badURL
    case timeout
    case serverError(Int)
    case parseError(message: String)
}
```

### 2. 抛出与捕获

```swift
func fetch(_ url: String) throws -> Data {
    guard url.hasPrefix("https") else {
        throw NetworkError.badURL
    }
    // ... 网络请求
    return Data()
}

do {
    let data = try fetch("https://example.com")
    print(data)
} catch NetworkError.badURL {
    print("URL 错误")
} catch NetworkError.timeout {
    print("超时")
} catch let NetworkError.serverError(code) {
    print("服务器错误: \(code)")
} catch {
    print("未知错误: \(error)")
}
```

### 3. try? / try!

```swift
// try?：失败返回 nil
let data1 = try? fetch("https://example.com")   // Data?

// try!：失败崩溃（仅确定不失败时使用）
let data2 = try! fetch("https://example.com")   // Data
```

### 4. Result 类型

```swift
func fetchResult(_ url: String) -> Result<Data, NetworkError> {
    guard url.hasPrefix("https") else {
        return .failure(.badURL)
    }
    return .success(Data())
}

switch fetchResult("http://bad") {
case .success(let data):
    print(data)
case .failure(let error):
    print(error)
}
```

---

## 十、并发编程

### 1. GCD（Grand Central Dispatch）

```swift
// 串行队列
let serialQueue = DispatchQueue(label: "com.example.serial")

// 并发队列
let concurrentQueue = DispatchQueue(label: "com.example.concurrent",
                                     attributes: .concurrent)

// 主队列
DispatchQueue.main.async {
    print("主线程")
}

// 异步执行
DispatchQueue.global().async {
    let data = loadData()
    DispatchQueue.main.async {
        updateUI(data)
    }
}

// 延迟执行
DispatchQueue.main.asyncAfter(deadline: .now() + 1) {
    print("1 秒后")
}

// 栅栏（barrier）
let queue = DispatchQueue(label: "rw", attributes: .concurrent)
queue.async { read() }
queue.async { read() }
queue.async(flags: .barrier) { write() }  // 等前面任务完成
queue.async { read() }
```

### 2. async / await（Swift 5.5+）

```swift
func fetchUser() async throws -> User {
    let (data, _) = try await URLSession.shared.data(from: url)
    return try JSONDecoder().decode(User.self, from: data)
}

// 调用
Task {
    do {
        let user = try await fetchUser()
        print(user)
    } catch {
        print(error)
    }
}

// 并发执行多个任务
async let user1 = fetchUser()
async let user2 = fetchUser()
let (u1, u2) = (try await user1, try await user2)
```

### 3. Actor（Swift 5.5+）

```swift
// 解决数据竞争的线程安全类型
actor Counter {
    private var value = 0

    func increment() {
        value += 1
    }

    func current() -> Int {
        return value
    }
}

let counter = Counter()
Task {
    await counter.increment()
    print(await counter.current())
}
```

`actor` 自动串行化访问，不需要手动加锁。

### 4. AsyncSequence

```swift
// 异步迭代
for try await event in eventStream {
    print(event)
}

// URL 加载行数据
let url = URL(string: "https://example.com/log")!
let (bytes, _) = try await url.bytes(for: .init(url: url))
for try await line in bytes.lines {
    print(line)
}
```

---

## 十一、属性包装器与 KeyPath

### 1. 属性包装器（Swift 5.1+）

```swift
@propertyWrapper
struct Clamped<T: Comparable> {
    var value: T
    let range: ClosedRange<T>

    init(wrappedValue: T, _ range: ClosedRange<T>) {
        self.range = range
        self.value = min(max(wrappedValue, range.lowerBound), range.upperBound)
    }

    var wrappedValue: T {
        get { value }
        set { value = min(max(newValue, range.lowerBound), range.upperBound) }
    }
}

struct Size {
    @Clamped(0...100) var width: Int = 50
    @Clamped(0...100) var height: Int = 50
}

var s = Size()
s.width = 200
print(s.width)  // 100（被限制）
```

### 2. KeyPath

```swift
struct User {
    let name: String
    let age: Int
}

let users = [
    User(name: "Alice", age: 30),
    User(name: "Bob", age: 25)
]

// KeyPath 排序
let sorted = users.sorted { $0[keyPath: \User.age] < $1[keyPath: \User.age] }

// map 到属性
let names = users.map(\.name)
print(names)  // ["Alice", "Bob"]
```

---

## 十二、高级特性

### 1. 反射（Mirror）

```swift
struct Person {
    let name: String
    let age: Int
}

let p = Person(name: "Swift", age: 10)
let mirror = Mirror(reflecting: p)
for child in mirror.children {
    print("\(child.label!): \(child.value)")
}
// name: Swift
// age: 10
```

### 2. 动态调用（@dynamicMemberLookup / @dynamicCallable）

```swift
@dynamicMemberLookup
struct JSON {
    var data: [String: Any]

    subscript(dynamicMember member: String) -> Any? {
        return data[member]
    }
}

let json = JSON(data: ["name": "Swift", "age": 10])
print(json.name)  // Swift
```

### 3. 结果构建器（@resultBuilder，Swift 5.4+）

```swift
@resultBuilder
struct StringBuilder {
    static func buildBlock(_ components: String...) -> String {
        return components.joined(separator: "\n")
    }
}

func buildText(@StringBuilder content: () -> String) -> String {
    return content()
}

let text = buildText {
    "Line 1"
    "Line 2"
    "Line 3"
}
print(text)
```

### 4. 宏（Macros，Swift 5.9+）

```swift
// 自定义宏（需 Macros 框架）
@freestanding(expression)
public macro stringify<T>(_ value: T) -> (T, String) = #externalMacro(module: "MacroKit", type: "StringifyMacro")

let (value, code) = #stringify(1 + 2)
// value = 3, code = "1 + 2"
```

### 5. 正则表达式（Swift 5.7+）

```swift
// 字面量
let regex = /(\d{4})-(\d{2})-(\d{2})/
if let match = "2026-08-04".firstMatch(of: regex) {
    print(match.1)  // 2026
    print(match.2)  // 08
}

// 泛型正则
let emailRegex = try Regex(#"^[a-z0-9]+@[a-z0-9]+\.[a-z]+$"#)
"abc@def.com".contains(emailRegex)  // true
```

---

## 十三、Swift 与 Objective-C 互操作

### 1. Swift 调用 Objective-C

在桥接头文件 `XXX-Bridging-Header.h` 中引入：

```objc
#import "LegacyObject.h"
```

Swift 代码直接使用：

```swift
let obj = LegacyObject()
obj.doSomething()
```

### 2. Objective-C 调用 Swift

Xcode 自动生成 `<ModuleName>-Swift.h`：

```objc
#import "MyApp-Swift.h"

SwiftClass *obj = [[SwiftClass alloc] init];
[obj swiftMethod];
```

Swift 类需标记 `@objc` 或继承 `NSObject`：

```swift
@objc class SwiftClass: NSObject {
    @objc func swiftMethod() {
        print("called from ObjC")
    }
}
```

### 3. 桥接类型

| Swift | Objective-C |
|-------|-------------|
| `String` | `NSString` |
| `Array` | `NSArray` |
| `Dictionary` | `NSDictionary` |
| `Int` / `Double` | `NSNumber` |
| `Data` | `NSData` |

---

## 十四、Swift 编译与性能

### 1. Swift 编译流程

```
.swift 源码
   ↓
[Parse] 生成 AST
   ↓
[Sema] 语义分析 + 类型检查
   ↓
[SILGen] 生成 Swift IR（SIL）
   ↓
[IRGen] 生成 LLVM IR
   ↓
[LLVM] 优化 + 生成机器码
   ↓
.o 目标文件 → 链接 → 可执行文件
```

### 2. 性能优化建议

| 优化点 | 说明 |
|--------|------|
| 优先 struct | 栈分配、无引用计数开销 |
| final class | 关闭动态派发 |
| private / fileprivate | 启用内联 |
| 避免过度协议 existential | 用泛型替代 `any Protocol` |
| Whole Module Optimization | 跨文件优化 |
| `-O / -Osize` | Release 配置 |

### 3. 类型推断的代价

```swift
// ❌ 复杂推断可能拖慢编译
let result = [1, 2, 3].map { $0 * 2 }.filter { $0 > 2 }.reduce(0, +)

// ✅ 显式类型
let result: Int = [1, 2, 3].map { $0 * 2 }.filter { $0 > 2 }.reduce(0, +)
```

---

## 十五、总结

| 主题 | 核心要点 |
|------|---------|
| 基础语法 | `let`/`var`、值类型/引用类型、Optional、字符串与集合 |
| 控制流 | `guard` 提前退出、`switch` 模式匹配、`for-in where` |
| 函数与闭包 | 默认参数、可变参数、`inout`、`@escaping`、`@autoclosure` |
| 枚举与结构体 | 关联值、原始值、递归枚举、`mutating`、计算属性 |
| 类与继承 | 初始化、`deinit`、属性观察者、ARC、weak/unowned |
| 协议 | 关联类型、协议扩展、POP、默认实现 |
| 泛型 | 类型约束、关联类型、`some` / `any` |
| 错误处理 | `throws` / `try` / `catch`、Result 类型 |
| 并发 | GCD、async/await、Actor、AsyncSequence |
| 高级特性 | 属性包装器、KeyPath、结果构建器、Macros、Regex |
| 互操作 | 桥接头、`@objc`、自动生成 `-Swift.h` |

Swift 的设计哲学是"易学难精"：基础语法一两天能上手，但真正写出 Swifty 的代码需要深入理解值类型、协议、泛型、并发模型。后续 SwiftUI 开发会大量运用这些特性。
