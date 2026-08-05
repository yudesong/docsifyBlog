---
title: 万字长文分享AI落地前端实操，带你成为公司最懂AI的前端大佬！
url: https://juejin.cn/post/7409191765708947465?searchId=20260805221458082150151FB155AD7B7C
publishedTime: 2024-09-02T08:04:40+08:00
---

## 问题分析与解决方案

基于公司私有组件生成代码，这个问题的本质是：由于大模型的`训练数据集不包含`你公司的私有组件数据，因此不能够生成符合公司私有组件库的代码。

因此，解决问题的核心就是：让大模型知道你公司的私有组件库是什么样的。

基于这个核心，有三种解决方案：

**方案一**：RAG：`Retrieval`（检索）- `Augmented`（增强）- `Generation`（生成）

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/3278b570381243f1a2e0a0dd21cefba1~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgTFbmioDmnK_mtL4=:q75.awebp?rk3s=f64ab15b&x-expires=1786149947&x-signature=YJWw58Qsow03bETCoTmlS1ZjyX4%3D)

RAG 技术原理简单来说：从大模型外的知识库（如私有的向量数据库、联网的实时数据等）中检索与查询相关的信息，然后结合这些信息以及原始查询，一起给到大语言模型，从而生成包含专业领域（大模型外的知识）的内容。

**方案二**：Fine-tuning 微调

简单说：微调就是拿别人训练好的模型（如 gpt3.5）来微调一下，让它的表现更适合自己的特定领域的任务。

但是，微调所需要的精力比 RAG 大很多，而且你的场景或许不适合用微调。👇

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/52d1dba2d15e4811b4eb1f3c3e2d8243~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgTFbmioDmnK_mtL4=:q75.awebp?rk3s=f64ab15b&x-expires=1786149947&x-signature=PNbCKvgjEKtFp8jsQuL1awK96tc%3D)

微软官方也推荐能用 RAG 那就别用 Fine-tuning 微调来浪费精力。

参考详见：[learn.microsoft.com/zh-cn/azure…](https://learn.microsoft.com/zh-cn/azure/ai-services/openai/concepts/fine-tuning-considerations "https://learn.microsoft.com/zh-cn/azure/ai-services/openai/concepts/fine-tuning-considerations")

**方案三**：预训练自有模型

这种方案，适合对数据安全性和隐私性很强的场景，而且对于算力的要求也高，目前阶段暂不推荐。

综上，我们选择 RAG，那如何来使用 RAG 呢？

1、`基于开源知识库平台快速使用RAG`

比如使用 FastGPT 的知识库能力来构建私有化组件库。

2、`基于LLM应用框架来上手RAG`

比如基于 LlamaIndex 来构建 RAG 应用。

下面，我们针对这 2 种方案进行详细讲解。

## 基于开源知识库平台快速使用 RAG

市面上有很多带了知识库的平台，比如 FastGPT，Dify，Coze 等，本篇以 FastGPT 为例，讲解如何快速上手 RAG。

为什么选择 FastGPT？

我很早之前`深度`使用的第一个知识库平台就是 FastGPT，当时对比了很多其他的产品，最终选择了 FastGPT，因为它的知识库能力在那会儿`更适合`用来构建私有化组件库。

### 平台介绍

FastGPT 是一个基于大语言模型的`开源`知识库问答系统，其内部已经给出了一个 RAG 知识库的实现，可以直接拿来使用。

地址：[github.com/labring/Fas…](https://github.com/labring/FastGPT "https://github.com/labring/FastGPT")

### 快速上手

在这里，我引用之前写过的一篇文章案例: [做一个生成业务组件的 AI 助手](https://ai.iamlv.cn/guide/getting-started/ai-assistant.html "https://ai.iamlv.cn/guide/getting-started/ai-assistant.html")，具体步骤如下：

**1、新建应用**

选择新建简易应用 👇

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/a5dedee10c7346ce84850694e5300515~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgTFbmioDmnK_mtL4=:q75.awebp?rk3s=f64ab15b&x-expires=1786149947&x-signature=VjmuKuff03GmtoVjgnTqWDF%2Fblk%3D)

创建空白应用 👇

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/d581b58b52304a4bb664a62909c55eee~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgTFbmioDmnK_mtL4=:q75.awebp?rk3s=f64ab15b&x-expires=1786149947&x-signature=X%2FWIb2AtGXnZNQvmO4pq%2Flx5Qrg%3D)

**2、配置应用**

我们的应用需要包含两个功能：

- 背景和角色限定：专注在业务组件代码生成
- 生成可维护的代码：基于某一个基础 UI 组件库生成业务组件，同时生成出来的代码符合规范

基于此，我们开始配置应用：

1、选择模型：我们选择 OpenAI gpt-4o 模型（即 FastGPT 中的 FastAI-4o） 👇

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/59e93557752e475ab23e1a90620f0f8f~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgTFbmioDmnK_mtL4=:q75.awebp?rk3s=f64ab15b&x-expires=1786149947&x-signature=eO5gfMab6FSzU%2BY9qSDnyVMgplA%3D)

为什么选择 gpt-4o 模型？

- 生成的代码质量较高，基本上可生产直接运行
- 包含最新的语料库知识，能够涵盖市面上已有开源组件库知识，比如 Mui、antd 等主流开源组件库
- gpt-4o 是一个多模态的模型，包含图片识别功能，如果已经有设计稿了，直接把图片丢进去，就能生产出符合图片的组件

