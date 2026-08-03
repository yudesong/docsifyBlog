---
title: Jetpack Compose 实践指南：从入门到进阶
url: https://juejin.cn/post/7431762980476141608?searchId=20260717145330D23120A8DCF28D7D9207
publishedTime: 2024-10-31T17:43:00+08:00
---

在Android开发领域，随着Jetpack Compose的推出，界面开发迎来了革命性的变化。Jetpack Compose是一个用于构建原生Android界面的现代UI框架，它使用更少的代码、强大的工具和直观的Kotlin API，极大地简化了Android UI的开发过程。本文将通过一系列实践案例，带您深入了解Jetpack Compose的应用与实践。

### 一、Jetpack Compose简介

Jetpack Compose是Google官方在2019年推出的UI框架，于2021年发布正式版。它彻底改变了传统的基于XML和View的UI开发方式，通过声明式的API和纯Kotlin代码来构建UI界面。Compose的核心是可组合函数（Composable functions），这些函数通过调用其他可组合函数来构建界面层次结构，而不需要编写XML布局文件。

### 二、开发环境准备

#### 1\. 创建新的Compose项目

在Android Studio中，创建一个新的Compose项目非常直接。按照以下步骤操作：

1. 打开Android Studio，选择“Start a new Android Studio project”。
2. 在“Choose your project”界面中，选择“Empty Compose Activity”。注意：这个选项可能在不同的Android Studio版本中名称略有不同，但通常会包含“Compose”关键字。
3. 填写您的项目名称、包名、保存位置等信息，然后点击“Finish”按钮。

#### 4\. 引入Compose依赖

对于使用Compose的Android项目，您需要在项目的 `build.gradle` 文件中添加相应的Compose依赖。但是，当您通过Android Studio的“New Project”向导选择“Empty Compose Activity”模板时，这些依赖通常会自动添加到您的项目中。不过，了解这些依赖仍然很重要，以便在需要时手动添加或更新它们。

以下是一个典型的Compose项目 `build.gradle` 文件中关于Compose依赖的部分示例（请注意，具体版本号可能会随时间而变化，因此请参考最新文档或Maven仓库中的信息）：

**在项目的 `build.gradle` （Module: app）文件中** ：

```groovy
dependencies {
    implementation "androidx.compose.ui:ui:$compose_version"
    implementation "androidx.compose.material:material:$compose_version"
    implementation "androidx.compose.ui:ui-tooling-preview:$compose_version"
    // 如果需要的话，还可以添加其他Compose相关的依赖

    // 其他必要的Android依赖...
}

// 在文件的顶部或适当位置定义compose_version变量（示例）
ext {
    compose_version = '1.3.0' // 请替换为最新的Compose版本号
}
```

**注意** ：

- `compose_version` 是一个在 `build.gradle` 文件中定义的变量，用于存储Compose库的版本号。这样做可以方便地在项目中统一管理和更新Compose库的版本。
- `ui` 模块是Compose的基础库，提供了构建UI所需的核心组件和API。
- `material` 模块提供了遵循Material Design规范的UI组件。
- `ui-tooling-preview` 模块包含了一些用于预览和调试Compose UI的工具。这个依赖项在发布应用时通常不需要包含在内。

