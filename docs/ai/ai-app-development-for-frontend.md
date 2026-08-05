---
title: AI 应用开发入门：前端也可以学习 AI
url: https://juejin.cn/post/7517197769247391756?searchId=20260805221256C12157A4AACDDEEFE7BC
publishedTime: 2025-06-19T11:09:28+08:00
---

## 前言

该篇主要是分享自己作为一个前端工程师学习 AI 的历程。你将学习到：

- 如何选择学习路线
- 学习 AI 相关的基础概念
- 学习 AI 应用开发的基本技术方案
- 学习基于 LLM 开发一个简单的 AI 编程助手

## 一、寻找方向

### 1.1 面对焦虑和兴趣

现在 AI 领域发展迅速，新的概念和技术层出不穷。不经会让人产生两种情绪

- 焦虑：面对未知技术概念 + “程序员即将被 AI 取代”的话题
- 兴趣：作为技术人，面对向 AI 这样的新兴技术又感觉到异常的兴奋

![image.png](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/756cea8989d74e949dc1b44f4e3fbb67~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5ZSQ5p-Q5Lq65Li2:q75.awebp?rk3s=f64ab15b&x-expires=1786050289&x-signature=L5sPF26iEZcS7vNXWC6kD6sFObU%3D)

所以我就在思考，作为一个前端工程师，面对如今 AI 的浪潮，我该怎么融入到这个圈子里面去呢

### 1.2 尝试寻找方向

随着各种 AI 应用产品的不断出现，获取信息的方式也开始变得便捷。于是我去问 AI：“作为一个前端工程师，我能干点什么？”

![image.png](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/7da0dc66237f44419a307c2ae381a3a2~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5ZSQ5p-Q5Lq65Li2:q75.awebp?rk3s=f64ab15b&x-expires=1786050289&x-signature=av4sDdi%2BlskjKT3TurwJzGSSwnw%3D)

说实话，当时看完 AI 的回复，我更懵逼了。因为它一上来就让我学习各种数学概念、算法、模型架构、机器学习等等。我相信很多非人工智能领域的开发者，看完直接就劝退了。并且会产生“AI 的发展，跟我没有关系”的念头。

### 1.3 找准目标

随着后来不断地学习，我发现想清楚目标很重要。你要想融入 AI 这个圈子，就得知道自己学这个东西，是为了达成什么目的

随着不断摸索，我发现有两条学习路线可以供大家参考：

- **AI 应用开发**：学习关键概念、实现应用的技术方案。目的是能够开发不同的 AI 应用产品，例如 Dify、豆包、Chatbox、Cursor 等
- **模型训练开发**：学习数学模型、机器学习等等，目的是训练出不同能力的模型，例如 GPT-4、Qwen3、DeepSeek-R1 等

![image.png](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/13a734ae488a4daaba5a7b41f1f624f2~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5ZSQ5p-Q5Lq65Li2:q75.awebp?rk3s=f64ab15b&x-expires=1786050289&x-signature=Qtb6p%2FMGwRyRbRfFaRUKdNEZ6Rk%3D)

本篇介绍的就是关于 AI 应用开发入门的相关内容。后续如果你想更深入的学习，也可以按照上面这个两个思路去搜索相关的内容。

## 二、认识 AI 应用

### 2.1 哪些是 AI 应用

关于 AI 应用，这里举一些日常生活中常见的产品

#### AI 低代码平台

类似 Dify、Coze 通过配置，制作出不同的 AI 应用（或者说智能体）

![image.png](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/a9c053485da94afa90c554173ae635d3~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5ZSQ5p-Q5Lq65Li2:q75.awebp?rk3s=f64ab15b&x-expires=1786050289&x-signature=%2FFFClwJZDI7SN0MrlSiWQnN1GnI%3D)

#### AI 辅助编程

类似 Cursor、Trae，通过 IDE 集成 AI 能力，帮助专注程序员的编码提效

![image.png](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/0a733b58db3e4c05b57e45ddaeee07e9~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5ZSQ5p-Q5Lq65Li2:q75.awebp?rk3s=f64ab15b&x-expires=1786050289&x-signature=HofBkWyvAXJ%2BSnLJ5qAgIypC678%3D)

#### AI 应用客户端

类似 ChatBox、Claude for Desktop、豆包客户端。通过本地客户端与 AI 结合，让 AI 能力边界扩展到你的电脑（如文件读写、操作 word 等）

![image.png](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/919cabf04a0d4532a56be0b9004c8d87~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5ZSQ5p-Q5Lq65Li2:q75.awebp?rk3s=f64ab15b&x-expires=1786050289&x-signature=NwNmOJAqO%2BhtI1Leh0Mt7RRiv3E%3D)

### 2.2 AI 应用基本结构

不管产品形态怎么变，其实简化后都是下面这三个主体。只不过不同产品，会在流程和设计上有自己的改造和延伸。

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/63073a46ff5b45ebb3248a035ef57ba2~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5ZSQ5p-Q5Lq65Li2:q75.awebp?rk3s=f64ab15b&x-expires=1786050289&x-signature=MM6x54n9MRARzdtyOoqYqk13RPw%3D)

- 交互界面：用户可操作的界面，用来处理用户输入和渲染 LLM 的输出
- Service：处理用户的输入，主要负责**提示词的组装**和**工具调用**
- LLM：大模型的核心服务

缺少基础概念的同学，整篇文章看下来可能会比较懵。所以下面先带大家来共识一下关于“大模型“和”提示词“的基础概念，然后再来开展实战的学习。

## 三、基础概念

### 什么是大模型、模型参数、参数文件

**大模型**

大模型通常说的是大语言模型。英文名 Large Language Model，缩写是 LLM。它通过学习海量的数据，能够理解和生成像人类一样的语言或完成复杂任务。

**模型参数**

如果大模型是大脑，那模型参数就大脑里面的神经元。就像你学说话时，会把学到的词汇、语法规则记在你的大脑神经元当中。一般来说，一个人的神经元越丰富，人就越聪明。同理，大模型的参数越大，能力就越强。常说的 1.5b、7b 就是表示模型参数的大小。