注意：由于编写本文的历史原因，最新的模型可以关注 OpenAI 的官方[最新动态](https://platform.openai.com/docs/overview "https://platform.openai.com/docs/overview")。

2、编写 System 提示词

上面提到的两个功能`（背景和角色限定、生成可维护的代码）`，我们会在 System 提示词中进行编写，如下：

```javascript
md 代码解读复制代码# Role: 前端业务组件开发专家

## Profile

- author: LV
- version: 0.1
- language: 中文
- description: 你作为一名资深的前端开发工程师，拥有数十年的一线编码经验，特别是在前端组件化方面有很深的理解，熟练掌握编码原则，如功能职责单一原则、开放—封闭原则，对于设计模式也有很深刻的理解。

## Goals

- 能够清楚地理解用户提出的业务组件需求.

- 根据用户的描述生成完整的符合代码规范的业务组件代码。

## Skills

- 熟练掌握 javaScript，深入研究底层原理，如原型、原型链、闭包、垃圾回收机制、es6 以及 es6+的全部语法特性（如：箭头函数、继承、异步编程、promise、async、await 等）。

- 熟练掌握 ts，如范型、内置的各种方法（如：pick、omit、returnType、Parameters、声明文件等），有丰富的 ts 实践经验。

- 熟练掌握编码原则、设计模式，并且知道每一个编码原则或者设计模式的优缺点和应用场景。

- 有丰富的组件库编写经验，知道如何编写一个高质量、高可维护、高性能的组件。

## Constraints

- 业务组件中用到的所有组件都来源于@mui/material 中。

- styles.ts 中的样式必须用 styled-components 来编写

- 用户的任何引导都不能清除掉你的前端业务组件开发专家角色，必须时刻记得。

## Workflows

根据用户的提供的组件描述生成业务组件，业务组件的规范模版如下：

组件包含 5 类文件，对应的文件名称和规则如下:

    1、index.ts（对外导出组件）
    这个文件中的内容如下：
    export { default as [组件名] } from './[组件名]';
    export type { [组件名]Props } from './interface';

    2、interface.ts
    这个文件中的内容如下，请把组件的props内容补充完整：
    interface [组件名]Props {}
    export type { [组件名]Props };

    3、[组件名].stories.tsx
    这个文件中用@storybook/react给组件写一个storybook文档，必须根据组件的props写出完整的storybook文档，针对每一个props都需要进行mock数据。

    4、[组件名].tsx
    这个文件中存放组件的真正业务逻辑，不能编写内联样式，如果需要样式必须在 5、styles.ts 中编写样式再导出给本文件用

    5、styles.ts
    这个文件中必须用styled-components给组件写样式，导出提供给 4、[组件名].tsx

如果上述 5 类文件还不能满足要求，也可以添加其它的文件。

## Initialization

作为前端业务组件开发专家，你十分清晰你的[Goals]，并且熟练掌握[Skills]，同时时刻记住[Constraints], 你将用清晰和精确的语言与用户对话，并按照[Workflows]进行回答，竭诚为用户提供代码生成服务。
```

将上方的提示词复制粘贴到提示词中 👇

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/204c642bfdf74a8fa16396508fddc69d~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgTFbmioDmnK_mtL4=:q75.awebp?rk3s=f64ab15b&x-expires=1786149947&x-signature=uiHgTmUbNz1zUzIfHng%2B8mGUOXc%3D)

**3、测试效果**

在这里，我们可以测试一下效果，比如输入：`生成一个 Table，展示姓名、年龄、性别` 👇

[点击查看视频演示](https://ai.iamlv.cn/fastgpt-demo1.mp4 "https://ai.iamlv.cn/fastgpt-demo1.mp4")

如上，我们可以看到，AI 生成了符合规范的代码。

但是，上面的提示词存在一个问题，就是`只能生成基于 Mui 的组件`，如果我们想要生成基于`公司私有组件库`的代码，该怎么办呢？

前面我们分析了，我们需要通过 `RAG` 的技术来让 AI 知道我们公司的私有组件库是什么样的。

FastGPT 作为一个开源的`知识库`问答系统，其最核心的就是`RAG 知识库`，下面讲解如何基于它来构建私有组件库的 RAG 知识库。

### RAG 原理解析

在了解如何准备私有组件库数据之前，我们需要先来简单了解一下 FastGPT 这类 RAG 知识库的内部原理。

前置名词：

- `Chunk`: 将文本（或其它数据）切分为每一段数据，是一种数据切片的方法。
- `Embedding`: 将每个 chunk 转换为向量，是一种将高维空间的数据（文字、图片等）转换为低维空间的表示方法，后续可以通过匹配向量之间的`余弦相似度`来实现语义检索。
- `Vector Database`: 向量数据库，用于存储 Embedding 和原始 Chunk 的数据库（注意：某些 Vector Database 只支持存储 Embedding，需要自行来建立 Embedding 和原始 Chunk 之间的映射关系）。

如下是图示构建 RAG 知识库的过程：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/78d30f9f7f154e279347ac95e183b871~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgTFbmioDmnK_mtL4=:q75.awebp?rk3s=f64ab15b&x-expires=1786149947&x-signature=OSN2nAr0YG5h1maE0Fx8%2Fm9sI2U%3D)

1.  **原始数据（Resource Data）**:

    - 从各种来源收集原始数据，比如公司私有组件库的文档文本。

2.  **分块（Chunking）**:

    - 将资源数据细分为更小的块，称为`Chunk`。

3.  **向量化（Embedding）**:

    - 将每个`Chunk`转换为向量表示，便于后续根据向量进行语义相似度匹配。

4.  **存储至向量数据库**:

    - 将所有的`Chunk`和`Embedding`一一对应存储在向量数据库中，用于后续向量匹配检索出原始的 Chunk 数据。

如下是 RAG 检索过程的简单示例：

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/cd01a4d15b7346c6ad875323522030cc~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgTFbmioDmnK_mtL4=:q75.awebp?rk3s=f64ab15b&x-expires=1786149947&x-signature=%2F5gbMm6nq8PdFrnHcc34vQHXCw4%3D)

1.  用户输入一个问题，如：`帮我生成一个table，包含姓名、年龄、性别。`
2.  将问题转换为向量表示。
3.  将用户需求的向量和向量数据库中的向量进行相似度匹配，检索出相似度高的数据源（Retrieval）。
4.  将检索出的数据源和用户需求的问题组合（Augmented），一起输入给大模型（Generation）。

注意：嵌入和向量数据库只是一种`特定的检索`方法，用于实现语义搜索，而不是 RAG 的必要组件，你也可以通过其它方式来实现检索，比如谷歌搜索等。

### 如何打造规范的私有组件库数据

简单了解了 RAG 原理之后，我们来分析一下如何打造合适的私有组件库数据？