请记得检查 [Google Maven仓库](https://link.juejin.cn/?target=https%3A%2F%2Fmaven.google.com%2Fweb%2Findex.html "https://maven.google.com/web/index.html") 或Jetpack Compose的 [官方文档](https://link.juejin.cn/?target=https%3A%2F%2Fdeveloper.android.com%2Fjetpack%2Fcompose "https://developer.android.com/jetpack/compose") 以获取最新的Compose版本号和依赖信息。

### 三、基础界面构建

#### 1\. Hello World

Compose中的“Hello World”示例非常简单。只需在 `setContent` 块中调用 `Text` 可组合函数即可：

```kotlin
import androidx.compose.material.Text
import androidx.compose.ui.tooling.preview.Preview
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import android.os.Bundle

class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            DefaultPreview()
        }
    }
}

@Preview(showBackground = true)
@Composable
fun DefaultPreview() {
    Text("Hello, World!")
}
```

![image.png](https://p3-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/bc7281116c8344728203e5de20c348bb~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5rSe56qd5oqA5pyv:q75.awebp?rk3s=f64ab15b&x-expires=1784756620&x-signature=CZ3A%2FTTe8J3XKMSN6UFiPdthrSU%3D)

#### 2.1 Text 组件

用于显示文本内容。可以设置字体、颜色、样式等。

```kotlin
Text(
    text = "Hello, Compose!",
    color = Color.Blue,
    fontSize = 20.sp,
    fontWeight = FontWeight.Bold
)
```

#### 2.2 Button 组件

Compose 提供了多种按钮组件，例如 `Button` 和 `OutlinedButton` 。可以设置点击事件和样式。

```kotlin
Button(onClick = { /* TODO */ }) {
    Text("Click Me")
}
```

#### 2.3 Image 组件

用于显示图片。Compose 支持通过 `painterResource` 加载本地资源图片，也可以结合 `Coil` 、 `Glide` 等库加载网络图片。

```kotlin
Image(
    painter = painterResource(id = R.drawable.image),
    contentDescription = "Sample Image"
)
```

#### 2.4 TextField 组件

用于输入文本的编辑框，类似于 EditText。

```kotlin
var text by remember { mutableStateOf("") }

TextField(
    value = text,
    onValueChange = { text = it },
    label = { Text("Enter something") }
)
```

#### 3.1 Column 和 Row

Column 和 Row 是最常用的布局组件，分别实现了垂直和水平布局。它们类似于 LinearLayout。

```kotlin
Column(
    modifier = Modifier
        .fillMaxSize()
        .padding(16.dp),
    verticalArrangement = Arrangement.spacedBy(8.dp)
) {
    Text("Item 1")
    Text("Item 2")
    Text("Item 3")
}
```

#### 3.2 Box

Box 类似于 FrameLayout，可以将多个组件叠加在一起。

```kotlin
Box(
    modifier = Modifier
        .fillMaxSize()
        .background(Color.Gray),
    contentAlignment = Alignment.Center
) {
    Text("Centered Text", color = Color.White)
}
```

#### 3.3 ConstraintLayout

ConstraintLayout 是一个高级布局组件，适合复杂布局需求。需要在 `build.gradle` 中添加依赖：

```gradle
implementation "androidx.constraintlayout:constraintlayout-compose:1.0.1"
```

使用示例：

```kotlin
ConstraintLayout(modifier = Modifier.fillMaxSize()) {
    val (button, text) = createRefs()

    Button(
        onClick = { /* TODO */ },
        modifier = Modifier.constrainAs(button) {
            top.linkTo(parent.top, margin = 16.dp)
        }
    ) {
        Text("Button")
    }

    Text("Hello ConstraintLayout", Modifier.constrainAs(text) {
        top.linkTo(button.bottom, margin = 8.dp)
        start.linkTo(button.start)
    })
}
```

### 四、状态管理与UI更新

在Jetpack Compose中，状态管理是实现响应式UI的关键。Compose通过自动检测和响应状态变化，确保UI始终与应用程序的内部状态保持一致。这种机制被称为“重组”（Recomposition）。

##### 1\. 可观察状态（Mutable State）

Compose使用 `mutableStateOf` 函数来创建可观察的状态。当你使用 `by remember { mutableStateOf(...) }` 创建状态时，Compose会跟踪这个状态，并在其值发生变化时自动触发UI的重组。

```kotlin
var count by remember { mutableStateOf(0) }
```

这里， `count` 是一个可观察的状态，它的任何变化都会触发包含它的Composable的重组。

##### 2\. 重组（Recomposition）

重组是Compose UI框架的核心机制之一。当Composable函数中的任何可观察状态发生变化时，Compose会重新调用该Composable函数并重新渲染相应的UI部分。重要的是，这种重组是增量式的，只影响需要更新的部分，而不是整个界面。

##### 3\. 侧效应（Side Effects）

有时，你可能需要在状态变化时执行一些副作用操作，比如发送网络请求、启动动画或更新其他非UI状态。Compose提供了 `LaunchedEffect` 和 `rememberCoroutineScope` 来帮助处理这些副作用。

- `LaunchedEffect` 允许你在状态变化时执行副作用操作，但它只在Composable首次执行时和特定键（key）的状态变化时运行。
- `rememberCoroutineScope` 用于在Composable中创建一个可记住的协程作用域，以便在协程中执行长时间运行的操作或监听。

##### 4\. 跨Composable共享状态

在大型应用中，状态可能需要在多个Composable之间共享。Compose提供了几种机制来实现这一点：

- **顶层ViewModel** ：使用ViewModel存储跨UI组件共享的状态。虽然ViewModel本身不是Compose特有的，但它可以与Compose无缝集成，特别是当使用 `viewLifecycleOwner` 和 `SavedStateHandle` 时。
- **共享状态持有者** ：创建一个Composable来封装和暴露状态，并通过参数将其传递给需要该状态的Composable。
- **全局状态管理库** ：使用如Jetpack Compose的 `accompanist-state-observation` 库或第三方状态管理库（如Jetpack Compose的 `rememberFlowWithLifecycle` 与状态流结合使用）来管理全局状态。

##### 5\. 避免不必要的重组

虽然Compose的重组机制非常高效，但在某些情况下，过多的重组可能会影响性能。为了避免不必要的重组，你可以：

- 使用 `remember` 来缓存不会改变的对象，避免在每次重组时都重新创建它们。
- 精细控制状态的变化通知，确保只在必要时触发状态变化。
- 使用 `distinctUntilChanged` 等操作符来过滤掉不必要的状态更新。

##### 6\. 实践案例

假设你正在开发一个计数器应用，其中包含增加和减少计数的按钮。你可以使用 `mutableStateOf` 来管理计数器的状态，并在按钮点击时更新该状态。Compose将自动检测到状态的变化，并重新渲染相关的UI部分。

```kotlin
@Composable
fun CounterScreen() {
    var count by remember { mutableStateOf(0) }

    Column {
        Text("Count: $count")
        Button(onClick = { count++ }) {
            Text("Increment")
        }
        Button(onClick = { count-- }) {
            Text("Decrement")
        }
    }
}
```

在这个例子中，每当用户点击“Increment”或“Decrement”按钮时， `count` 状态都会变化，Compose将自动触发重组，并更新显示计数的 `Text` 组件。

### 五、性能优化

Jetpack Compose通过其设计哲学和一系列内置机制，旨在提高UI渲染的性能。在Compose中，性能优化主要围绕减少不必要的计算和渲染操作展开。以下是一些关键的性能优化策略，结合代码示例进行说明。

##### 1. 减少不必要的重组

Compose的UI更新机制基于状态的变化自动触发重组。然而，过多的重组会导致性能问题。为了减少不必要的重组，我们可以采取以下措施：

- **使用 `remember` 缓存不变的数据** ： Compose中的 `remember` 函数可以帮助你缓存那些不需要在每次重组时都重新计算或创建的数据。
  ```kotlin
  val cachedData = remember { createExpensiveData() }
  ```
- **精确控制状态更新** ： 确保只有在真正需要时才更新状态，避免无谓的重组。
- **利用 `distinctUntilChanged` 减少状态变化** ： 如果你使用的是响应式流（如Flow或LiveData），并且状态更新非常频繁，但UI不需要每次更新都响应，可以使用 `distinctUntilChanged` 来减少状态变化通知。

##### 2. 优化布局和渲染

- **利用Intrinsic Measurement机制** ： Compose使用Intrinsic Measurement机制来自动计算组件的大小，这避免了传统Android开发中复杂的MeasureSpec计算和手动布局过程，从而提高了布局性能。
- **使用 `LazyColumn` 和 `LazyRow` 处理长列表** ： 对于需要展示大量项目的列表或网格，使用 `LazyColumn` 和 `LazyRow` 可以显著提高性能，因为它们只渲染可视区域内的项。
  ```kotlin
  LazyColumn {
      items(items) { item ->
          ListItem(item)
      }
  }
  ```

##### 3. 合理使用remember和LaunchedEffect

- **`remember` 用于缓存** ： 如上所述， `remember` 用于缓存那些计算成本高昂或不会频繁变化的数据。
- **`LaunchedEffect` 处理副作用** ： `LaunchedEffect` 用于执行副作用操作，如网络请求、动画启动等。与 `remember` 结合使用时，可以确保副作用操作只在必要时执行。
  ```kotlin
  LaunchedEffect(key1 = someState) {
      // 执行副作用操作，如网络请求
  }
  ```

##### 4. 避免深层嵌套布局

尽量避免过深的布局嵌套，因为这会增加Compose在重组时的工作量。尽量使用扁平化的布局结构，或者通过自定义Composable来封装复杂的布局逻辑。

##### 5. 监控和分析性能

- **使用Android Studio的性能分析工具** ： Android Studio提供了强大的性能分析工具，如Layout Inspector和Profiler，可以帮助你分析和优化Compose应用的性能。
- **关注CPU和GPU使用情况** ： 确保应用不会过度使用CPU或GPU资源，特别是在动画和复杂渲染操作中。

##### 6. 示例代码总结

下面是一个简单的示例，展示了如何在Compose中结合使用 `remember` 、 `LazyColumn` 和 `LaunchedEffect` 来优化性能：

```kotlin
@Composable
fun MyListScreen(items: List<Item>) {
    LazyColumn {
        items(items) { item ->
            val cachedData = remember { createExpensiveDataForItem(item) }
            ListItem(item = item, cachedData = cachedData)
        }
    }

    LaunchedEffect(Unit) {
        // 在组件首次渲染时执行一次性的初始化操作
        // 注意：这里的key是Unit，意味着这个副作用只会在组件首次渲染时执行一次
    }
}

@Composable
fun ListItem(item: Item, cachedData: Any) {
    // 使用cachedData渲染项内容...
}

// 假设这是一个计算成本高昂的操作
fun createExpensiveDataForItem(item: Item): Any {
    // ...
    return someResult
}
```

在这个示例中，我们使用了 `remember` 来缓存每个列表项的计算结果，避免了在每次滚动时都重新计算。同时，我们使用了 `LazyColumn` 来优化长列表的渲染性能。 `LaunchedEffect` 用于处理可能需要的初始化操作，但在这个例子中，我们只是为了演示而使用了它（实际上这里使用 `LaunchedEffect` 可能并不必要，除非你有一些初始化逻辑需要在组件首次渲染时执行）。

### 六、兼容性与迁移

对于现有的Android项目，可以逐步将Compose集成到现有View体系中。使用 `ComposeView` 可以在XML布局文件中定义一个Compose视图区域，并在该区域内使用Compose代码构建UI。此外，对于Compose暂时不支持的复杂View或库，可以通过 `AndroidView` 组件来嵌入原生View。

### 七、总结

Jetpack Compose为Android UI开发带来了革命性的变化。通过声明式的API和纯Kotlin代码，它极大地简化了UI开发过程，提高了开发效率和代码质量。随着Compose的不断发展和完善，它将成为未来Android UI开发的主流工具。希望本文能帮助您更好地理解和实践Jetpack Compose。