**参数文件**

参数文件是存储人工智能模型中**已训练参数**的重要文件。它包含模型在训练过程中学习到的所有权重和偏置值，模型通过这些参数来完成特定任务，比如语言生成、图像识别等。

### 本地安装 LLM

可以下载一个 [Ollama](https://ollama.com/ "https://ollama.com/")，它是一个用于本地部署和使用大模型的工具。这里我们将通过本地部署 LLM 的形式，来带你体验一下 LLM 是什么

![image.png](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/399ba95d4a804cd1a71f60a54f6a3801~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5ZSQ5p-Q5Lq65Li2:q75.awebp?rk3s=f64ab15b&x-expires=1786050289&x-signature=rDdF6WDvTMBY38bXoZHda9GuAJM%3D) 安装后点击启动，就可以在终端测试是否安装成功

![image.png](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/0293657089b3412f9061e1fa73496f89~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5ZSQ5p-Q5Lq65Li2:q75.awebp?rk3s=f64ab15b&x-expires=1786050289&x-signature=tAQ7rQsKOwXHEnoDo8KZEnpUKUA%3D)

接下来便是挑选合适的大模型。选择一个你喜欢的模型，下载时需注意模型参数大小。如下图的 deepseek 模型，它分别有 1.5b ~ 671b 的参数可以选择。模型参数越大，你下载的**参数文件**就越大，运行时所需的内存就越大。

![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/9ae542990997400499b773097bfdc40b~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5ZSQ5p-Q5Lq65Li2:q75.awebp?rk3s=f64ab15b&x-expires=1786050289&x-signature=84qEjp7rUjn1%2FNW2g4BpN%2F6fjC4%3D)

执行下载命令（也是运行指令）。这里我选择的是 deepseek-r1 一个 7b 的模型，这里主要根据你的设备条件来选择，不过多赘述。

```
bash 代码解读复制代码ollama run deepseek-r1:7b
```

### 如何与大模型交互

接下来你可以与人类聊天一样，正常与它对话了。你会发现哪怕是断网，它也是可以正常回复你内容的。

![image.png](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/211cfb67e1b64f9995d7d07c4eb19a78~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5ZSQ5p-Q5Lq65Li2:q75.awebp?rk3s=f64ab15b&x-expires=1786050289&x-signature=P2MT17XCRfb4kYfcp4EqSIT1Sfs%3D)

这里之所以演示通过部署本地 LLM 并进行交互，是想告诉你 LLM 本身是一个“离线可执行的智能程序”，它并不需要联网搜索外界信息，它是一个提前“学习”了很多知识的“智能程序”（或“可执行文件”）。以 macOS 为例，你下载的**参数文件** 都在 ~/.ollama/models/blobs 下。当你通过`olllama run deepseek-r1:7b`时，它就会把这些“知识”加载进来。 ![image.png](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/28c8ace0dadd46d9ba4352e1c6dec21e~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5ZSQ5p-Q5Lq65Li2:q75.awebp?rk3s=f64ab15b&x-expires=1786050289&x-signature=Jw7oqJ50%2FO%2FZ4zD33kRsMVwthDI%3D)

> 本地大模型一般是经过量化的，一般来说个人电脑的配置有限，能跑的参数规格也会比较小，所以不要期待它有太高的智商。实际开发中可以选择大模型厂商提供的服务来进行使用。

### 什么是提示词 Prompt

比如，上面你通过终端给大模型的输入，或者下图中通过 UI 界面输入的信息，都属于提示词的“一部分”（为什么是一部分，后面提示词工程章节会细说）。

![image.png](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/211c7074172844f8b564058fa8351c84~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5ZSQ5p-Q5Lq65Li2:q75.awebp?rk3s=f64ab15b&x-expires=1786050289&x-signature=p%2FAVffP5sUVjGKKuy%2FPafH54E%2FU%3D) 提示词（Prompt）是大模型中的一种输入方式，就是我们给模型的一段文字或问题，用来告诉它我们想要得到什么样的回答。提示词就像是在和模型对话时的指令或提示，帮助引导模型生成更符合你需求的输出。

**换句话说，如果大模型是你的程序员，你是一个产品需求。如何让程序员把产品做的更符合要求，取决于你给程序员描述的是否清楚。**

所以提示词的处理和设计非常重要，我们接下来的开发实践，基本都将围绕着提示词进行开展。

## 四、开发实践

学完了基础概念，这一章节我们会通过**实现一个 AI 编程助手**，来让你理解 AI 应用开发最基本的逻辑。

### 4.1 需求分析

这里我们先来确认一下具体的需求。

核心：实现一个能够帮我写代码的 AI 编程助手。

功能：

1.  我们希望它擅长解决前端领域的各种问题，能够精通 react、webpack、Antd 等常见用的前端框架
2.  希望它解决问题时，能够给出具体的解决方案，从「设计思路」、「代码实现」两个方面输出
3.  可以命令它把代码写到某某文件中

### 4.2 搭建基础服务

上面说到，一个 AI 应用具备三个最基本的主体。下面我们来开发 Service 这个主体，它是 AI 应用中非常核心的一部分。

#### 创建模型服务

首先，你可以选择任意一家模型厂商，开通一个模型服务来使用。这里我选择了字节的火山引擎提供的 deepseek-v3 的一个模型

![image.png](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/994ef9a61e094fc08f6d2149d295daef~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5ZSQ5p-Q5Lq65Li2:q75.awebp?rk3s=f64ab15b&x-expires=1786050289&x-signature=OgAGyVMHnVXBsqdDRZnYcH9Y7Bs%3D)

#### 开发 SSE 接口

为了方便，这里基于 Express 快速搭建一个服务。

> 以下所有环节，我都只演示核心代码，具体代码细节可以去到对应的仓库分支查阅。

声明一个 POST 的接口，使用 SSE 传输协议。这里我们主要做的事情是：

- 设置 SSE 响应头
- 将用户的输入传给 LLM 的接口
- 调用 handleSseResponse 处理 LLM 返回内容并推送给前端

```javascript
js 代码解读复制代码app.post('/api/chat', validateChatRequest, async (req, res) => {
  try {
    const { query } = req.body;

    // 设置SSE响应头
    res.setHeader('Content-Type', 'text/event-stream');
    res.setHeader('Cache-Control', 'no-cache');
    res.setHeader('Connection', 'keep-alive');
    res.flushHeaders();

    // 发送初始数据
    res.write('data: {"status": "started"}\n\n');

    // 调用API并流式返回结果
    const response = await fetch(`${API_BASE_URL}/chat/completions`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        Authorization: `Bearer ${ARK_API_KEY}`
      },
      body: JSON.stringify({
        model: MODEL_NAME,
        messages: [{ role: 'user', content: query }],
        stream: true
      })
    });

    if (!response.ok) {
      throw new Error(`HTTP error! status: ${response.status}`);
    }

    await handleSseResponse(res, response.body);
    res.end();
  } catch (error) {
    console.error('调用API时出错:', error.message);
    res.write(`data: {"error": "抱歉，发生了错误: ${error.message}"}\n\n`);
    res.end();
  }
});
```

然后需要在 handleSseResponse 中处理 LLM 返回的内容。这里主要的事情是：

- 通过 reader 不断读取 LLM 返回内容
- 解析数据
- 将数据拼接成我们的格式，传递给前端

```javascript
js 代码解读复制代码// 处理SSE响应
async function handleSseResponse(res, stream) {
  const reader = stream.getReader();
  const decoder = new TextDecoder();

  try {
    while (true) {
      // 循环读取 LLM 服务推送过来的数据
      const { done, value } = await reader.read();
      if (done) break;

      const chunk = decoder.decode(value, { stream: true });
      const lines = chunk.split('\n').filter(line => line.trim());

      for (const line of lines) {
        if (!line.startsWith('data:')) continue;

        const data = line.slice(5).trim();
        if (data === '[DONE]') {
          res.write('data: {"status": "completed"}\n\n');
          continue;
        }


         // 解析数据
        try {
          const parsed = JSON.parse(data);
          if (parsed.choices) {
            // 将数据拼接成我们需要的格式传递给前端
            res.write(`data: {"content": "${escapeSse(parsed.choices[0].delta.content)}"}\n\n`);
          }
        } catch (e) {
          console.error('解析响应失败:', e);
        }
      }
    }
  } catch (error) {
    throw new Error(`流处理错误: ${error.message}`);
  } finally {
    reader.releaseLock();
  }
}

```

### 4.2 前端渲染

接着，我们来开发前端。前端主要做的事情是两件：

- 调用 SSE 接口，接收后端不断推送过来的数据块
- 将数据解析拼接以后，交给 Markdown 渲染器，渲染成富文本

前端代码如下：

```javascript
js 代码解读复制代码import React, { useState, useRef } from 'react';
import { marked } from 'marked';

export default function App() {
  const [messages, setMessages] = useState([]);
  const [input, setInput] = useState('');
  const [loading, setLoading] = useState(false);

  const handleSend = async () => {
    if (!input.trim()) return;
    setMessages((msgs) => [...msgs, { role: 'user', content: input }]);
    setLoading(true);
    let aiContent = '';
    const newMsg = { role: 'assistant', content: '' };
    setMessages((msgs) => [...msgs, newMsg]);

    // 使用 EventSource 兼容 SSE
    const response = await fetch('/api/chat', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ query: input })
    });
    const reader = response.body.getReader();
    const decoder = new TextDecoder();
    let done = false;
    let buffer = '';
    while (!done) {
      const { value, done: doneReading } = await reader.read();
      done = doneReading;
      if (value) {
        buffer += decoder.decode(value, { stream: true });
        const parts = buffer.split('\n\n');
        buffer = parts.pop();
        for (const part of parts) {
          if (part.startsWith('data: ')) {
            try {
              const data = JSON.parse(part.slice(6));
              if (data.content) {
                aiContent += data.content.replace(/\\n/g, '\n').replace(/\\r/g, '\r').replace(/\\"/g, '"');
                setMessages((msgs) => {
                  const updated = [...msgs];
                  updated[updated.length - 1] = { role: 'assistant', content: aiContent };
                  return updated;
                });
              }
            } catch {}
          }
        }
      }
    }
    setLoading(false);
    setInput('');
  };

  return (
    <div style={{ maxWidth: 600, margin: '40px auto', fontFamily: 'sans-serif' }}>
      <h2>AI 对话演示</h2>
      <div style={{ border: '1px solid #eee', borderRadius: 8, padding: 16, minHeight: 300, background: '#fafbfc' }}>
        {messages.map((msg, i) => (
          <div key={i} style={{ margin: '12px 0', textAlign: msg.role === 'user' ? 'right' : 'left' }}>
            <div style={{ display: 'inline-block', maxWidth: '90%', background: msg.role === 'user' ? '#d2eafd' : '#f3f3f3', borderRadius: 6, padding: 8 }}>
              <span dangerouslySetInnerHTML={{ __html: marked.parse(msg.content || '') }} />
            </div>
          </div>
        ))}
        {loading && <div style={{ color: '#aaa' }}>AI 正在思考...</div>}
      </div>
      <div style={{ display: 'flex', marginTop: 16 }}>
        <textarea
          value={input}
          onChange={e => setInput(e.target.value)}
          rows={2}
          style={{ flex: 1, resize: 'none', borderRadius: 4, border: '1px solid #ccc', padding: 8 }}
          placeholder="请输入你的问题..."
          disabled={loading}
        />
        <button
          onClick={handleSend}
          disabled={loading || !input.trim()}
          style={{ marginLeft: 8, padding: '0 20px', borderRadius: 4, border: 'none', background: '#1677ff', color: '#fff', fontWeight: 600 }}
        >发送</button>
      </div>
    </div>
  );
}

```

至此，你已经完成了一个最最基本的 AI 应用。

![image.png](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/8e5cbcb009b246bbaf91f01d19eb33b4~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5ZSQ5p-Q5Lq65Li2:q75.awebp?rk3s=f64ab15b&x-expires=1786050289&x-signature=v5wdPzN9XgLw2SpKzB%2BQ2C3VwFc%3D)

> 上面详细代码在：[github.com/zixingtangm…](https://github.com/zixingtangmouren/ai-study-demo/tree/demo-01 "https://github.com/zixingtangmouren/ai-study-demo/tree/demo-01")

### 4.4 面临的问题

但是作为一个完善 AI 编程助手，还需要面对以下几个问题。下面这些问题的解决方案，是本文讲解的重点，也是开发一个 AI 应用的基础。

#### 角色的设定

因为开发的是一个编程助手的应用，所以我们希望这个 AI 助手尽可能回复的是编程指导、代码编写相关的东西。所以必须给设定 LLM 一个角色，让它知道自己是谁，要做什么。

#### 记住历史对话

LLM 本身是不具备对话记忆能力的。这些历史对话的内容，得靠我们上层的应用来管理，并且需要在每次新一轮的对话中传递给它。让 LLM 记住历史对话是一个非常必要的功能，就好比你跟人类沟通，很多时候你也没办法通过一轮沟通就能得出结论，与 LLM 交流也是同理。

#### 输出格式规范

作为一个 AI 编程助手，我们期望它在面对用户关于代码的问题时，能够以“解决思路、代码实现”两个维度来给出具体的解决方案，所以在 LLM 输出的内容上也需要加上一些规范。

#### 信息有限

如果需要 AI 编程助手解决的是一个公司内部的项目，例如，项目用的都是公司内部封装的组件，内部的业务逻辑。但是它作为一个通用的模型，又没有学习过跟你公司项目相关的知识，我们只能通过“外挂”一个知识库的方式来解决 LLM 信息有限的问题。

#### 能力边界有限

作为一个 AI 编程助手，你可能期望它帮你输出了方案以后，能够自己把方案写到你的电脑里。这就又涉及到，如何让 LLM 调用外部工具的能力。

#### 小结

这些看似复杂的问题，其实都是**通过提示词工程来解决的**。虽然技术方案会有不同，但是最终都是的落地，还是在输入给 LLM 的提示词上。

### 4.5 提示词工程

我将通过在代码中编写固定的提示词，来模拟演示如何解决上述的问题。这里我们不会用到任何跟代码相关的技术方案，都是纯文字描述。

#### 代码层的提示词

我们先来看一下所谓的提示词在代码层面是什么样子的。

你可能以为通过用户界面输入的文字，就是最终输入给 LLM 的提示词。其实不然，你的输入只是提示词中的一部分。 从代码层面上来看，你的输入只是 Messages 参数的一部分，大多数 LLM 服务都封装一个这样的参数。

![image.png](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/50f0eea63b5e47998b3632b7a4eda0ec~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5ZSQ5p-Q5Lq65Li2:q75.awebp?rk3s=f64ab15b&x-expires=1786050289&x-signature=INHSD9DpEY%2ByDvkLWxNPuX6yb2o%3D)

这个 Messages 表示的是一组对话，每个 Message 的 role 用来表示说话的角色，常见的分为四种：

- system：系统提示词。一般是应用开发者设定好的。
- user: 用户。表述用户输入的信息。
- assistant: AI。表示 AI 回复的信息。
- tool：工具。表示工具执行的结果

content 则是表示具体的说话内容，最终它会被拼接成一个单一的提示词字符串传给 LLM。例如下面的例子，我们输入的参数是这样

```
json 代码解读复制代码[
  {"role": "system", "content": "你是一名友好的AI助手。"},
  {"role": "user", "content": "什么是机器学习？"},
  {"role": "assistant", "content": "机器学习是让计算机从数据中自主学习的一种技术，而无需显式编程。"},
  {"role": "user", "content": "那它怎么应用在日常生活中？"}
]
```

但是 LLM 的平台会将它拼接成如下的字符串。模型对于上下文如何理解，如何输出内容，就是取决下面这些对话文本的输入。

```
txt 代码解读复制代码系统: 你是一名友好的AI助手。
用户: 什么是机器学习？
助手: 机器学习是让计算机从数据中自主学习的一种技术，而无需显式编程。
用户: 那它怎么应用在日常生活中
```

这里还会额外添加一些特殊的标记或格式（不同平台和模型可能略有不同），用来帮助模型区分不同角色。

#### 系统提示词

作为入门，提示词的其他概念不过多赘述，只需要关注**系统提示词**这个概念即可。

一般来说 System 提示词部分的内容会被重点标记，LLM 分配给这部分的注意力权重也会更加高。所以一般它的作用是作为模型的行为指导，让模型在整个对话中保持一致的角色、风格、内容主题。

以我们的这个 AI 编程助手的场景为例，我们希望 LLM 的回复都是**编程指导相关**的内容，以及我们希望关于代码的部分，它**能够按照 “设计思路、代码实现” 两个维度**进行回答，所以要这样做：

![image.png](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/348e44c41a5349c9adcdcde87bff799a~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5ZSQ5p-Q5Lq65Li2:q75.awebp?rk3s=f64ab15b&x-expires=1786050289&x-signature=X7pLHBVABHbrip3O0f%2Fo3WRXuz4%3D)

> 上面的系统提示词，使用了 Markdown 的形式，目的是为了让提示词的结构更加清晰。

设定完毕以后，可以尝试询问一下，来确定是否符合预期。

![image.png](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/fc04f0cb654849fdb33f8256b6c9c817~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5ZSQ5p-Q5Lq65Li2:q75.awebp?rk3s=f64ab15b&x-expires=1786050289&x-signature=Qpo871Ste366Vpc9UuBni8uYWBk%3D)

现在 LLM 知道自己是谁了，以及它知道了自己的主要工作是什么

#### 历史对话

目前 LLM 不具备记忆对话的能力，你可以试一下。

![image.png](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/e618b570f9de4e4885ef822bf1662de5~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5ZSQ5p-Q5Lq65Li2:q75.awebp?rk3s=f64ab15b&x-expires=1786050289&x-signature=GZM2sSJEK0VPcBXc%2FtZZUvsHktM%3D)

![image.png](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/da57b1cba84d4439a457f33541c34dab~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5ZSQ5p-Q5Lq65Li2:q75.awebp?rk3s=f64ab15b&x-expires=1786050289&x-signature=8DGxpVd6zqRl3ilEHNtjauT%2FZ%2BM%3D)

让 LLM 记得之前对话内容的办法就是，每次新的对话**都把过去的聊天的内容全部塞到提示词中**。对于 LLM 而言，每次输入都需要重新看一遍之前的内容，这些内容就是所谓的**上下文**。这个就是理解上下文的过程。

> 举个例子，就好比你同事转发了一组聊天记录，并附带一个要解决的问题给你。你必须重头看一遍聊天记录，才能知道发生了什么，然后要做什么

![image.png](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/2f6c81284ef94226ae4d6344418f4e09~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5ZSQ5p-Q5Lq65Li2:q75.awebp?rk3s=f64ab15b&x-expires=1786050289&x-signature=CTyN%2F4iaavsK5WjC1ghWOHGyzTo%3D)

这里通过一个 history 字符串来演示，将过去的聊天内容都拼接到了一起

![image.png](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/b9c02ae4ffef45679fcd04903b6cc2e2~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5ZSQ5p-Q5Lq65Li2:q75.awebp?rk3s=f64ab15b&x-expires=1786050289&x-signature=TMbCDYkcjMup8aBZ4BoEpiOM5hU%3D)

然后插入到提示词中

![image.png](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/d085aa7d6d0047eea6c2e88e9ae30733~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5ZSQ5p-Q5Lq65Li2:q75.awebp?rk3s=f64ab15b&x-expires=1786050289&x-signature=LZLCvVMFhhNunlSBIkPQNyXYM4Q%3D) 再来测试一下

![image.png](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/12570c6f2a0d4b3894fa371b2a7404c8~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5ZSQ5p-Q5Lq65Li2:q75.awebp?rk3s=f64ab15b&x-expires=1786050289&x-signature=Gx7D1FqVO5%2BN0Jpy%2FE8s4tHek%2FY%3D)

虽然只是演示，但是我相信你也发现了，想让 LLM 知道过去聊了什么，就得把过去的内容一并带入给它

#### 信息有限

因为 LLM 一般都是基于公开的数据进行训练的，所以问到关于你公司内部的资料、代码，它是肯定不懂的。当你问到一个关于公司内部的组件如何使用、业务是怎么样的，它就会胡乱回答，这就是所谓的**幻觉**。

解决的办法也很简单，就是**把你公司内部的数据也塞到提示词里面**：

![image.png](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/2c7bb3aba0f041c09b0c235f94a805a8~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5ZSQ5p-Q5Lq65Li2:q75.awebp?rk3s=f64ab15b&x-expires=1786050289&x-signature=xvm%2BXX0bTxy%2BfhRhiUJ7ET98QfU%3D)

将模拟检索的外部内容，插入到提示词中： ![image.png](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/b74136a5339f4c528bba1a2ca412aa72~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5ZSQ5p-Q5Lq65Li2:q75.awebp?rk3s=f64ab15b&x-expires=1786050289&x-signature=p2IIK1Z1x8xuTYFXfChfgztC9PY%3D)

再来测试一下效果

![image.png](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/e38119d710af4875ad79a7282a94b4f2~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5ZSQ5p-Q5Lq65Li2:q75.awebp?rk3s=f64ab15b&x-expires=1786050289&x-signature=zawZT%2FLrja%2BLxDUFh31631z5Xgc%3D)

#### 小结

上面我们通过提示词的设计，已经模拟的解决了角色设定、输出规范、历史记忆、外部内容检索的问题。

但是实际开发中**用户的历史对话不是固定的**，**检索内容也是需要根据用户的问题动态查找**的，所以这就需要我们开发人员，通过技术的方案去解决这些问题。

> 上面关于提示词工程的代码演示代码在这里：[github.com/zixingtangm…](https://github.com/zixingtangmouren/ai-study-demo/tree/demo-02 "https://github.com/zixingtangmouren/ai-study-demo/tree/demo-02")

### 4.6 Memory

在提示词工程章节中，我们通过模拟历史对话，将它插入到提示词中，让 LLM 具备了对话记忆的能力。这一小节，我们就来讲解，实际的应用中是如何管理历史对话、处理历史对话的。

#### 整体流程

历史对话的整体方案流程如下

![image.png](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/fba3b3d554e540cb8e07c6100082d781~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5ZSQ5p-Q5Lq65Li2:q75.awebp?rk3s=f64ab15b&x-expires=1786050289&x-signature=O1YVX1thwKpHvRcE3bEpOl9O2sQ%3D)

核心就是四个流程：

- 实现一个消息管理队列，用于记录用户和 LLM 的对话内容
- 用户输入时，会将过去的对话总结
  - 指定轮次拼接：将指定轮次的对话，通过字符串拼接起来
  - LLM 总结：将过往的内容不断通过 LLM 进行浓缩总结
- 将对话总结插入提示词
- 提示词输入给 LLM

#### 代码实现

**HistoryMessages**

声明一个 historyMessages 记录历史的对话。每次调用对话接口时记录用户的输入，最后在 LLM 回复完毕以后，记录 LLM 的回复。

![image.png](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/7b22bf557cf64956889d5ce584baa9d5~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5ZSQ5p-Q5Lq65Li2:q75.awebp?rk3s=f64ab15b&x-expires=1786050289&x-signature=5KlaPz9yWl9T5YLqnobYEjFrKbU%3D)

接下来便是总结部分，我们需要将过往的聊天记录拼接或者总结成一段文本，在新一轮的对话中插入到提示词中。下面分别演示两种方案。

**方案 1**

将过去的固定轮次的对话，直接用字符串拼接起来

![image.png](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/8e21ae1d0ee445f1a728d5f6bf1ae616~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5ZSQ5p-Q5Lq65Li2:q75.awebp?rk3s=f64ab15b&x-expires=1786050289&x-signature=U4rifdY930aCJRcIcWM1d%2B48Mrs%3D)

插入到提示词中

![image.png](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/cd1e138fb9bc44509ee351b149599c76~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5ZSQ5p-Q5Lq65Li2:q75.awebp?rk3s=f64ab15b&x-expires=1786050289&x-signature=wsBZ%2F56gcBRZZRKs7jEbKiwVWYs%3D)

方案 1 的实现比较简单粗暴。

- 优点：会详细的保留过去固定轮次的对话细节
- 缺点：更久远的对话会丢失，这将导致 LLM 忘记很早之前聊了什么

**方案 2**：

只记录当前这轮对话。在 LLM 回复完毕后，再后置的让 LLM 总结一次过去的聊天内容 + 当前这一轮的聊天内容。

先声明一个历史总结的字符串

![image.png](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/d2f67c91715448ae88edafcccef622b8~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5ZSQ5p-Q5Lq65Li2:q75.awebp?rk3s=f64ab15b&x-expires=1786050289&x-signature=ospi6NcqmAess88ObIzKibC4J%2BE%3D)

下面函数的逻辑是，先设定一个总结摘要角色的 LLM。然后将过去的历史总结 + 新一轮的对话内容交给它，让它总结出新的历史总结，最后将 HistoryMessages 置空。

![image.png](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/1548d68f24434f80ae6d6b2be15ff054~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5ZSQ5p-Q5Lq65Li2:q75.awebp?rk3s=f64ab15b&x-expires=1786050289&x-signature=QAjmrOb8P%2FsVFVMtSrVNj8nUF1g%3D)

然后将这个函数应用在对话接口中 ![image.png](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/a8bd6f4109a645179a084c41983e7164~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5ZSQ5p-Q5Lq65Li2:q75.awebp?rk3s=f64ab15b&x-expires=1786050289&x-signature=%2B1piwvvygDpT7ronKpM%2Bj%2B4Zhes%3D)

插入对话总结到提示词

![image.png](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/7fd54ed690b941399eccf7913ea5672e~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5ZSQ5p-Q5Lq65Li2:q75.awebp?rk3s=f64ab15b&x-expires=1786050289&x-signature=v5770DrOT0mrdn37KgUl7FBYWmo%3D)

相比于方案 1，方案 2 通过 LL 总结摘要的方式，它具备以下优缺点

- 优点：让 LLM 能够知道过去很久远的聊天内容
- 缺点：但是对于过往聊天的细节会越来越模糊

#### 上下文限制

不知道你是否有思考过，上面的方案中。为什么只能将固定轮次的历史对话或者一个总结摘要后的历史对话，传入给 LLM 呢？

其实本质的原因是：LLM 的输入输出上下文 token 是有限的。大模型的上下文 **token 限制** 是指模型在每次生成或推理时，能够处理的最大**输入和输出文本片段的长度**（用 **tokens** 表示）。如果输入和输出的总长度超过这个限制，模型将无法正常处理，会报错或截断部分内容。

![image.png](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/c5bf527ac9d34dc2a89c639454e2ca64~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5ZSQ5p-Q5Lq65Li2:q75.awebp?rk3s=f64ab15b&x-expires=1786050289&x-signature=tEifcbqzBNn92oYKdMMFJ%2B%2FQYrk%3D)

总之，就是你输入的提示词内容不能太长。

> 关于 Memory 的详细代码在这里：[github.com/zixingtangm…](https://github.com/zixingtangmouren/ai-study-demo/tree/demo-03 "https://github.com/zixingtangmouren/ai-study-demo/tree/demo-03")

### 4.7 RAG

在提示词工程章节中有演示到，如果在提示词中插入外部搜索到的内容，能让 LLM 降低幻觉，做到能够回答特定领域的问题。接下来，就来介绍一下 Retrieval-Augmented Generation 这项技术。

#### 整体流程

关于 RAG 技术，整体分为两个流程：

- 数据导入：将私域的数据进行分块、向量化处理，然后将向量数据导入到向量数据库中
- 数据检索：将用户的问题向量化处理，然后去向量数据库中匹配相似或者相关语义的内容，然后插入到提示词中交给 LLM 进行分析

![image.png](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/ce609eaea5fe47b5887eca23cd292f40~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5ZSQ5p-Q5Lq65Li2:q75.awebp?rk3s=f64ab15b&x-expires=1786050289&x-signature=hqfpF2qG9%2BXW%2Ffd%2FyyrdvBBakQM%3D)

核心就是基于用户问题，在内部数据中搜索相关语义的内容，插入到提示词中让 LLM 参考回答。

#### 方案讲解

一般来说，企业内部的数据，都是通过这三种形式保存：

- 文本
- PDF
- 网站

所以我们要做的就是把这些私域的数据**向量化处理**以后，存到一个向量数据库里面。但是为什么要用这种方式进行存储呢？

![image.png](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/42591c92f5ba4a5f8cf7bf1419be4465~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5ZSQ5p-Q5Lq65Li2:q75.awebp?rk3s=f64ab15b&x-expires=1786050289&x-signature=XrrHbM%2FVN6YneWRuEAQ2BgbicOI%3D)

因为传统的全文检索或关键字匹配，依赖单词的**字面匹配**，无法理解用户输入的语义。例如：

- 用户输入“什么是人工智能？”
- 数据库中的一段文本中描述为“人工智能是一种模仿人类智能的技术。”

传统检索就无法匹配到这段文字，因为没有直接匹配的关键词。而向量数据库通过**语义搜索**，可以基于句子意思的相似性找到相关内容：

- 先将用户问题（Query）和文档中的文本转换为向量表示。
- 在向量空间里，通过向量相似度（如余弦相似度）比较，找到与用户问题语义相近的内容。

语义搜索解决了传统搜索无法捕捉“内容语义相似性”的问题，这是 RAG 技术的核心需求。

#### 代码实现

先来介绍数据导入。

这里要做的就是导入数据、拆分数据、数据向量化、存入数据库。这里我们直接使用 langchain 提供的相关工具。

需要注意是，将数据进行向量化处理，需要用到额外的 Emedding 模型。这里可以直接去火山引擎开通一个。

![image.png](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/e1ab948b92f54790a06b6805c397372a~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5ZSQ5p-Q5Lq65Li2:q75.awebp?rk3s=f64ab15b&x-expires=1786050289&x-signature=KAiLLVijbNvtwD9vEsUXhGMfv%2FA%3D)

下载 langchain 相关的包

```
bash 代码解读复制代码pnpm add @langchain/core @langchain/community @langchain/openai langchain -S
```

下载 faiss-node，这个还需要额外 build 一下

```
bash 代码解读复制代码pnpm add faiss-node@0.5.1

pnpm rebuild faiss-node
```

使用 TextLoader 导入文档

![image.png](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/4fc4532897ea4f5d9edaa111a91e9aa0~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5ZSQ5p-Q5Lq65Li2:q75.awebp?rk3s=f64ab15b&x-expires=1786050289&x-signature=qDYei5afXFHGoUsdtKFX4VpgM04%3D)

创建一个分割器将文档进行分割

![image.png](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/8a20689f62f241eeaa3bb3518095aadf~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5ZSQ5p-Q5Lq65Li2:q75.awebp?rk3s=f64ab15b&x-expires=1786050289&x-signature=NlEF%2BN3J%2Ftr9CjppgjT%2Fyt%2BsAFg%3D)

创建 Embedding 模型

![image.png](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/c36e125e2b014392b6a7bc2aa465053b~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5ZSQ5p-Q5Lq65Li2:q75.awebp?rk3s=f64ab15b&x-expires=1786050289&x-signature=XLQOrEzQf%2FbljwWmqz6KUSseCDY%3D)

批量处理数据并存入向量数据库中

![image.png](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/01334c61243b4952ad134029cd3d78bc~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5ZSQ5p-Q5Lq65Li2:q75.awebp?rk3s=f64ab15b&x-expires=1786050289&x-signature=MSQdLeYj896RiQOOSgWDU%2Bb%2Fxho%3D)

执行完毕以后，将会在项目中看到一个这样的一个文件夹

![image.png](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/196addf354d94a8a8ace8f7b529adc5c~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5ZSQ5p-Q5Lq65Li2:q75.awebp?rk3s=f64ab15b&x-expires=1786050289&x-signature=rjlBX65wLOmbTzaJTBLSk%2Ff8THU%3D)

然后再来实现数据的检索。

主要做的就是基于用户的问题在向量数据库中查询相关的内容，第一步也是需要创建一个向量模型，第二步就是加载向量数据库

![image.png](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/fc31b4cd31bf4185ac9edac586f29017~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5ZSQ5p-Q5Lq65Li2:q75.awebp?rk3s=f64ab15b&x-expires=1786050289&x-signature=Kh8Reyvm9Jb0pf5nsz2z8CvRbac%3D)

第三步就是将用户的输入进行检索匹配

![image.png](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/1c971aa807504fc1ab59c6f05e18bcb9~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5ZSQ5p-Q5Lq65Li2:q75.awebp?rk3s=f64ab15b&x-expires=1786050289&x-signature=w2EXpE87KgU%2FZ8UY0xts2jTni%2F8%3D)

第四部就是拼接内容

![image.png](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/9ac6d656be0b4623b5720385e89c59b0~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5ZSQ5p-Q5Lq65Li2:q75.awebp?rk3s=f64ab15b&x-expires=1786050289&x-signature=YHmn8DIahqRRtHe52jMzax1rtbI%3D)

最后就是基于搜到内容，插入到提示词中，让 LLM 进行参考回答。

#### 小结

到目前为止，我们通过 Memory、 RAG 的技术方案已经开发了一个具备对话记忆、能够回答特定领域内容的 AI 编程助手了，其本质就是在提示词上插入相关内容。

但是目前这个 AI 助手只能进行问答对话的，只是算是一个 Chat Bot，如果我们希望它像是长了“手脚”一样，能够调用外部的能力，该怎么做呢？这就需要用到 LLM 的 Function Calling 的能力了。

> 关于 RAG 的详细代码在这里：[github.com/zixingtangm…](https://github.com/zixingtangmouren/ai-study-demo/tree/demo-04 "https://github.com/zixingtangmouren/ai-study-demo/tree/demo-04")

### 4.8 Tools

关于 Tool，可以说是 AI 发展的又一次进阶。因为它让 LLM 具备了调用外部接口的能力。想象力再次扩大，不仅仅只是简单的对话，而是可以实际帮你执行一些任务。也正是它的出现，现在才有了 Agent、MCP 相关的概念。

#### 整体流程

![image.png](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/7a38d81a758145b4b791f4ec365d91c2~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5ZSQ5p-Q5Lq65Li2:q75.awebp?rk3s=f64ab15b&x-expires=1786050289&x-signature=S%2FExv1LyAKNj1hlWnPPNIFTfPxA%3D)

核心思路如下

- 先定义一些工具描述（告诉 LLM 这个什么时候调用）传入给LLM
- LLM 会根据你的问题，自主思考决策是否调用工具
- 如果 LLM 决策是调用工具，就会返回工具调用的指令（包含函数名、所需的调用参数）
- 然后我们根据函数名找到对应的 tool 传入参数并执行
- 再将执行结果输入给 LLM，继续推进对话

#### 代码实现

声明工具，最重要的就是 function 下面的这些参数

- name：函数名
- description：函数的描述，一般是描述什么场景，怎么使用
- parameters：函数所需的参数类型

fun 就是我们要执行的具体的函数。一般工具有两个核心场景：

- 查询外部提供给 LLM
- 执行一些操作（文件读写等）

下面声明了一个将代码写入固定文件的工具，它需要传入代码片段，然后调用 fs 将代码写入文件中。

![image.png](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/91f511639485478b9180fec3b4752d75~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5ZSQ5p-Q5Lq65Li2:q75.awebp?rk3s=f64ab15b&x-expires=1786050289&x-signature=2pwdyQRNdMPobojvWdotzmvJWYU%3D)

在调用对话接口，传入 tools 参数

![image.png](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/c97e8dbb31324623bf2fac25200bff3c~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5ZSQ5p-Q5Lq65Li2:q75.awebp?rk3s=f64ab15b&x-expires=1786050289&x-signature=XDrieI11gk1GYhP1fDAd9WzVNVM%3D)

用户输入

![image.png](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/60217cf90f504c259e8b387d40e6d831~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5ZSQ5p-Q5Lq65Li2:q75.awebp?rk3s=f64ab15b&x-expires=1786050289&x-signature=1ScywB149bqhM6%2BS7z25AEyShnA%3D)

如果 LLM 判断当前对话需要调用工具时，就会返回一个调用工具的指令。这时候我们注意看，返回的内容里面 content 变成空了，但是 tool_calls 里面有了值。第一次的返回，里面返回一个函数名。

![image.png](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/36a48f5be249446a91daceedf62eee4a~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5ZSQ5p-Q5Lq65Li2:q75.awebp?rk3s=f64ab15b&x-expires=1786050289&x-signature=iZLWdO%2BkccwSdQrZ%2FXHCrKyHPzA%3D)

再来看第二次的返回。它在 arguments 中返回了一个代码片段

![image.png](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/68384573e4c2457eace333dbe2fcaa33~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5ZSQ5p-Q5Lq65Li2:q75.awebp?rk3s=f64ab15b&x-expires=1786050289&x-signature=S1q8Mig1JG2XCibglScoxpIxcH0%3D)

所以接下来，我们要做的就是：

1.  区分LLM返回类型，如果返回的是工具调用，则需要记录具体的函数名
2.  将不断接收的 arguments 拼接起来
3.  解析 arguments，判断参数是否符合预期
4.  将 arguments 传入工具函数并执行

在 handleSseResponse 调整处理 LLM 返回的逻辑

![image.png](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/5599fb62be8c4a14acbbedb55d0ceae2~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5ZSQ5p-Q5Lq65Li2:q75.awebp?rk3s=f64ab15b&x-expires=1786050289&x-signature=irQAT3dqv1r1QeGfnPA0t1BUb%2FE%3D)

在 handleSseResponse 执行完毕后，如果是执行工具的场景，我们就要找到工具、解析参数，然后执行它拿到结果。

重要的是，一定记得把 LLM 返回的指令信息，和工具执行的结果加入到历史对话里面。因为这样，LLM 才能知道当前任务的执行状态。

![image.png](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/ca27865247dd47758d627eaa335d6372~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5ZSQ5p-Q5Lq65Li2:q75.awebp?rk3s=f64ab15b&x-expires=1786050289&x-signature=mZVn7GOJENjCs9Fr4qIjYVOeHfw%3D)

然后我们可以在 handleSseResponse 返回工具的执行状态，让用户感知执行的过程。

![image.png](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/6969284c3fb54bfb968410cdf3ea688a~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5ZSQ5p-Q5Lq65Li2:q75.awebp?rk3s=f64ab15b&x-expires=1786050289&x-signature=Smd1cPolEA%2BW26Bz%2Bzx72%2BrxQ%2BQ%3D)

有了执行结果以后，我们还需要调用一次 LLM，将当前对话上下文再次传入给它。这样做的目的是为了让 LLM 判断和决策任务的执行情况，是否再次跟用户进行沟通。

> 关于 RAG 的详细代码在这里：[github.com/zixingtangm…](https://github.com/zixingtangmouren/ai-study-demo/tree/demo-05 "https://github.com/zixingtangmouren/ai-study-demo/tree/demo-05")

## 五、重塑认知

### 5.1 关键流程

当我们再重新回顾一下上面的所有方案，你就会发现，其实整条链路就是在围绕着**提示词设计**和**在提示词插入数据** 在进行出处理，

- 提示词的组装：角色和输出规范设定、历史对话插入、检索内容插入、工具描述插入
- 工具执行：根据 LLM 指令执行工具

![image.png](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/3b659deb8dcd42a487bb5e57219e233d~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5ZSQ5p-Q5Lq65Li2:q75.awebp?rk3s=f64ab15b&x-expires=1786050289&x-signature=BZi4Ktr%2F%2FEdytloFCqGks4iVmp4%3D)

当你后续不断接触新的 AI 应用，你会发现其实大家都离不开上面这些最基本的流程和方案。当然，实际的流程，需要考虑的点也更多，更复杂。

### 5.2 分析应用

如果你有用过 Dify、Coze 这样的 AI 低代码平台，当你重新去再去看它们的编辑器界面，我相信你应该能明白为什么是这样的设计了（下面我做了一个简单的标注）

![image.png](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/b5ce97ab67644d1fa1582a773b3730df~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5ZSQ5p-Q5Lq65Li2:q75.awebp?rk3s=f64ab15b&x-expires=1786050289&x-signature=Q36talJkKZu%2BkuQYxYwL97GQ614%3D)

再比如像 Cursor 和 ChatBox，当你深入使用以后，你其实也能慢慢发现它们的门道。例如，Cursor 会有 Rule 和 Codebase Indexing 的概念。

![image.png](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/1ab107c6637e4de089467c5fce96868f~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5ZSQ5p-Q5Lq65Li2:q75.awebp?rk3s=f64ab15b&x-expires=1786050289&x-signature=hSAUK8tnzctC69Q4maBTj0BlAAU%3D)

其实本质就是，通过 Rule 在提示词中加一些输出规范、限制等。通过 Codebase Indexing 将你的代码仓库构建成一个向量数据库，根据你的问题，快速检索相关的代码。

## 六、最后

这篇文章是自己花了一周多时间慢慢写出来的，因为我发现自己会用，跟如何教会别人差别很大。不过在输出的过程中，也确实不断加深了我对相关概念的理解。

因为篇幅有限，本篇能够讲的也就这么多。本篇完全原创，如果觉得对你有帮助，点赞关注支持一下。后续我也会继续更新一些关于 AI 相关的文章。