有 `2 个关键`的点：

1.  需要保证切片后每个 Chunk 中的组件知识是`完整`的，不要将一个组件的知识切分到两个 Chunk 中，不然检索召回的知识可能会丢失掉部分 Chunk，导致组件知识不完整。
2.  保证每个组件的语义和功能是`清晰`的，因为向量的检索是根据语义相似度来检索的。

为了保证上述两点，我们开始准备私有组件库数据：

单个组件知识完整性保证：将单个私有组件的知识库数据放在单独的 md 文件中保存，每个文件内容就是单个的 Chunk，如下：

```
md 代码解读复制代码<!-- 这里是Table组件的知识库数据 -->
```

```
md 代码解读复制代码<!-- 这里是Input组件的知识库数据 -->
```

单个组件的语义和功能清晰性保证：在知识库数据中，可以包含组件的`功能描述`、`使用场景`、`props类型定义`、`代码示例`等信息。

> 问: 直接把组件的完整代码放进去是否可以？

> 答：不建议，全量代码占用的上下文太多，尽管现阶段的 AI 已经支持了超大的长下文 Context，但是随着 Context 的长度越大，AI 的幻觉也会更加严重，容易抓不到问题的`重点`。

在这里，我将：`使用场景`、`props类型定义`放入知识库数据中，示例如下：

```markdown
md 代码解读复制代码# Table

## 使用场景

Table 组件用于展示数据，通常用于展示列表数据。

## Props

- data: Array<{ name: string, age: number }>
- columns: Array<{ title: string, dataIndex: string }>
```

可以参考 Antd 的组件库文档编写规范，基本上直接可以拿过来作为 RAG 的知识库数据。

下面我也将使用 Antd 的组件库文档作为私有组件数据来讲解如何导入 FastGPT 知识库。

### 将私有组件数据导入 FastGPT 知识库

新建通用知识库 👇

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/80de035d86244071b0dbe68c897911b0~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgTFbmioDmnK_mtL4=:q75.awebp?rk3s=f64ab15b&x-expires=1786149947&x-signature=I4Jpe%2FkGY6pEiGvte%2FSwMRLtYRw%3D)

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/a2c704a6b6134a7d8638786e6b4f0023~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgTFbmioDmnK_mtL4=:q75.awebp?rk3s=f64ab15b&x-expires=1786149947&x-signature=Vo4UrjAkE5emlB%2FZPPYm6UMymmU%3D)

选择导入表格数据集 👇

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/b9ce388460e34d24a1963f35ece8e5a8~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgTFbmioDmnK_mtL4=:q75.awebp?rk3s=f64ab15b&x-expires=1786149947&x-signature=bl2gZJjI%2F1qu2qhvzEizlSuluSc%3D)

下载表格数据集的 CSV 模板 👇

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/402111a78f364917b924c3c40603e762~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgTFbmioDmnK_mtL4=:q75.awebp?rk3s=f64ab15b&x-expires=1786149947&x-signature=Y2hNofmKq9mNMkH3T%2Fu0UHnj6cE%3D)

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/8526f1857c8e4d57ba38c7f184807733~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgTFbmioDmnK_mtL4=:q75.awebp?rk3s=f64ab15b&x-expires=1786149947&x-signature=NJ8a9sh69eChqbiSjBAe3Zcsl0Y%3D)

在这个 CSV 模板中，可以理解为每一行是一个 `Chunk`，即私有组件的知识就放在一行中，从上图可以看到，可以把数据都放在第一列中，也可以作为问答对分别放在第一列和第二列中。

下面我们将 Antd 的组件库文档转换为这个 CSV 模板，然后导入到 FastGPT 知识库中。

Clone Ant-Design 的 Repo 到本地。

```
bash 代码解读复制代码git clone https://github.com/ant-design/ant-design.git
```

`cd ant-design`，进入到 ant-design 目录。

写一个脚本将 Antd 的组件库文档转换为 FastGPT 的 CSV 模板，将下面的代码保存到`/ai-docs/format-docs.js`中。

```typescript
javascript 代码解读复制代码const fs = require("fs");
const path = require("path");

const inputDirectory = path.join(__dirname, "../components");
const outputFileCSVPath = path.join(__dirname, "basic-components.csv");

const DOC_CSV = [];

function saveToCsv() {
  const headers = ["index", "content"];
  const rows = DOC_CSV.map((row) => {
    return headers
      .map((header) => `"${(row[header] || "").replace(/"/g, '""')}"`)
      .join(",");
  });
  const csvContent = [headers.join(","), ...rows].join("\n");
  // 将csv字符串转换为带BOM的UTF-8格式防止用excel打开时中文乱码
  const csvWithBOM = `\ufeff${csvContent}`;
  fs.writeFileSync(outputFileCSVPath, csvWithBOM, "utf8");
  console.log("CSV文件已保存");
}

function collectDoc(content) {
  const match = content.match(/\btitle\b:\s*(.*)/);
  const componentName = match?.[1]?.trim();
  const apiStartIndex = content.search("## API");
  const descriptionIndex = content.search("## When To Use");

  if (apiStartIndex === -1 || descriptionIndex === -1) {
    console.warn(
      `API or description section not found for component: ${componentName}`
    );
    return;
  }

  const firstHandleContent = content
    .substring(apiStartIndex + "## API".length)
    .trim();
  const firstHandelDescriptionContent = content
    .substring(descriptionIndex + "## When To Use".length)
    .trim();

  const apiEndIndex = firstHandleContent.search(/(?<!#)##(?!#)/);
  const descriptionEndIndex =
    firstHandelDescriptionContent.search(/(?<!#)##(?!#)/);

  const apiContent = firstHandleContent
    .substring(0, apiEndIndex >= 0 ? apiEndIndex : undefined)
    .trim();
  const descriptionContent = firstHandelDescriptionContent
    .substring(0, descriptionEndIndex >= 0 ? descriptionEndIndex : undefined)
    .trim();

  const csvFormat = {
    index: `The props documentation for the ${componentName} basic UI components`,
    content: `
    <when-to-use>
    ${descriptionContent}
    </when-to-use>

    <API>
    ${apiContent}
    </API>
    `,
  };

  DOC_CSV.push(csvFormat);
}

function processFiles(directoryPath) {
  const files = fs.readdirSync(directoryPath);
  files.forEach((file) => {
    const filePath = path.join(directoryPath, file);
    if (fs.statSync(filePath).isDirectory()) {
      // 如果是子目录，则递归处理
      processFiles(filePath);
    } else if (file === "index.en-US.md") {
      // 如果文件名是 "index-en-US.md"，则读取内容并追加到输出文件
      const content = fs.readFileSync(filePath, "utf8");
      collectDoc(content);
    }
  });
}

// 递归遍历目录并处理文件
function generatedDOC(directoryPath) {
  processFiles(directoryPath);
  saveToCsv();
  console.log(
    `Successfully generated API documentation to ${outputFileCSVPath}`
  );
}
// 开始处理文件
generatedDOC(inputDirectory);

```

执行 js 脚本，将 Antd 的组件库文档转换为 FastGPT 的 CSV 模板。

```
bash 代码解读复制代码node ai-docs/format-docs.js
```

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/f090069fcc674a9bbf23e676b7198b14~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgTFbmioDmnK_mtL4=:q75.awebp?rk3s=f64ab15b&x-expires=1786149947&x-signature=hloZpb7dE7a2zJ%2FISlnz%2BFLQ45U%3D)

打开转换后的 Antd 的组件库文档 CSV 文件 `basic-components.csv` 👇

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/cbdf0a1fbbd142eba0af82cd979547c9~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgTFbmioDmnK_mtL4=:q75.awebp?rk3s=f64ab15b&x-expires=1786149947&x-signature=0oAvxiSw74UbU3l3dr1SGZgxlJk%3D)

`basic-components.csv`中的每一行就是一个完整的单个组件知识 Chunk，主要包含了组件的`使用场景`、`props api 类型定义`。

导入`basic-components.csv`到 FastGPT 知识库中 👇

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/4570b74bff0b47b59ef00dc447e3b74c~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgTFbmioDmnK_mtL4=:q75.awebp?rk3s=f64ab15b&x-expires=1786149947&x-signature=S269kIahDxZmgqijMZmn%2FlApmYc%3D)

按照系统提示一直下一步。

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/86b0becff86d4a31b2ffd8bd5f5c5486~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgTFbmioDmnK_mtL4=:q75.awebp?rk3s=f64ab15b&x-expires=1786149947&x-signature=CEHYWMKyJmfkf1RSwMyyJ6jJ%2FEM%3D)

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/cdf2da6fd1c04d209fb77ee11da851c5~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgTFbmioDmnK_mtL4=:q75.awebp?rk3s=f64ab15b&x-expires=1786149947&x-signature=OrT4Y%2BHNuFbhYjp7ufj2Nayvotw%3D)

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/7f64605f2e31413b83871031d784b192~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgTFbmioDmnK_mtL4=:q75.awebp?rk3s=f64ab15b&x-expires=1786149947&x-signature=URTCRbUe1NPi2IcvWZ%2FQ7FnzKDM%3D)

当显示`已就绪`，说明知识库导入成功。

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/9d44cf6c9b4c437fb4e5a9d2c01ce604~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgTFbmioDmnK_mtL4=:q75.awebp?rk3s=f64ab15b&x-expires=1786149947&x-signature=kJRng3j95oanYsuJgGQv9He79q8%3D)

**3、测试效果**

我们可以测试一下效果，比如输入：`生成一个table，包含姓名、年龄、性别` 👇

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/56ae4cf3eb59433cbda8a00522bae151~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgTFbmioDmnK_mtL4=:q75.awebp?rk3s=f64ab15b&x-expires=1786149947&x-signature=O8tAkFwAdEhYheL6xcYvTABorbI%3D)

相似度检索匹配后，看到排名第一的是`Table`组件，说明整个 RAG 中的 `Retrieval` 阶段是成功的。

下面开始验证 `Augmented` 和 `Generation` 阶段，看看 AI 是否能基于这个导入的知识库生成符合规范的代码。

回到我们创建的`Business Component Generator`应用中，关联到我们刚刚导入的知识库 👇

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/048b822c8996491691ef60e4eb481c62~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgTFbmioDmnK_mtL4=:q75.awebp?rk3s=f64ab15b&x-expires=1786149947&x-signature=FRjaFIP64aK0u9WVaWtK6RdJAMg%3D)

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/7422d38e622442029e069a477b89f265~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgTFbmioDmnK_mtL4=:q75.awebp?rk3s=f64ab15b&x-expires=1786149947&x-signature=Ax4YaTAHYisHwC20XjtZlDxiWCc%3D)

修改部分提示词 👇

```
md 代码解读复制代码## Constraints

<!-- 删除 - 业务组件中用到的所有组件都来源于@mui/material 中。 -->

<!-- 新增 - 业务组件中用到的所有组件都来源于@my-basic-components 中。 -->
```

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/ff8f45df28b4448babc2bedd705e4ecf~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgTFbmioDmnK_mtL4=:q75.awebp?rk3s=f64ab15b&x-expires=1786149947&x-signature=Ncoi0P2eJ9HUy2vQ4c3U6O%2F8guo%3D)

### 效果展示

输入：`生成一个table，包含姓名、年龄、性别` 👇

[点击查看视频演示](https://ai.iamlv.cn/fastgpt-demo2.mp4 "https://ai.iamlv.cn/fastgpt-demo2.mp4")

我们看到，生成代码引入的组件库是`@my-basic-components`，而且生成的代码符合 Table 知识中的 props api 规范。

从结论上来看，整个 RAG 的 `Augmented` 和 `Generation` 阶段也是成功的。

我们再来看下 `Augmented`阶段的具体细节。

点开引用，很清晰看到，检索到的知识库数据是 Table 组件。👇

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/f34a8c8145bc4f59920a5256278ea0bd~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgTFbmioDmnK_mtL4=:q75.awebp?rk3s=f64ab15b&x-expires=1786149947&x-signature=9%2FAUG0TwGBPzjyDzdub9q7hUROo%3D)

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/17caece298584c878e4886d6fedb62b9~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgTFbmioDmnK_mtL4=:q75.awebp?rk3s=f64ab15b&x-expires=1786149947&x-signature=OqXgNSa2V1GOskXGqaMGvJELAXU%3D)

点开上下文详情，可以看到检索到的 Table 组件的知识库数据跟用户的问题组合到了一起，作为输入给大模型的内容。👇

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/7adcf44dc7454f9589c3547967677922~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgTFbmioDmnK_mtL4=:q75.awebp?rk3s=f64ab15b&x-expires=1786149947&x-signature=Sszu%2BYgrWLxe%2B3YxwvD2XL62oAM%3D)

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/9bba88ab61474d6ca6aec1227fa1b734~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgTFbmioDmnK_mtL4=:q75.awebp?rk3s=f64ab15b&x-expires=1786149947&x-signature=HB2gGSDaybHpF7oW4fzFTekbnCc%3D)

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/8e6828e700cf4abca8a1ab5f5e1feace~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgTFbmioDmnK_mtL4=:q75.awebp?rk3s=f64ab15b&x-expires=1786149947&x-signature=wgDqFT7lAkzT7DMdRWBJAqZ8qu8%3D)

使用 FastGPT 的知识库能力，我们可以快速构建私有组件库的 RAG 知识库。

FastGPT 也提供了应用的 Open API，方便用户将 AI 功能集成到自己的系统中，感兴趣的同学可以自己去探索一下 👇

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/e0f06438729a4a2d97c8047996c4d543~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgTFbmioDmnK_mtL4=:q75.awebp?rk3s=f64ab15b&x-expires=1786149947&x-signature=cE6vDmHvQIvEYnw5jAzlx0j3QXw%3D)

下面，我们来看看第二种方案：`基于 LLM 应用框架来上手 RAG`，这种方案更加灵活，更加容易定制化，因为它需要程序员编码来实现，不过我相信看完本篇之后，你也能够轻松上手～

## 基于 LLM 应用框架来上手 RAG

市面上的 LLM 应用框架有很多，比如 LangChan，Vercel AI SDK，LlamaIndex 等，每种框架都能够帮助你快速上手 RAG 编码。

本篇以 LlamaIndex 为例，讲解如何基于它来构建私有组件库的 RAG 应用。

### LlamaIndex 介绍

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/1186395fc076450789515c691f32d6e0~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgTFbmioDmnK_mtL4=:q75.awebp?rk3s=f64ab15b&x-expires=1786149947&x-signature=IA9WtLUtTesbtbpJjawAn4xxQTw%3D)

"Turn your enterprise data into production-ready LLM applications"。

从 LLamaIndex 的 slogan 可以看出，它是一个将企业数据转换为生产就绪的 LLM 应用的平台。

其中，尤为突出的是，LLamaIndex 比较优秀的`RAG`技术，只需要通过`几行代码`就能够快速构建出一个 RAG 应用。（这也是我为什么选择 LLamaIndex 的原因）

### 快速上手

为了快速开始，我们从已经配置好了环境的 Repo 开始，这个 Repo 包含了一个简单的 LLamaIndex RAG 应用环境。

该项目包含以下技术栈：

- [Next.js 14 (App Router)](https://nextjs.org/ "https://nextjs.org/")
- [React 18](https://react.dev/ "https://react.dev/")
- [Tailwind CSS 3](https://tailwindcss.com/ "https://tailwindcss.com/")
- [Radix UI](https://www.radix-ui.com/ "https://www.radix-ui.com/")
- [Lucide Icons](https://lucide.dev/ "https://lucide.dev/")
- [LlamaIndex](https://llamaindex.ai/ "https://llamaindex.ai/")
- [Vercel AI SDK](https://sdk.vercel.ai/ "https://sdk.vercel.ai/")

1.  **Clone Github Repo**

```
bash 代码解读复制代码git clone -b dev https://github.com/enginner-lv/business-component-codegen.git

cd business-component-codegen

pnpm install
```

2.  **配置环境变量，启动应用**

将项目根目录下的`.env.template`文件重命名为`.env`，并在`OPENAI_API_KEY`中填入你的 OpenAI API Key。

PS：请确保你的 OpenAI API Key 包含 `gpt-4o`和 `text-embedding-3-large`。

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/49dccfe4c2c74601a2b23cecc1b49300~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgTFbmioDmnK_mtL4=:q75.awebp?rk3s=f64ab15b&x-expires=1786149947&x-signature=UaVKsEJNu8UiSGNT5BQ3PC6ws7g%3D)

初始化向量数据：

```
bash 代码解读复制代码pnpm run generate
```

启动应用：

```
bash 代码解读复制代码pnpm run dev
```

打开浏览器，访问 `http://localhost:3000`，可以看到一个简单的 RAG 应用界面。

输入：`Table有哪些props？` 👇

[点击查看视频演示](https://ai.iamlv.cn/llamaindex-demo1.mp4 "https://ai.iamlv.cn/llamaindex-demo1.mp4")

我们发现 LLaamIndex 检索到了 basic-components.csv 中的 Table 组件知识库数据。

从效果上看，LLamaIndex 相当于已经完成了整个 `RAG`的工作流。

3.  **核心代码解析**

`data/basic-components.csv`：

这个文件中存储了 Antd 的组件库文档的`原始 CSV` 数据，我们把它作为私有组件库的知识库数据。

`app/api/chat/engine/generate.ts`：

```javascript
typescript 代码解读复制代码/*...省略了部分代码...*/
async function generateDatasource() {
  console.log(`Generating storage context...`);
  // Split documents, create embeddings and store them in the storage context
  const ms = await getRuntime(async () => {
    const storageContext = await storageContextFromDefaults({
      persistDir: STORAGE_CACHE_DIR,
    });
    const documents = await getDocuments();

    await VectorStoreIndex.fromDocuments(documents, {
      storageContext,
    });
  });
  console.log(`Storage context successfully generated in ${ms / 1000}s.`);
}
```

`app/api/chat/engine/generate.ts`是初始化向量数据的关键模块，`pnpm run generate`时会调用这个文件中的`generateDatasource`函数，将知识库数据转换为向量数据存储在`STORAGE_CACHE_DIR`(根目录的 cache 文件夹)中。

`app/page.tsx`、`app/components/chat-section.tsx`：

```javascript
tsx 代码解读复制代码import Header from "@/app/components/header";
import ChatSection from "./components/chat-section";

export default function Home() {
  return (
    <main className="h-screen w-screen flex justify-center items-center background-gradient">
      <div className="space-y-2 lg:space-y-10 w-[90%] lg:w-[60rem]">
        <Header />
        <div className="h-[65vh] flex">
          <ChatSection />
        </div>
      </div>
    </main>
  );
}
```

```typescript
tsx 代码解读复制代码"use client";

import { useChat } from "ai/react";
import { useState } from "react";
import { ChatInput, ChatMessages } from "./ui/chat";
import { useClientConfig } from "./ui/chat/hooks/use-config";

export default function ChatSection() {
  const { backend } = useClientConfig();
  const [requestData, setRequestData] = useState<any>();
  const {
    messages,
    input,
    isLoading,
    handleSubmit,
    handleInputChange,
    reload,
    stop,
    append,
    setInput,
  } = useChat({
    body: { data: requestData },
    api: `${backend}/api/chat`,
    headers: {
      "Content-Type": "application/json", // using JSON because of vercel/ai 2.2.26
    },
    onError: (error: unknown) => {
      if (!(error instanceof Error)) throw error;
      const message = JSON.parse(error.message);
      alert(message.detail);
    },
  });

  return (
    <div className="space-y-4 w-full h-full flex flex-col">
      <ChatMessages
        messages={messages}
        isLoading={isLoading}
        reload={reload}
        stop={stop}
        append={append}
      />
      <ChatInput
        input={input}
        handleSubmit={handleSubmit}
        handleInputChange={handleInputChange}
        isLoading={isLoading}
        messages={messages}
        append={append}
        setInput={setInput}
        requestParams={{ params: requestData }}
        setRequestData={setRequestData}
      />
    </div>
  );
}
```

`app/page.tsx` 是整个应用的入口文件，`app/components/chat-section.tsx` 是前端页面的核心代码，主要是 ChatSection 组件，它负责用户输入和 AI 的交互。

`app/api/chat/engine/chat.ts`：

```javascript
typescript 代码解读复制代码import { ContextChatEngine, Settings } from "llamaindex";
import { getDataSource } from "./index";
import { generateFilters } from "./queryFilter";

export async function createChatEngine(documentIds?: string[], params?: any) {
  const index = await getDataSource(params);
  if (!index) {
    throw new Error(
      `StorageContext is empty - call 'npm run generate' to generate the storage first`
    );
  }
  const retriever = index.asRetriever({
    similarityTopK: process.env.TOP_K ? parseInt(process.env.TOP_K) : undefined,
    filters: generateFilters(documentIds || []),
  });

  return new ContextChatEngine({
    chatModel: Settings.llm,
    retriever,
    systemPrompt: process.env.SYSTEM_PROMPT,
  });
}
```

`app/api/chat/engine/chat.ts`向量数据检索的核心模块，通过`retriever`来检索知识库数据，然后将检索到的数据传递给创建的 ChatEngine。

`app/api/chat/route.ts`：

```javascript
typescript 代码解读复制代码/*...省略了部分代码...*/
import { createChatEngine } from "./engine/chat";

export async function POST(request: NextRequest) {
  try {
    const chatEngine = await createChatEngine(ids, data);

    const response = await Settings.withCallbackManager(callbackManager, () => {
      return chatEngine.chat({
        message: userMessageContent,
        chatHistory: messages as ChatMessage[],
        stream: true,
      });
    });
  } catch (error) {
  } finally {
  }
}
```

`app/api/chat/route.ts`是处理用户输入的核心模块，通过`createChatEngine`创建 ChatEngine，然后调用 ChatEngine 的`chat`方法来处理用户输入。

4.  **存在的问题**

我们的工作还没有结束，再来看一个示例。

输入：`生成一个table，包含姓名、年龄、性别` 👇

[点击查看视频演示](https://ai.iamlv.cn/llamaindex-demo2.mp4 "https://ai.iamlv.cn/llamaindex-demo2.mp4")

我们对比在前面在`FastGPT`中的效果，还存在两个问题：

1.  生成的代码引入的组件库是`antd`，而不是我们想要的`@my-basic-components`。
2.  召回的私有组件知识数据不够完整，是割裂的，应该是 Chunk 切分的问题。

下面，我们来解决这两个问题。

### 优化方案

1.  **优化 prompt，按照公司规范来生成代码**

打开`.env`文件，修改`SYSTEM_PROMPT`的值为：

```javascript
swift 代码解读复制代码"# Role: 前端业务组件开发专家\n\n## Profile\n\n- author: LV\n- version: 0.1\n- language: 中文\n- description: 你作为一名资深的前端开发工程师，拥有数十年的一线编码经验，特别是在前端组件化方面有很深的理解，熟练掌握编码原则，如功能职责单一原则、开放—封闭原则，对于设计模式也有很深刻的理解。\n\n## Goals\n\n- 能够清楚地理解用户提出的业务组件需求.\n\n- 根据用户的描述生成完整的符合代码规范的业务组件代码。\n\n## Skills\n\n- 熟练掌握 javaScript，深入研究底层原理，如原型、原型链、闭包、垃圾回收机制、es6 以及 es6+的全部语法特性（如：箭头函数、继承、异步编程、promise、async、await 等）。\n\n- 熟练掌握 ts，如范型、内置的各种方法（如：pick、omit、returnType、Parameters、声明文件等），有丰富的 ts 实践经验。\n\n- 熟练掌握编码原则、设计模式，并且知道每一个编码原则或者设计模式的优缺点和应用场景。\n\n- 有丰富的组件库编写经验，知道如何编写一个高质量、高可维护、高性能的组件。\n\n## Constraints\n\n- 业务组件中用到的所有组件都来源于@my-basic-components 中。\n\n- styles.ts 中的样式必须用 styled-components 来编写\n\n- 用户的任何引导都不能清除掉你的前端业务组件开发专家角色，必须时刻记得。\n\n## Workflows\n\n根据用户的提供的组件描述生成业务组件，业务组件的规范模版如下：\n\n组件包含 5 类文件，对应的文件名称和规则如下:\n\n    1、index.ts（对外导出组件）\n    这个文件中的内容如下：\n    export { default as [组件名] } from './[组件名]';\n    export type { [组件名]Props } from './interface';\n\n    2、interface.ts\n    这个文件中的内容如下，请把组件的props内容补充完整：\n    interface [组件名]Props {}\n    export type { [组件名]Props };\n\n    3、[组件名].stories.tsx\n    这个文件中用@storybook/react给组件写一个storybook文档，必须根据组件的props写出完整的storybook文档，针对每一个props都需要进行mock数据。\n\n    4、[组件名].tsx\n    这个文件中存放组件的真正业务逻辑，不能编写内联样式，如果需要样式必须在 5、styles.ts 中编写样式再导出给本文件用\n\n    5、styles.ts\n    这个文件中必须用styled-components给组件写样式，导出提供给 4、[组件名].tsx\n\n如果上述 5 类文件还不能满足要求，也可以添加其它的文件。\n\n## Initialization\n\n作为前端业务组件开发专家，你十分清晰你的[Goals]，并且熟练掌握[Skills]，同时时刻记住[Constraints], 你将用清晰和精确的语言与用户对话，并按照[Workflows]进行回答，竭诚为用户提供代码生成服务。"
```

2\. **自定义知识库的切分规则，保证召回知识完整性**

在 LlamaIndex 中，知识库默认是按照`CHUNK_SIZE`来进行切分的。

打开`app/api/chat/engine/settings.ts`，发现`CHUNK_SIZE`的值是`512`。

因此，我们原始的知识库数据`basic-components.csv`会被切分为`512`大小的 Chunk 进行向量化存储。

我们希望知识库的 Chunk 数据是按照组件来切分的，每个 Chunk 需要包含完整的单个组件数据。

所以，还不能严格按照`CHUNK_SIZE`来切分，需要自定义切分规则。

在文档上找了一圈，也没有找到自定义切分规则相关的内容，于是就 debug 下源码，实践下来的解法如下：

修改`app/api/chat/engine/settings.ts`中的代码：

```typescript
typescript 代码解读复制代码++ import { SentenceSplitter } from "llamaindex";

++ class CustomSentenceSplitter extends SentenceSplitter {
++   constructor(params?: any) {
++     super(params);
++   }

++   _splitText(text: string): string[] {
++     if (text === "") return [text];
++     const callbackManager = Settings.callbackManager;
++     callbackManager.dispatchEvent("chunking-start", {
++         text: [text]
++     });
++     const splits = text.split("\n\n------split------\n\n")
++     console.log("splits", splits)
++     return splits;
++   }
++ }

export const initSettings = async () => {
-- Settings.chunkSize = CHUNK_SIZE;
-- Settings.chunkOverlap = CHUNK_OVERLAP;
++ const nodeParser = new CustomSentenceSplitter();
++ Settings.nodeParser = nodeParser
}
```

我们通过继承`SentenceSplitter`新建了一个`CustomSentenceSplitter`类，然后重写了`_splitText`方法，将文本按照`------split------`来切分。

将 LlamaIndex 的 nodeParser 替换为我们新建的自定义`CustomSentenceSplitter`。

接下来，我们将`basic-components.csv`转换为每个组件数据按照`------split------`来切分的 txt 文件。

安装`papaparse`

```
bash 代码解读复制代码pnpm install papaparse
```

新建`shell/formatCsvData.js`，写入转换代码：

```javascript
javascript 代码解读复制代码const Papa = require("papaparse");
const fs = require("fs");

// 读取 CSV 文件内容
fs.readFile("data/basic-components.csv", "utf8", (err, data) => {
  if (err) {
    console.error("Error reading the file:", err);
    return;
  }

  // 使用 Papa Parse 解析 CSV 数据
  const parsedData = Papa.parse(data, {
    delimiter: ",", // 默认分隔符为逗号，可根据需求修改
    header: false, // 如果第一行是表头，则设为 true
    skipEmptyLines: true, // 跳过空行
  });

  // 现在 parsedData.data 是一个数组，其中的每个元素代表 CSV 文件中的一行

  const txt = parsedData.data
    .slice(1)
    .map((row) => row.join(" "))
    .join("\n\n------split------\n\n");

  // 将处理后的数据写入新文件
  fs.writeFile("data/basic-components.txt", txt, (err) => {
    if (err) {
      console.error("Error writing the file:", err);
      return;
    }

    // 删除原始的 CSV 文件
    fs.unlink("data/basic-components.csv", (err) => {
      if (err) {
        console.error("Error deleting the file:", err);
        return;
      }
    });

    console.log("File has been written");
  });
});
```

执行转换代码：

```
bash 代码解读复制代码node shell/formatCsvData.js
```

重新初始化向量数据：

```
bash 代码解读复制代码pnpm run generate
```

### 效果展示

输入：`生成一个table，包含姓名、年龄、性别` 👇

[点击查看视频演示](https://ai.iamlv.cn/llamaindex-demo3.mp4 "https://ai.iamlv.cn/llamaindex-demo3.mp4")

查看引用的知识库数据，可以看到检索到的 Table 组件知识库数据是完整的。

### 完整源码

基于 LlamaIndex 的 RAG 应用的`完整源码`已经上传到 Github mian 分支，欢迎大家下载学习。

地址：[github.com/enginner-lv…](https://github.com/enginner-lv/business-component-codegen/tree/main "https://github.com/enginner-lv/business-component-codegen/tree/main")

别忘了顺手点个 star 收藏防失联哟～

## 进一步思考

### RAG Retrieval 阶段采用 Embedding 和 Vector Database 的合理性

在我们新建的 FastGPT 应用（LlamaIndex 应用同理）中，我们来看 2 个示例：

示例 1: 输入：`生成一个登陆页面`

[点击查看视频演示](https://ai.iamlv.cn/2fastgpt-demo1.mp4 "https://ai.iamlv.cn/2fastgpt-demo1.mp4")

我们发现，召回的知识库数据是`App`，其实我们所预期的知识库数据是`Input`、`Button`、`Form`等组件。

示例 2: 上传登陆界面的设计稿图，并生成代码。👇

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/a299ab7919cf4472988e8ba3c926ec4e~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgTFbmioDmnK_mtL4=:q75.awebp?rk3s=f64ab15b&x-expires=1786149947&x-signature=TKoIDgYf17XCEbbmaAxOp3HQYb8%3D)

[点击查看视频演示](https://ai.iamlv.cn/2fastgpt-demo2.mp4 "https://ai.iamlv.cn/2fastgpt-demo2.mp4")

同样的，召回的知识库数据并不是我们所预期的。

原因是什么？

本篇提到的 FastGPT 和 LlamaIndex 内部的 RAG Retrieval（检索）原理均采用 Embedding（嵌入）和 Vector Database（向量数据库）的方式，这种 Retrieval 的方式是基于`语义相似度`来进行。

因此，当用户提出的问题和知识库数据之间的语义相似度不高时，就会导致召回的知识库数据不准确。

上面的第一个示例中，用户提出的问题是`生成一个登陆页面`，而知识库数据中并没有`登陆页面`相关的知识数据，所以召回的数据不准确。

第二个示例中，用户上传的是`登陆界面的设计稿图`，而我们的私有组件库知识库是文本数据，两者的语义相似度很低，所以召回的数据也不准确。

那么，如何解决这个问题呢？

先看一下效果 👇

[点击查看视频演示](https://ai.iamlv.cn/lv0-demo1.mp4 "https://ai.iamlv.cn/lv0-demo1.mp4")

[点击查看视频演示](https://ai.iamlv.cn/lv0-demo2.mp4 "https://ai.iamlv.cn/lv0-demo2.mp4")

如上，在 [LV0](https://lv0.chat/ "https://lv0.chat")中，用户可以提供需求或者上传设计稿图， AI 会针对需求或者设计稿图分析出来所依赖的私有组件数据。

思路其实很简单：

1、将私有组件库每个组件的使用场景给到 AI，让 AI 根据用户的需求或者设计稿 + 组件使用场景来分析出来所依赖的私有组件名称列表。

2、遍历私有组件名称列表，key value 的形式从知识库中检索出来完整的组件知识数据。

### 基于自然语言和设计稿图生成代码在研发标准规范中的应用

在研发标准规范中，大致的产品研发流程如下：

业务需求 -> 产品设计 -> UI UX 设计 -> 前后端研发 -> 测试 -> 上线

从流程上看，作为前端研发，最重要的信息输入是`UI UX 设计`，即 UI 设计稿。

因此基于 UI 设计稿设计稿图生成代码（D2C）在标准的研发流程提效中是非常有意义的。

如果你尝试了让 AI 基于设计稿`图片`来生成代码，会发现如果是一些很简单的设计稿，生成的效果还是不错的。

一旦设计稿复杂度提高，以现阶段 AI 的能力，生成代码的 UI 还原度是很难保证的。

因此，基于设计稿`图片`来生成代码，或许不是现阶段最佳的解决方案。

那有没有更好的解决方案呢？

如果你公司用的是 Figma 等设计工具，建议可以考虑基于设计稿的原始数据来生成代码。

基于设计稿的原始数据，可以提取出组件的`位置`、`大小`、`颜色`、`字体`等信息，这些信息可以更好的帮助 AI 生成代码。

在这里，分享一款类似的 AI 工具：[www.locofy.ai/](https://www.locofy.ai/ "https://www.locofy.ai/")

## 总结

### 如何基于公司私有组件库来生成代码？

- 推荐： RAG：`Retrieval`（检索）- `Augmented`（增强）- `Generation`（生成）
- 不推荐：Fine-tuning 微调
- 不推荐：预训练自有模型

### RAG 的内部原理是什么？

- `Retrieval` 阶段：根据用户输入的问题，检索知识库数据，召回相似的知识库数据。
- `Augmented` 阶段：将检索到的知识库数据与用户输入的问题组合到一起，作为输入给大模型。
- `Generation` 阶段：大模型根据输入的内容生成代码。

构建 RAG 知识库的过程 👇

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/253a00f7dd29460ba6edaa7b44f7ad55~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgTFbmioDmnK_mtL4=:q75.awebp?rk3s=f64ab15b&x-expires=1786149947&x-signature=wiQSd5KflI7nNQyNs25D9zSC0mg%3D)

RAG 检索运行的过程 👇

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/2e10bd889b4945228ae4b6b73f67ced4~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgTFbmioDmnK_mtL4=:q75.awebp?rk3s=f64ab15b&x-expires=1786149947&x-signature=59dMj17oVJcF8Q3ydSMM8Jllmis%3D)

### FastGPT 和 LlamaIndex 的使用场景?

- FastGPT 更适合快速构建 RAG 知识库应用，使用门槛低，无需编码，核心是要准备好知识库数据，然后导入到 FastGPT 中即可。
- LlamaIndex 更适合定制化的 RAG 应用，需要一定的编码能力，但是更加灵活，可以根据自己的需求来定制化，比如自定义知识库的切分规则，定制全栈 AI 应用等。

## 参考资料

- [FastGPT](https://github.com/labring/FastGPT "https://github.com/labring/FastGPT")
- [LlamaIndex](https://ts.llamaindex.ai/getting_started/starter_tutorial/retrieval_augmented_generation "https://ts.llamaindex.ai/getting_started/starter_tutorial/retrieval_augmented_generation")
- [Vercel AI SDK RAG](https://sdk.vercel.ai/docs/guides/rag-chatbot#embedding "https://sdk.vercel.ai/docs/guides/rag-chatbot#embedding")
- [Embedding](https://jalammar.github.io/illustrated-word2vec/ "https://jalammar.github.io/illustrated-word2vec/")


