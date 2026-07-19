---
title: 玩转OpenClaw | OpenClaw安装与IM接入文档
url: https://www.koenli.com/38c2b10a.html
publishedTime: 2026-03-09T07:15:01.000Z
---

## [](#什么是OpenClaw "什么是OpenClaw")什么是 OpenClaw

* * *

OpenClaw（曾用名 Clawdbot/Moltbot）是一款开源的个人 AI 助手，它不仅能在本地电脑或云服务器上**自主执行任务**，还能通过 Discord、钉钉、飞书等即时通讯工具与用户交互。

[![](https://img.koenli.com/%E7%8E%A9%E8%BD%ACOpenClaw%20%7C%20OpenClaw%E5%AE%89%E8%A3%85%E4%B8%8EIM%E6%8E%A5%E5%85%A5%E6%96%87%E6%A1%A3_1.avif)](https://img.koenli.com/%E7%8E%A9%E8%BD%ACOpenClaw%20%7C%20OpenClaw%E5%AE%89%E8%A3%85%E4%B8%8EIM%E6%8E%A5%E5%85%A5%E6%96%87%E6%A1%A3_1.avif)

## [](#系统要求 "系统要求")系统要求

* * *

+   Node 22+ (OpenClaw 安装脚本会自动安装)
+   macOS、Linux（Ubuntu22.04+、CentOS8）、Windows（推荐 WSL2）
+   pnpm（只有在从源代码构建时才需要）
+   推荐配置：4 核 CPU8G 内存 50G + 硬盘（首选 SSD）

## [](#实验环境 "实验环境")实验环境

* * *

> **说明**  
> OpenClaw 作为具备持久记忆与主动执行能力的开源 AI 智能体，可调用系统资源，在缺乏管控下存在信息泄露、误操作与系统破坏等风险，**因此不建议在个人常用终端上直接安装，可以选择闲置设备或购买云服务器进行部署体验**

| 配置 | 操作系统 | 内网 IP | 公网 IP |
| --- | --- | --- | --- |
| 4 vCPUs 8GB  
超高 IO 云盘 200GB | Ubuntu 22.04 64 位 | 192.168.1.16 | 58.\*.\*.226 |

## [](#Ubuntu软件源配置 "Ubuntu软件源配置")Ubuntu 软件源配置

* * *

> **说明**  
> 本文使用的是阿里云 Ubuntu 22.04 LTS (jammy) 的源，如果你使用的是其他版本的 Ubuntu，请到[阿里云 Ubuntu 镜像站](https://developer.aliyun.com/mirror/ubuntu?spm=a2c6h.13651102.0.0.3e221b11XMFvUH)拷贝对应版本的软件源配置

<table><tbody><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br><span class="line">7</span><br><span class="line">8</span><br><span class="line">9</span><br><span class="line">10</span><br><span class="line">11</span><br><span class="line">12</span><br><span class="line">13</span><br><span class="line">14</span><br><span class="line">15</span><br><span class="line">16</span><br><span class="line">17</span><br><span class="line">18</span><br><span class="line">19</span><br><span class="line">20</span><br><span class="line">21</span><br><span class="line">22</span><br><span class="line">23</span><br><span class="line">24</span><br><span class="line">25</span><br><span class="line">26</span><br><span class="line">27</span><br></pre></td><td class="code"><pre><span class="line"><span class="hljs-comment"># 备份源文件</span></span><br><span class="line"><span class="hljs-built_in">cp</span> /etc/apt/sources.list /etc/apt/sources.list.bak</span><br><span class="line"><span class="hljs-built_in">echo</span> &gt; /etc/apt/sources.list</span><br><span class="line"></span><br><span class="line"><span class="hljs-comment"># 根据Ubuntu版本到阿里云Ubuntu镜像站拷贝对应版本的配置更新源文件</span></span><br><span class="line"><span class="hljs-built_in">cat</span> &gt; /etc/apt/sources.list &lt;&lt; <span class="hljs-string">EOF</span></span><br><span class="line"><span class="hljs-string">deb https://mirrors.aliyun.com/ubuntu/ jammy main restricted universe multiverse</span></span><br><span class="line"><span class="hljs-string">deb-src https://mirrors.aliyun.com/ubuntu/ jammy main restricted universe multiverse</span></span><br><span class="line"><span class="hljs-string"></span></span><br><span class="line"><span class="hljs-string">deb https://mirrors.aliyun.com/ubuntu/ jammy-security main restricted universe multiverse</span></span><br><span class="line"><span class="hljs-string">deb-src https://mirrors.aliyun.com/ubuntu/ jammy-security main restricted universe multiverse</span></span><br><span class="line"><span class="hljs-string"></span></span><br><span class="line"><span class="hljs-string">deb https://mirrors.aliyun.com/ubuntu/ jammy-updates main restricted universe multiverse</span></span><br><span class="line"><span class="hljs-string">deb-src https://mirrors.aliyun.com/ubuntu/ jammy-updates main restricted universe multiverse</span></span><br><span class="line"><span class="hljs-string"></span></span><br><span class="line"><span class="hljs-string"># deb https://mirrors.aliyun.com/ubuntu/ jammy-proposed main restricted universe multiverse</span></span><br><span class="line"><span class="hljs-string"># deb-src https://mirrors.aliyun.com/ubuntu/ jammy-proposed main restricted universe multiverse</span></span><br><span class="line"><span class="hljs-string"></span></span><br><span class="line"><span class="hljs-string">deb https://mirrors.aliyun.com/ubuntu/ jammy-backports main restricted universe multiverse</span></span><br><span class="line"><span class="hljs-string">deb-src https://mirrors.aliyun.com/ubuntu/ jammy-backports main restricted universe multiverse</span></span><br><span class="line"><span class="hljs-string">EOF</span></span><br><span class="line"></span><br><span class="line"><span class="hljs-comment"># 更新软件包索引</span></span><br><span class="line">apt-get update</span><br><span class="line"></span><br><span class="line"><span class="hljs-comment"># 修复所有未配置的软件包</span></span><br><span class="line">dpkg --configure -a</span><br></pre></td></tr></tbody></table>

## [](#安装OpenClaw "安装OpenClaw")安装 OpenClaw

* * *

下载 CLI，通过 npm 进行全局安装，并跳过启动引导向导。

<table><tbody><tr><td class="gutter"><pre><span class="line">1</span><br></pre></td><td class="code"><pre><span class="line">curl -fsSL https://openclaw.ai/install.sh | bash -s -- --no-onboard</span><br></pre></td></tr></tbody></table>

当出现如下输出时代表 OpenClaw 已经安装完成

<table><tbody><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br><span class="line">7</span><br></pre></td><td class="code"><pre><span class="line">......</span><br><span class="line">🦞 OpenClaw installed successfully (OpenClaw 2026.3.8 (3caab92))!</span><br><span class="line">cracks claws Alright, what are we building?</span><br><span class="line"></span><br><span class="line">· Skipping onboard (requested); run openclaw onboard later</span><br><span class="line"></span><br><span class="line">FAQ: https://docs.openclaw.ai/start/faq</span><br></pre></td></tr></tbody></table>

## [](#初始化配置向导 "初始化配置向导")初始化配置向导

* * *

执行以下命令启动初始化配置向导

<table><tbody><tr><td class="gutter"><pre><span class="line">1</span><br></pre></td><td class="code"><pre><span class="line">openclaw onboard --install-daemon</span><br></pre></td></tr></tbody></table>

> **说明**  
> 在初始化配置向导中通过上下左右方向键选择选项，通过回车键确认选项

确认已阅读安全提示

<table><tbody><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br></pre></td><td class="code"><pre><span class="line">*  I understand this is personal-by-default and shared/multi-user use requires lock-down. Continue?</span><br><span class="line">|  &gt; Yes /   No</span><br></pre></td></tr></tbody></table>

选择快速启动（之后可通过 `openclaw configure` 修改配置）

<table><tbody><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br></pre></td><td class="code"><pre><span class="line">*  Onboarding mode</span><br><span class="line">|  &gt; QuickStart (Configure details later via openclaw configure.)</span><br><span class="line">|    Manual</span><br></pre></td></tr></tbody></table>

选择 AI 模型提供商。如果要对接的模型有在内置的 Provider 中，选择对应选项按照提示填写相关信息即可；反之则到对应的模型开放平台获取 “`API base_url`”、“`API Key`”、“`Model ID`” 信息，然后选择 “`Custom Provider`” 进行自定义配置。(本文以对接 DeepSeek 为例)

| 开放平台 | 网址 |
| --- | --- |
| DeepSeek 开放平台 | [https://platform.deepseek.com/](https://platform.deepseek.com/) |
| 阿里云百炼 | [https://bailian.console.aliyun.com/cn-beijing/?tab=api#/api](https://bailian.console.aliyun.com/cn-beijing/?tab=api#/api) |

<table><tbody><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br><span class="line">7</span><br><span class="line">8</span><br><span class="line">9</span><br><span class="line">10</span><br><span class="line">11</span><br><span class="line">12</span><br><span class="line">13</span><br><span class="line">14</span><br><span class="line">15</span><br><span class="line">16</span><br><span class="line">17</span><br><span class="line">18</span><br><span class="line">19</span><br><span class="line">20</span><br><span class="line">21</span><br><span class="line">22</span><br><span class="line">23</span><br><span class="line">24</span><br><span class="line">25</span><br><span class="line">26</span><br><span class="line">27</span><br><span class="line">28</span><br><span class="line">29</span><br></pre></td><td class="code"><pre><span class="line">*  Model/auth provider</span><br><span class="line">|    OpenAI</span><br><span class="line">|    Anthropic</span><br><span class="line">|    Chutes</span><br><span class="line">|    vLLM</span><br><span class="line">|    MiniMax</span><br><span class="line">|    Moonshot AI (Kimi K2.5)</span><br><span class="line">|    Google</span><br><span class="line">|    xAI (Grok)</span><br><span class="line">|    Mistral AI</span><br><span class="line">|    Volcano Engine</span><br><span class="line">|    BytePlus</span><br><span class="line">|    OpenRouter</span><br><span class="line">|    Kilo Gateway</span><br><span class="line">|    Qwen</span><br><span class="line">|    Z.AI</span><br><span class="line">|    Qianfan</span><br><span class="line">|    Copilot</span><br><span class="line">|    Vercel AI Gateway</span><br><span class="line">|    OpenCode Zen</span><br><span class="line">|    Xiaomi</span><br><span class="line">|    Synthetic</span><br><span class="line">|    Together AI</span><br><span class="line">|    Hugging Face</span><br><span class="line">|    Venice AI</span><br><span class="line">|    LiteLLM</span><br><span class="line">|    Cloudflare AI Gateway</span><br><span class="line">|  &gt; Custom Provider (Any OpenAI or Anthropic compatible endpoint)</span><br><span class="line">|    Skip <span class="hljs-keyword">for</span> now</span><br></pre></td></tr></tbody></table>

填写 “`API base_url`”

<table><tbody><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br></pre></td><td class="code"><pre><span class="line">*  API Base URL</span><br><span class="line">|  https://api.deepseek.com/v1█</span><br></pre></td></tr></tbody></table>

选择直接粘贴 “`API Key`” 存储在 OpenClaw 配置文件中

<table><tbody><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br></pre></td><td class="code"><pre><span class="line">*  How <span class="hljs-keyword">do</span> you want to provide this API key?</span><br><span class="line">|  &gt; Paste API key now (Stores the key directly <span class="hljs-keyword">in</span> OpenClaw config)</span><br><span class="line">|    Use external secret provider</span><br></pre></td></tr></tbody></table>

输入 “`API Key`”

<table><tbody><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br></pre></td><td class="code"><pre><span class="line">*  API Key (leave blank <span class="hljs-keyword">if</span> not required)</span><br><span class="line">|  sk-e4xxxxxxxx</span><br></pre></td></tr></tbody></table>

根据对接的模型选择对应的 API 接口标准，此处选择 “`OpenAI-compatible`”

<table><tbody><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br></pre></td><td class="code"><pre><span class="line">*  Endpoint compatibility</span><br><span class="line">|  &gt; OpenAI-compatible (Uses /chat/completions)</span><br><span class="line">|    Anthropic-compatible</span><br><span class="line">|    Unknown (detect automatically)</span><br></pre></td></tr></tbody></table>

填写模型名称 “`Model ID`”

> **说明**  
> `deepseek-chat` 对应 DeepSeek-V3.2 的非思考模式，`deepseek-reasoner` 对应 DeepSeek-V3.2 的思考模式。

<table><tbody><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br></pre></td><td class="code"><pre><span class="line">*  Model ID</span><br><span class="line">|  deepseek-chat█</span><br></pre></td></tr></tbody></table>

OpenClaw 会自动对配置的模型进行接口验证，出现如下提示说明接口测试成功

<table><tbody><tr><td class="gutter"><pre><span class="line">1</span><br></pre></td><td class="code"><pre><span class="line">o  Verification successful.</span><br></pre></td></tr></tbody></table>

配置当前模型接入配置的内部名称（可自定义）

<table><tbody><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br></pre></td><td class="code"><pre><span class="line">*  Endpoint ID</span><br><span class="line">|  custom-api-deepseek-com█</span><br></pre></td></tr></tbody></table>

配置模型别名（可选）

<table><tbody><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br></pre></td><td class="code"><pre><span class="line">*  Model <span class="hljs-built_in">alias</span> (optional)</span><br><span class="line">|  DeepSeek-Chat█</span><br></pre></td></tr></tbody></table>

配置通道，OpenClaw 默认支持与多种即时通讯软件对接，但大部分都是国外平台，国内的只有一个飞书，此处先选择 “`Skip for now`”，之后再进行通道配置

<table><tbody><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br><span class="line">7</span><br><span class="line">8</span><br><span class="line">9</span><br><span class="line">10</span><br><span class="line">11</span><br><span class="line">12</span><br><span class="line">13</span><br><span class="line">14</span><br><span class="line">15</span><br><span class="line">16</span><br><span class="line">17</span><br><span class="line">18</span><br><span class="line">19</span><br><span class="line">20</span><br><span class="line">21</span><br><span class="line">22</span><br></pre></td><td class="code"><pre><span class="line">*  Select channel (QuickStart)</span><br><span class="line">|    Telegram (Bot API)</span><br><span class="line">|    WhatsApp (QR <span class="hljs-built_in">link</span>)</span><br><span class="line">|    Discord (Bot API)</span><br><span class="line">|    IRC (Server + Nick)</span><br><span class="line">|    Google Chat (Chat API)</span><br><span class="line">|    Slack (Socket Mode)</span><br><span class="line">|    Signal (signal-cli)</span><br><span class="line">|    iMessage (imsg)</span><br><span class="line">|    LINE (Messaging API)</span><br><span class="line">|    Feishu/Lark (飞书)</span><br><span class="line">|    Nostr (NIP-04 DMs)</span><br><span class="line">|    Microsoft Teams (Bot Framework)</span><br><span class="line">|    Mattermost (plugin)</span><br><span class="line">|    Nextcloud Talk (self-hosted)</span><br><span class="line">|    Matrix (plugin)</span><br><span class="line">|    BlueBubbles (macOS app)</span><br><span class="line">|    Zalo (Bot API)</span><br><span class="line">|    Zalo (Personal Account)</span><br><span class="line">|    Synology Chat (Webhook)</span><br><span class="line">|    Tlon (Urbit)</span><br><span class="line">|  &gt; Skip <span class="hljs-keyword">for</span> now (You can add channels later via `openclaw channels add`)</span><br></pre></td></tr></tbody></table>

配置实时搜索功能（先跳过）

<table><tbody><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br><span class="line">7</span><br></pre></td><td class="code"><pre><span class="line">*  Search provider</span><br><span class="line">|    Brave Search</span><br><span class="line">|    Gemini (Google Search)</span><br><span class="line">|    Grok (xAI)</span><br><span class="line">|    Kimi (Moonshot)</span><br><span class="line">|    Perplexity Search</span><br><span class="line">|  &gt; Skip <span class="hljs-keyword">for</span> now (Configure later with openclaw configure --section web)</span><br></pre></td></tr></tbody></table>

选择是否配置 OpenClaw 技能（先跳过）

<table><tbody><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br></pre></td><td class="code"><pre><span class="line">*  Configure skills now? (recommended)</span><br><span class="line">|    Yes / &gt; No</span><br></pre></td></tr></tbody></table>

选择要启用的钩子，此处建议勾选 `session-memory` 和 `command-logger`

| hooks | 功能 |
| --- | --- |
| boot-md | 启动时自动加载某些指令 / 提示词 |
| bootstrap-extra-files | 启动时自动加载额外文件 |
| command-logger | 自动记录所有通过 AI 助手执行的命令事件，生成审计日志，便于后续的操作回溯、调试排查或安全审计。 |
| session-memory | 发 `/new` 或 `/reset` 时自动保存当前对话记忆 |

<table><tbody><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br></pre></td><td class="code"><pre><span class="line">*  Enable hooks?</span><br><span class="line">|  [ ] Skip <span class="hljs-keyword">for</span> now</span><br><span class="line">|  [ ] 🚀 boot-md</span><br><span class="line">|  [ ] 📎 bootstrap-extra-files</span><br><span class="line">|  [+] 📝 command-logger (Log all <span class="hljs-built_in">command</span> events to a centralized audit file)</span><br><span class="line">|  [+] 💾 session-memory (Save session context to memory when /new or /reset <span class="hljs-built_in">command</span> is issued)</span><br></pre></td></tr></tbody></table>

选择在终端还是在网页上进行聊天测试，此处选择在终端进行测试

<table><tbody><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br></pre></td><td class="code"><pre><span class="line">*  How <span class="hljs-keyword">do</span> you want to hatch your bot?</span><br><span class="line">|  &gt; Hatch <span class="hljs-keyword">in</span> TUI (recommended)</span><br><span class="line">|    Open the Web UI</span><br><span class="line">|    Do this later</span><br></pre></td></tr></tbody></table>

如下图，在 TUI 界面输入 “`你现在是什么模型`”，若返回的是你上面配置的模型信息，即为配置成功。

> **说明**  
> 按两次 “`Ctrl+C`” 可退出 TUI 测试界面

[![](https://img.koenli.com/%E7%8E%A9%E8%BD%ACOpenClaw%20%7C%20OpenClaw%E5%AE%89%E8%A3%85%E4%B8%8EIM%E6%8E%A5%E5%85%A5%E6%96%87%E6%A1%A3_2.png)](https://img.koenli.com/%E7%8E%A9%E8%BD%ACOpenClaw%20%7C%20OpenClaw%E5%AE%89%E8%A3%85%E4%B8%8EIM%E6%8E%A5%E5%85%A5%E6%96%87%E6%A1%A3_2.png)

## [](#访问OpenClaw-Gateway-Dashboard "访问OpenClaw Gateway Dashboard")访问 OpenClaw Gateway Dashboard

* * *

执行以下命令获取 Dashboard 访问地址，地址格式如：`http://IP:PORT/#token=xxxxx`，然后通过浏览器访问。

<table><tbody><tr><td class="gutter"><pre><span class="line">1</span><br></pre></td><td class="code"><pre><span class="line">openclaw dashboard</span><br></pre></td></tr></tbody></table>

[![](https://img.koenli.com/%E7%8E%A9%E8%BD%ACOpenClaw%20%7C%20OpenClaw%E5%AE%89%E8%A3%85%E4%B8%8EIM%E6%8E%A5%E5%85%A5%E6%96%87%E6%A1%A3_5.png)](https://img.koenli.com/%E7%8E%A9%E8%BD%ACOpenClaw%20%7C%20OpenClaw%E5%AE%89%E8%A3%85%E4%B8%8EIM%E6%8E%A5%E5%85%A5%E6%96%87%E6%A1%A3_5.png)

> **说明**  
> 出于安全考虑，Dashboard 监听在环回接口 `127.0.0.1` 上，默认只允许在部署 OpenClaw 的机器上访问。如果你的 OpenClaw 是部署在云服务器上，可以先通过以下几种方式建立隧道转发后再在你本地主机进行访问
> 
> +   方式一  
>     本地主机如果是 MacOS/Linux，可以直接打开终端执行以下命令建议隧道转发
>     
>     <table><tbody><tr><td class="gutter"><pre><span class="line">1</span><br></pre></td><td class="code"><pre><span class="line">ssh -N -L 18789:127.0.0.1:18789 root@&lt;服务器公网IP&gt;</span><br></pre></td></tr></tbody></table>
>     
> +   方式二  
>     在 SecureCRT 中配置隧道转发  
>     [![](https://img.koenli.com/%E7%8E%A9%E8%BD%ACOpenClaw%20%7C%20OpenClaw%E5%AE%89%E8%A3%85%E4%B8%8EIM%E6%8E%A5%E5%85%A5%E6%96%87%E6%A1%A3_3.png)](https://img.koenli.com/%E7%8E%A9%E8%BD%ACOpenClaw%20%7C%20OpenClaw%E5%AE%89%E8%A3%85%E4%B8%8EIM%E6%8E%A5%E5%85%A5%E6%96%87%E6%A1%A3_3.png)
> +   方式三  
>     在 XShell 中配置隧道转发  
>     [![](https://img.koenli.com/%E7%8E%A9%E8%BD%ACOpenClaw%20%7C%20OpenClaw%E5%AE%89%E8%A3%85%E4%B8%8EIM%E6%8E%A5%E5%85%A5%E6%96%87%E6%A1%A3_4.png)](https://img.koenli.com/%E7%8E%A9%E8%BD%ACOpenClaw%20%7C%20OpenClaw%E5%AE%89%E8%A3%85%E4%B8%8EIM%E6%8E%A5%E5%85%A5%E6%96%87%E6%A1%A3_4.png)

如果觉得自动生成的 Token 过长不好记，也可以通过以下命令自定义 Token 并重启网关生效

<table><tbody><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br></pre></td><td class="code"><pre><span class="line">openclaw config <span class="hljs-built_in">set</span> gateway.auth.token <span class="hljs-string">"&lt;自定义Token&gt;"</span></span><br><span class="line">openclaw gateway restart</span><br><span class="line">openclaw dashboard</span><br></pre></td></tr></tbody></table>

## [](#开启工具执行权限 "开启工具执行权限")开启工具执行权限

* * *

在最近的 OpenClaw 2026.3.2 版本更新中，OpenClaw 默认的权限策略发生了变化：

+   默认情况下，Agent 只允许进行纯对话
+   涉及调用 Skills 工具外部接口的动作，会受到更严格的权限控制
+   当某个操作被当前权限策略禁止时，你会在对话中看到类似提示：
    +   “我没有权限执行此操作”
    +   或其他表示 “当前不允许调用技能 / 工具” 的说明

那么即便模型正常、通道正常、Skills 已安装，机器人也可能因为默认权限收紧而拒绝帮你调用工具。

因此如果你安装的 `OpenClaw 2026.3.2` 之后的版本，为了恢复 Agent 在对话中调用 Skills / 工具的能力，需要执行以下命令将工具执行权限调整为完整模式

<table><tbody><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br></pre></td><td class="code"><pre><span class="line">openclaw config <span class="hljs-built_in">set</span> tools.profile full</span><br><span class="line">openclaw gateway restart</span><br><span class="line">openclaw config get tools.profile</span><br></pre></td></tr></tbody></table>

## [](#对接即时通讯-IM-软件 "对接即时通讯(IM)软件")对接即时通讯 (IM) 软件

* * *

## [](#对接QQ机器人 "对接QQ机器人")对接 QQ 机器人

> **说明**  
> 官方插件仍处于不断更新迭代过程中，安装方法也在不断优化，如果发现安装步骤与本文介绍有出入建议查看[官方对接文档](https://docs.qq.com/doc/DS2FmdkJZZEJJWEFF)获取最新对接指南

前往腾讯 [QQ 开放平台](https://q.qq.com/qqbot/openclaw/login.html)官网，使用手机 QQ 扫描图中二维码注册登录

> **说明**  
> 如果当前尚未注册 QQ 开放平台，执行扫码操作后，系统将自动完成 QQ 开放平台注册流程，并将扫码所用的 QQ 账号与该平台账号进行绑定。

[![](https://img.koenli.com/%E7%8E%A9%E8%BD%ACOpenClaw%20%7C%20OpenClaw%E5%AE%89%E8%A3%85%E4%B8%8EIM%E6%8E%A5%E5%85%A5%E6%96%87%E6%A1%A3_6.png)](https://img.koenli.com/%E7%8E%A9%E8%BD%ACOpenClaw%20%7C%20OpenClaw%E5%AE%89%E8%A3%85%E4%B8%8EIM%E6%8E%A5%E5%85%A5%E6%96%87%E6%A1%A3_6.png)

在 QQ 开放平台的 QQ 机器人页面，点击 “`创建机器人`”，即可直接新建一个 QQ 机器人。

[![](https://img.koenli.com/%E7%8E%A9%E8%BD%ACOpenClaw%20%7C%20OpenClaw%E5%AE%89%E8%A3%85%E4%B8%8EIM%E6%8E%A5%E5%85%A5%E6%96%87%E6%A1%A3_7.png)](https://img.koenli.com/%E7%8E%A9%E8%BD%ACOpenClaw%20%7C%20OpenClaw%E5%AE%89%E8%A3%85%E4%B8%8EIM%E6%8E%A5%E5%85%A5%E6%96%87%E6%A1%A3_7.png)

机器人创建完成后，在页面中找到 “`AppID`” 和 “`AppSecret`” 两个参数，分别点击右侧 “`复制`” 按钮，将其保存到个人记事本或备忘录中，后续步骤中需要使用。

> **说明**  
> 出于安全考虑，AppSecret 不支持明文保存，二次查看将会强制重置，请自行妥善保存。

[![](https://img.koenli.com/%E7%8E%A9%E8%BD%ACOpenClaw%20%7C%20OpenClaw%E5%AE%89%E8%A3%85%E4%B8%8EIM%E6%8E%A5%E5%85%A5%E6%96%87%E6%A1%A3_8.png)](https://img.koenli.com/%E7%8E%A9%E8%BD%ACOpenClaw%20%7C%20OpenClaw%E5%AE%89%E8%A3%85%E4%B8%8EIM%E6%8E%A5%E5%85%A5%E6%96%87%E6%A1%A3_8.png)

回到 QQ 软件，可以看到新建的 QQ 机器人已经被添加至消息列表中，并向你发送了第一条消息。但此时还不能与机器人正常进行对话，会提示 “该机器人去火星了，稍后再试吧”，因为 QQ 机器人此时尚未与之前部署的 OpenClaw 打通，需要继续后面的配置步骤

[![](https://img.koenli.com/%E7%8E%A9%E8%BD%ACOpenClaw%20%7C%20OpenClaw%E5%AE%89%E8%A3%85%E4%B8%8EIM%E6%8E%A5%E5%85%A5%E6%96%87%E6%A1%A3_9.jpg)](https://img.koenli.com/%E7%8E%A9%E8%BD%ACOpenClaw%20%7C%20OpenClaw%E5%AE%89%E8%A3%85%E4%B8%8EIM%E6%8E%A5%E5%85%A5%E6%96%87%E6%A1%A3_9.jpg)

在运行 OpenClaw 的设备上执行以下命令为 OpenClaw 应用配置 QQ 机器人的 `AppID` 和 `AppSecret`

> **说明**  
> 如果安装 QQBot 插件失败，可以执行 `npm config set registry https://registry.npmmirror.com` 将 npm 源切换到淘宝源再安装

<table><tbody><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br></pre></td><td class="code"><pre><span class="line"><span class="hljs-comment"># 安装openclaw-qqbot最新版本</span></span><br><span class="line">openclaw plugins install @tencent-connect/openclaw-qqbot@latest</span><br><span class="line"><span class="hljs-comment"># 配置绑定当前QQ机器人</span></span><br><span class="line">openclaw channels add --channel qqbot --token <span class="hljs-string">"&lt;AppID&gt;:&lt;AppSecret&gt;"</span></span><br><span class="line"><span class="hljs-comment"># 重启OpenClaw服务</span></span><br><span class="line">openclaw gateway restart</span><br></pre></td></tr></tbody></table>

配置完成之后尝试和 QQ 机器人进行聊天，如果 QQ 机器人能够正常回答，说明已经成功完成 OpenClaw 接入 QQ 机器人了。

## [](#对接企业微信机器人 "对接企业微信机器人")对接企业微信机器人

> **说明**  
> 官方插件仍处于不断更新迭代过程中，安装方法也在不断优化，如果发现安装步骤与本文介绍有出入建议查看[官方对接文档](https://open.work.weixin.qq.com/help2/pc/cat?doc_id=21657)获取最新对接指南

> **说明**  
> 出于安全考虑防止企业敏感数据泄露，如果需要对接到企业微信机器人的话，建议优先选择自建企业进行体验

### [](#创建企业 "创建企业")创建企业

按照[企业微信官方文档](https://open.work.weixin.qq.com/help2/pc/15422#1%E3%80%81%E5%88%9B%E5%BB%BA%E4%BC%81%E4%B8%9A)完成企业创建

> **说明**  
> 一定要创建企业，不要选择个人组建团队，因为个人组建团队是没有工作台功能的。

### [](#以长连接方式创建智能机器人，获取Bot-ID和Secret "以长连接方式创建智能机器人，获取Bot ID和Secret")以长连接方式创建智能机器人，获取 Bot ID 和 Secret

> **说明**  
> 通过长连接方式创建的智能机器人，支持主动向用户发送消息。

在客户端 - 工作台，点击 “`智能机器人`”

[![](https://img.koenli.com/%E7%8E%A9%E8%BD%ACOpenClaw%20%7C%20OpenClaw%E5%AE%89%E8%A3%85%E4%B8%8EIM%E6%8E%A5%E5%85%A5%E6%96%87%E6%A1%A3_10.png)](https://img.koenli.com/%E7%8E%A9%E8%BD%ACOpenClaw%20%7C%20OpenClaw%E5%AE%89%E8%A3%85%E4%B8%8EIM%E6%8E%A5%E5%85%A5%E6%96%87%E6%A1%A3_10.png)

点击 “`创建机器人`”，选择 “`手动创建`”

[![](https://img.koenli.com/%E7%8E%A9%E8%BD%ACOpenClaw%20%7C%20OpenClaw%E5%AE%89%E8%A3%85%E4%B8%8EIM%E6%8E%A5%E5%85%A5%E6%96%87%E6%A1%A3_11.png)](https://img.koenli.com/%E7%8E%A9%E8%BD%ACOpenClaw%20%7C%20OpenClaw%E5%AE%89%E8%A3%85%E4%B8%8EIM%E6%8E%A5%E5%85%A5%E6%96%87%E6%A1%A3_11.png)

填写机器人信息（包含头像、名称、简介），设置可见范围，然后下拉到底部，点击 “`API模式创建`”

[![](https://img.koenli.com/%E7%8E%A9%E8%BD%ACOpenClaw%20%7C%20OpenClaw%E5%AE%89%E8%A3%85%E4%B8%8EIM%E6%8E%A5%E5%85%A5%E6%96%87%E6%A1%A3_12.png)](https://img.koenli.com/%E7%8E%A9%E8%BD%ACOpenClaw%20%7C%20OpenClaw%E5%AE%89%E8%A3%85%E4%B8%8EIM%E6%8E%A5%E5%85%A5%E6%96%87%E6%A1%A3_12.png)

选择 “`使用长链接`” 方式创建，并获取 `Bot ID` 和 `Secret` 信息并记录，后续步骤需要用到

[![](https://img.koenli.com/%E7%8E%A9%E8%BD%ACOpenClaw%20%7C%20OpenClaw%E5%AE%89%E8%A3%85%E4%B8%8EIM%E6%8E%A5%E5%85%A5%E6%96%87%E6%A1%A3_13.png)](https://img.koenli.com/%E7%8E%A9%E8%BD%ACOpenClaw%20%7C%20OpenClaw%E5%AE%89%E8%A3%85%E4%B8%8EIM%E6%8E%A5%E5%85%A5%E6%96%87%E6%A1%A3_13.png)

### [](#将OpenClaw关联到企业微信机器人 "将OpenClaw关联到企业微信机器人")将 OpenClaw 关联到企业微信机器人

在运行 OpenClaw 的设备上执行以下命令安装企业微信插件

<table><tbody><tr><td class="gutter"><pre><span class="line">1</span><br></pre></td><td class="code"><pre><span class="line">openclaw plugins install @wecom/wecom-openclaw-plugin</span><br></pre></td></tr></tbody></table>

重启 OpenClaw 服务

<table><tbody><tr><td class="gutter"><pre><span class="line">1</span><br></pre></td><td class="code"><pre><span class="line">openclaw gateway restart</span><br></pre></td></tr></tbody></table>

执行以下命令，添加渠道

<table><tbody><tr><td class="gutter"><pre><span class="line">1</span><br></pre></td><td class="code"><pre><span class="line">openclaw channels add</span><br></pre></td></tr></tbody></table>

选择 “`Yes`”

<table><tbody><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br></pre></td><td class="code"><pre><span class="line">*  Configure chat channels now?</span><br><span class="line">|  &gt; Yes /   No</span><br></pre></td></tr></tbody></table>

选择 “`企业微信 (WeCom)`”

<table><tbody><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br><span class="line">7</span><br><span class="line">8</span><br><span class="line">9</span><br><span class="line">10</span><br><span class="line">11</span><br><span class="line">12</span><br><span class="line">13</span><br><span class="line">14</span><br><span class="line">15</span><br><span class="line">16</span><br><span class="line">17</span><br><span class="line">18</span><br><span class="line">19</span><br><span class="line">20</span><br><span class="line">21</span><br><span class="line">22</span><br><span class="line">23</span><br><span class="line">24</span><br></pre></td><td class="code"><pre><span class="line">*  Select a channel</span><br><span class="line">|    Telegram (Bot API)</span><br><span class="line">|    WhatsApp (QR <span class="hljs-built_in">link</span>)</span><br><span class="line">|    Discord (Bot API)</span><br><span class="line">|    IRC (Server + Nick)</span><br><span class="line">|    Google Chat (Chat API)</span><br><span class="line">|    Slack (Socket Mode)</span><br><span class="line">|    Signal (signal-cli)</span><br><span class="line">|    iMessage (imsg)</span><br><span class="line">|    LINE (Messaging API)</span><br><span class="line">|    QQ Bot</span><br><span class="line">|  &gt; 企业微信 (WeCom) (需要设置 · disabled)</span><br><span class="line">|    Feishu/Lark (飞书)</span><br><span class="line">|    Nostr (NIP-04 DMs)</span><br><span class="line">|    Microsoft Teams (Bot Framework)</span><br><span class="line">|    Mattermost (plugin)</span><br><span class="line">|    Nextcloud Talk (self-hosted)</span><br><span class="line">|    Matrix (plugin)</span><br><span class="line">|    BlueBubbles (macOS app)</span><br><span class="line">|    Zalo (Bot API)</span><br><span class="line">|    Zalo (Personal Account)</span><br><span class="line">|    Synology Chat (Webhook)</span><br><span class="line">|    Tlon (Urbit)</span><br><span class="line">|    Finished</span><br></pre></td></tr></tbody></table>

输入上面获取到的企业微信机器人 `Bot ID` 和 `Secret`

<table><tbody><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br></pre></td><td class="code"><pre><span class="line">o  企业微信机器人 Bot ID</span><br><span class="line">|  aibCxxxxxxxxxxxxxYjeq</span><br><span class="line">|</span><br><span class="line">o  企业微信机器人 Secret</span><br><span class="line">|  PSLxxxxxxxxxxxxxxxxxxxxxxxxxxFAOp█</span><br><span class="line">—</span><br></pre></td></tr></tbody></table>

选择 “`finish`”

<table><tbody><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br><span class="line">7</span><br><span class="line">8</span><br><span class="line">9</span><br><span class="line">10</span><br><span class="line">11</span><br><span class="line">12</span><br><span class="line">13</span><br><span class="line">14</span><br><span class="line">15</span><br><span class="line">16</span><br><span class="line">17</span><br><span class="line">18</span><br><span class="line">19</span><br><span class="line">20</span><br><span class="line">21</span><br><span class="line">22</span><br><span class="line">23</span><br><span class="line">24</span><br></pre></td><td class="code"><pre><span class="line">*  Select a channel</span><br><span class="line">|    Telegram (Bot API)</span><br><span class="line">|    WhatsApp (QR <span class="hljs-built_in">link</span>)</span><br><span class="line">|    Discord (Bot API)</span><br><span class="line">|    IRC (Server + Nick)</span><br><span class="line">|    Google Chat (Chat API)</span><br><span class="line">|    Slack (Socket Mode)</span><br><span class="line">|    Signal (signal-cli)</span><br><span class="line">|    iMessage (imsg)</span><br><span class="line">|    LINE (Messaging API)</span><br><span class="line">|    QQ Bot</span><br><span class="line">|    企业微信 (WeCom)</span><br><span class="line">|    Feishu/Lark (飞书)</span><br><span class="line">|    Nostr (NIP-04 DMs)</span><br><span class="line">|    Microsoft Teams (Bot Framework)</span><br><span class="line">|    Mattermost (plugin)</span><br><span class="line">|    Nextcloud Talk (self-hosted)</span><br><span class="line">|    Matrix (plugin)</span><br><span class="line">|    BlueBubbles (macOS app)</span><br><span class="line">|    Zalo (Bot API)</span><br><span class="line">|    Zalo (Personal Account)</span><br><span class="line">|    Synology Chat (Webhook)</span><br><span class="line">|    Tlon (Urbit)</span><br><span class="line">|  &gt; Finished (Done)</span><br></pre></td></tr></tbody></table>

选择”`Yes`“配置配对方式，然后选择 “`Pairing`”

<table><tbody><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br><span class="line">7</span><br><span class="line">8</span><br><span class="line">9</span><br><span class="line">10</span><br><span class="line">11</span><br><span class="line">12</span><br><span class="line">13</span><br><span class="line">14</span><br><span class="line">15</span><br><span class="line">16</span><br><span class="line">17</span><br><span class="line">18</span><br><span class="line">19</span><br><span class="line">20</span><br><span class="line">21</span><br></pre></td><td class="code"><pre><span class="line">o  Configure DM access policies now? (default: pairing)</span><br><span class="line">|  Yes</span><br><span class="line">|</span><br><span class="line">o  企业微信 DM access ---------------------------------------------------------------------+</span><br><span class="line">|                                                                                          |</span><br><span class="line">|  Default: pairing (unknown DMs get a pairing code).                                      |</span><br><span class="line">|  Approve: openclaw pairing approve wecom &lt;code&gt;                                          |</span><br><span class="line">|  Allowlist DMs: channels.wecom.dmPolicy=<span class="hljs-string">"allowlist"</span> + channels.wecom.allowFrom entries.  |</span><br><span class="line">|  Public DMs: channels.wecom.dmPolicy=<span class="hljs-string">"open"</span> + channels.wecom.allowFrom includes <span class="hljs-string">"*"</span>.     |</span><br><span class="line">|  Multi-user DMs: run: openclaw config <span class="hljs-built_in">set</span> session.dmScope <span class="hljs-string">"per-channel-peer"</span> (or         |</span><br><span class="line">|  <span class="hljs-string">"per-account-channel-peer"</span> <span class="hljs-keyword">for</span> multi-account channels) to isolate sessions.             |</span><br><span class="line">|  Docs: channels/pairing             |</span><br><span class="line">|                                                                                          |</span><br><span class="line">+------------------------------------------------------------------------------------------+</span><br><span class="line">|</span><br><span class="line">*  企业微信 DM policy</span><br><span class="line">|  &gt; Pairing (recommended)</span><br><span class="line">|    Allowlist (specific <span class="hljs-built_in">users</span> only)</span><br><span class="line">|    Open (public inbound DMs)</span><br><span class="line">|    Disabled (ignore DMs)</span><br><span class="line">—</span><br></pre></td></tr></tbody></table>

按照提示完成后续配置

<table><tbody><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br><span class="line">7</span><br><span class="line">8</span><br></pre></td><td class="code"><pre><span class="line">o  Add display names <span class="hljs-keyword">for</span> these accounts? (optional)</span><br><span class="line">|  Yes</span><br><span class="line">|</span><br><span class="line">o  wecom account name (default)</span><br><span class="line">|  企业微信</span><br><span class="line">Config overwrite: /root/.openclaw/openclaw.json (sha256 df4583de8dd0c8fd1a7f7f4959ebeae3a831c973e1283dd20713379bdcbc6158 -&gt; 3eb746c7b36bb73343974e7809b64c63d6b1bd01233812de35d6184b779e562c, backup=/root/.openclaw/openclaw.json.bak)</span><br><span class="line">|</span><br><span class="line">—  Channels updated.</span><br></pre></td></tr></tbody></table>

回到企业微信工作台中，点击 “`保存`” 完成机器人创建

[![](https://img.koenli.com/%E7%8E%A9%E8%BD%ACOpenClaw%20%7C%20OpenClaw%E5%AE%89%E8%A3%85%E4%B8%8EIM%E6%8E%A5%E5%85%A5%E6%96%87%E6%A1%A3_14.png)](https://img.koenli.com/%E7%8E%A9%E8%BD%ACOpenClaw%20%7C%20OpenClaw%E5%AE%89%E8%A3%85%E4%B8%8EIM%E6%8E%A5%E5%85%A5%E6%96%87%E6%A1%A3_14.png)

然后跟机器人任意发一条消息，比如 “`你好`”，机器人会回复一个配置密钥信息

[![](https://img.koenli.com/%E7%8E%A9%E8%BD%ACOpenClaw%20%7C%20OpenClaw%E5%AE%89%E8%A3%85%E4%B8%8EIM%E6%8E%A5%E5%85%A5%E6%96%87%E6%A1%A3_15.png)](https://img.koenli.com/%E7%8E%A9%E8%BD%ACOpenClaw%20%7C%20OpenClaw%E5%AE%89%E8%A3%85%E4%B8%8EIM%E6%8E%A5%E5%85%A5%E6%96%87%E6%A1%A3_15.png)

复制此信息最后一行，并到终端上执行完成配对。

<table><tbody><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br><span class="line">7</span><br></pre></td><td class="code"><pre><span class="line">root@openclaw:~<span class="hljs-comment"># openclaw pairing approve wecom RJSHVK7E</span></span><br><span class="line">11:23:44 [plugins] plugins.allow is empty; discovered non-bundled plugins may auto-load: qqbot (/root/.openclaw/extensions/qqbot/index.ts), wecom-openclaw-plugin (/root/.openclaw/extensions/wecom-openclaw-plugin/dist/index.esm.js). Set plugins.allow to explicit trusted ids.</span><br><span class="line"></span><br><span class="line">🦞 OpenClaw 2026.3.8 (3caab92) — Built by lobsters, <span class="hljs-keyword">for</span> humans. Don<span class="hljs-string">'t question the hierarchy.</span></span><br><span class="line"><span class="hljs-string"></span></span><br><span class="line"><span class="hljs-string">Approved wecom sender LiMingXian.</span></span><br><span class="line"><span class="hljs-string">root@openclaw:~# </span></span><br></pre></td></tr></tbody></table>

此时就可在企业微信中正常对话了。

## [](#对接钉钉机器人 "对接钉钉机器人")对接钉钉机器人

> **说明**  
> 官方插件仍处于不断更新迭代过程中，安装方法也在不断优化，如果发现安装步骤与本文介绍有出入建议查看[官方对接文档](https://open.dingtalk.com/document/dingstart/build-dingtalk-ai-employees)获取最新对接指南

> **说明**  
> 出于安全考虑防止企业敏感数据泄露，如果需要对接到钉钉机器人的话，建议优先选择自建企业进行体验

### [](#创建企业-1 "创建企业")创建企业

打开手机钉钉 App，点击左上角头像，选择 “`创建/加入新的企业团队`”，使用场景选择 “`企业`”-“`创建企业/团队`”，按照提示完成信息填写即可。

### [](#一键创建OpenClaw机器人 "一键创建OpenClaw机器人")一键创建 OpenClaw 机器人

登录[钉钉开发者后台](https://open.dingtalk.com/)，组织选择上一步自建的企业

[![](https://img.koenli.com/%E7%8E%A9%E8%BD%ACOpenClaw%20%7C%20OpenClaw%E5%AE%89%E8%A3%85%E4%B8%8EIM%E6%8E%A5%E5%85%A5%E6%96%87%E6%A1%A3_16.png)](https://img.koenli.com/%E7%8E%A9%E8%BD%ACOpenClaw%20%7C%20OpenClaw%E5%AE%89%E8%A3%85%E4%B8%8EIM%E6%8E%A5%E5%85%A5%E6%96%87%E6%A1%A3_16.png)

在应用开发 - 钉钉应用下，点击 “`立即创建`”，一键创建 OpenClaw 机器人

[![](https://img.koenli.com/%E7%8E%A9%E8%BD%ACOpenClaw%20%7C%20OpenClaw%E5%AE%89%E8%A3%85%E4%B8%8EIM%E6%8E%A5%E5%85%A5%E6%96%87%E6%A1%A3_17.png)](https://img.koenli.com/%E7%8E%A9%E8%BD%ACOpenClaw%20%7C%20OpenClaw%E5%AE%89%E8%A3%85%E4%B8%8EIM%E6%8E%A5%E5%85%A5%E6%96%87%E6%A1%A3_17.png)

在创建 OpenClaw 机器人界面，填写机器人基本信息（包括机器人名称、机器人简介和机器人图标），最后点击 “`确定`”

[![](https://img.koenli.com/%E7%8E%A9%E8%BD%ACOpenClaw%20%7C%20OpenClaw%E5%AE%89%E8%A3%85%E4%B8%8EIM%E6%8E%A5%E5%85%A5%E6%96%87%E6%A1%A3_18.png)](https://img.koenli.com/%E7%8E%A9%E8%BD%ACOpenClaw%20%7C%20OpenClaw%E5%AE%89%E8%A3%85%E4%B8%8EIM%E6%8E%A5%E5%85%A5%E6%96%87%E6%A1%A3_18.png)

OpenClaw 机器人创建成功后，会自动展示应用的 `Client ID` 和 `Client Secret`，复制保存好用于后续使用。

> **说明**
> 
> +   `Client ID` 和 `Client Secret` 是应用的关键信息，也是操作应用数据的核心参数，请妥善保管，切勿轻易提供给他人使用。
> +   关闭窗口后仍可在应用「凭证与基础信息」中查看

[![](https://img.koenli.com/%E7%8E%A9%E8%BD%ACOpenClaw%20%7C%20OpenClaw%E5%AE%89%E8%A3%85%E4%B8%8EIM%E6%8E%A5%E5%85%A5%E6%96%87%E6%A1%A3_19.png)](https://img.koenli.com/%E7%8E%A9%E8%BD%ACOpenClaw%20%7C%20OpenClaw%E5%AE%89%E8%A3%85%E4%B8%8EIM%E6%8E%A5%E5%85%A5%E6%96%87%E6%A1%A3_19.png)

> **说明**  
> 自动创建的 OpenClaw 机器人会默认开通 `Card.Streaming.Write`、`Card.Instance.Write` 和 `qyapi_robot_sendmsg` 权限，开发者无需再手动申请。

### [](#将OpenClaw关联到钉钉机器人 "将OpenClaw关联到钉钉机器人")将 OpenClaw 关联到钉钉机器人

在运行 OpenClaw 的设备上执行以下命令安装钉钉官方插件

<table><tbody><tr><td class="gutter"><pre><span class="line">1</span><br></pre></td><td class="code"><pre><span class="line">openclaw plugins install @dingtalk-real-ai/dingtalk-connector</span><br></pre></td></tr></tbody></table>

执行 `vim ~/.openclaw/openclaw.json` 进入文件编辑，在 `~/.openclaw/openclaw.json` 文件中添加 `channels/dingtalk-connector`、`gateway/auth` 和 `gateway/http/endpoints`3 个配置项属性

> **说明**  
> 下方已省略其他配置项，只提供了核心配置项及需要配置钉钉的相关属性内容。

<table><tbody><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br><span class="line">7</span><br><span class="line">8</span><br><span class="line">9</span><br><span class="line">10</span><br><span class="line">11</span><br><span class="line">12</span><br><span class="line">13</span><br><span class="line">14</span><br><span class="line">15</span><br><span class="line">16</span><br><span class="line">17</span><br><span class="line">18</span><br><span class="line">19</span><br><span class="line">20</span><br><span class="line">21</span><br><span class="line">22</span><br><span class="line">23</span><br><span class="line">24</span><br></pre></td><td class="code"><pre><span class="line"><span class="hljs-punctuation">{</span></span><br><span class="line">  <span class="hljs-attr">"channels"</span><span class="hljs-punctuation">:</span> <span class="hljs-punctuation">{</span></span><br><span class="line">    <span class="hljs-attr">"dingtalk-connector"</span><span class="hljs-punctuation">:</span> <span class="hljs-punctuation">{</span></span><br><span class="line">      <span class="hljs-attr">"clientId"</span><span class="hljs-punctuation">:</span> <span class="hljs-string">"钉钉应用的Client ID"</span><span class="hljs-punctuation">,</span>       <span class="hljs-comment">// 必选：填入上方的 钉钉 Client ID</span></span><br><span class="line">      <span class="hljs-attr">"clientSecret"</span><span class="hljs-punctuation">:</span> <span class="hljs-string">"钉钉应用的Client Secret"</span><span class="hljs-punctuation">,</span> <span class="hljs-comment">// 必选：填入上方的 Client Secret</span></span><br><span class="line">      <span class="hljs-attr">"gatewayToken"</span><span class="hljs-punctuation">:</span> <span class="hljs-string">"Gateway 认证 token"</span><span class="hljs-punctuation">,</span>  <span class="hljs-comment">// 必选：Gateway 认证 token, openclaw.json配置中 gateway.auth.token 的值 </span></span><br><span class="line">      <span class="hljs-attr">"gatewayPassword"</span><span class="hljs-punctuation">:</span> <span class="hljs-string">""</span><span class="hljs-punctuation">,</span>              <span class="hljs-comment">// 可选：Gateway 认证 password（与 token 二选一）</span></span><br><span class="line">      <span class="hljs-attr">"sessionTimeout"</span><span class="hljs-punctuation">:</span> <span class="hljs-number">1800000</span>           <span class="hljs-comment">// 可选：会话超时(ms)，默认 30 分钟</span></span><br><span class="line">    <span class="hljs-punctuation">}</span></span><br><span class="line">  <span class="hljs-punctuation">}</span><span class="hljs-punctuation">,</span></span><br><span class="line">  <span class="hljs-attr">"gateway"</span><span class="hljs-punctuation">:</span> <span class="hljs-punctuation">{</span> <span class="hljs-comment">// gateway通常是已有的节点，配置时注意把http部分追加到已有节点下</span></span><br><span class="line">    <span class="hljs-attr">"auth"</span><span class="hljs-punctuation">:</span> <span class="hljs-punctuation">{</span></span><br><span class="line">      <span class="hljs-attr">"mode"</span><span class="hljs-punctuation">:</span> <span class="hljs-string">"token"</span><span class="hljs-punctuation">,</span></span><br><span class="line">      <span class="hljs-attr">"token"</span><span class="hljs-punctuation">:</span> <span class="hljs-string">"Gateway 认证 token"</span> <span class="hljs-comment">// 必选：一般是安装时默认就有</span></span><br><span class="line">    <span class="hljs-punctuation">}</span><span class="hljs-punctuation">,</span></span><br><span class="line">    <span class="hljs-attr">"http"</span><span class="hljs-punctuation">:</span> <span class="hljs-punctuation">{</span></span><br><span class="line">      <span class="hljs-attr">"endpoints"</span><span class="hljs-punctuation">:</span> <span class="hljs-punctuation">{</span></span><br><span class="line">        <span class="hljs-attr">"chatCompletions"</span><span class="hljs-punctuation">:</span> <span class="hljs-punctuation">{</span></span><br><span class="line">          <span class="hljs-attr">"enabled"</span><span class="hljs-punctuation">:</span> <span class="hljs-literal"><span class="hljs-keyword">true</span></span> <span class="hljs-comment">// 必选</span></span><br><span class="line">        <span class="hljs-punctuation">}</span></span><br><span class="line">      <span class="hljs-punctuation">}</span></span><br><span class="line">    <span class="hljs-punctuation">}</span></span><br><span class="line">  <span class="hljs-punctuation">}</span></span><br><span class="line"><span class="hljs-punctuation">}</span></span><br></pre></td></tr></tbody></table>

重启 OpenClaw 服务

<table><tbody><tr><td class="gutter"><pre><span class="line">1</span><br></pre></td><td class="code"><pre><span class="line">openclaw gateway restart</span><br></pre></td></tr></tbody></table>

执行以下命令确认 dingtalk-connector 已加载

<table><tbody><tr><td class="gutter"><pre><span class="line">1</span><br></pre></td><td class="code"><pre><span class="line">openclaw plugins list</span><br></pre></td></tr></tbody></table>

### [](#使用钉钉机器人 "使用钉钉机器人")使用钉钉机器人

> 场景一：单聊中使用机器人

在顶部搜索框中搜索已创建机器人名称直接使用。

[![](https://img.koenli.com/%E7%8E%A9%E8%BD%ACOpenClaw%20%7C%20OpenClaw%E5%AE%89%E8%A3%85%E4%B8%8EIM%E6%8E%A5%E5%85%A5%E6%96%87%E6%A1%A3_20.png)](https://img.koenli.com/%E7%8E%A9%E8%BD%ACOpenClaw%20%7C%20OpenClaw%E5%AE%89%E8%A3%85%E4%B8%8EIM%E6%8E%A5%E5%85%A5%E6%96%87%E6%A1%A3_20.png)

> 场景二：群聊中使用机器人

打开钉钉客户端，进入任意群聊。

+   如果是已有群聊，需要确保群归属组织与创建机器人时的组织相同。
+   创建新的群聊，请确保创建时候选择的归属组织与创建机器人时的组织相同。

单击群设置（右上角），然后选择 “`机器人`”，在机器人管理模块下，选择 “`添加机器人`”，搜索上面创建并发布的机器人名称，点击机器人进行添加即可。

[![](https://img.koenli.com/%E7%8E%A9%E8%BD%ACOpenClaw%20%7C%20OpenClaw%E5%AE%89%E8%A3%85%E4%B8%8EIM%E6%8E%A5%E5%85%A5%E6%96%87%E6%A1%A3_21.png)](https://img.koenli.com/%E7%8E%A9%E8%BD%ACOpenClaw%20%7C%20OpenClaw%E5%AE%89%E8%A3%85%E4%B8%8EIM%E6%8E%A5%E5%85%A5%E6%96%87%E6%A1%A3_21.png)

机器人添加成功后，通过 @机器人，实现自动回复。

## [](#对接飞书 "对接飞书")对接飞书

> **说明**  
> OpenClaw 内置有飞书插件，可以以机器人的身份运行，能收发消息、执行有限操作；飞书官方也在 2026.3.6 发布了官方插件，在获得授权后，OpenClaw 可以直接以你的身份看文档找资料、核对日历看档期、理解群聊上下文。两者详细区别整理可[点击此处](https://bytedance.larkoffice.com/docx/MFK7dDFLFoVlOGxWCv5cTXKmnMh#YufzddsvUoSUykxnwdxcqRWYn4g)查看，本文对接的是飞书官方插件。

> **说明**  
> 官方插件仍处于不断更新迭代过程中，安装方法也在不断优化，如果发现安装步骤与本文介绍有出入建议查看[官方对接文档](https://bytedance.larkoffice.com/docx/MFK7dDFLFoVlOGxWCv5cTXKmnMh)获取最新对接指南

### [](#安装飞书官方插件 "安装飞书官方插件")安装飞书官方插件

在运行 OpenClaw 的设备上执行以下命令安装飞书官方插件

> **说明**  
> 如果执行命令行出错，可在命令行前增加 `sudo` 重新执行

<table><tbody><tr><td class="gutter"><pre><span class="line">1</span><br></pre></td><td class="code"><pre><span class="line">npx -y @larksuite/openclaw-lark-tools install</span><br></pre></td></tr></tbody></table>

执行过程中，可选择 “`新建机器人`” 或 “`关联已有机器人`”

> **说明**  
> 我安装的时候没有出现该选项，直接出来二维码，官方文档也有很多人反馈这个问题，估计是插件 Bug

[![](https://img.koenli.com/%E7%8E%A9%E8%BD%ACOpenClaw%20%7C%20OpenClaw%E5%AE%89%E8%A3%85%E4%B8%8EIM%E6%8E%A5%E5%85%A5%E6%96%87%E6%A1%A3_22.png)](https://img.koenli.com/%E7%8E%A9%E8%BD%ACOpenClaw%20%7C%20OpenClaw%E5%AE%89%E8%A3%85%E4%B8%8EIM%E6%8E%A5%E5%85%A5%E6%96%87%E6%A1%A3_22.png)

通过手机飞书客户端扫描二维码，填写机器人名称，点击 “`创建`”

创建完成后，点击 “`打开机器人`”，在飞书中向机器人发送任意消息，即可开始对话。

> **说明**
> 
> +   若希望快速完成用户授权，便于后续 OpenClaw 通过你的身份完成消息、文档、多维表格、日历等任务，可以在飞书对话中发送 `/feishu auth` 来完成批量授权。
> +   为了让 OpenClaw 能学会这些新技能并正确使用，建议在飞书对话中发送`学习一下我安装的新飞书插件，列出有哪些能力`。

在飞书对话中发送 `/feishu start`。如果返回了版本号信息，则代表安装成功。

### [](#高级配置指令 "高级配置指令")高级配置指令

#### [](#切换到流式输出 "切换到流式输出")切换到流式输出

在运行 OpenClaw 的设备上执行以下命令可切换到流式输出

<table><tbody><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br></pre></td><td class="code"><pre><span class="line">openclaw config <span class="hljs-built_in">set</span> channels.feishu.streaming <span class="hljs-literal">true</span></span><br><span class="line">openclaw gateway restart</span><br></pre></td></tr></tbody></table>

在运行 OpenClaw 的设备上执行以下命令可关闭流式输出

<table><tbody><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br></pre></td><td class="code"><pre><span class="line">openclaw config <span class="hljs-built_in">set</span> channels.feishu.streaming <span class="hljs-literal">false</span></span><br><span class="line">openclaw gateway restart</span><br></pre></td></tr></tbody></table>

在运行 OpenClaw 的设备上执行以下命令设置流式输出卡片上支持显示更多内容

<table><tbody><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br></pre></td><td class="code"><pre><span class="line">openclaw config <span class="hljs-built_in">set</span> channels.feishu.footer.elapsed <span class="hljs-literal">true</span>  <span class="hljs-comment"># 开启耗时</span></span><br><span class="line">openclaw config <span class="hljs-built_in">set</span> channels.feishu.footer.status <span class="hljs-literal">true</span>  <span class="hljs-comment"># 开启状态展示</span></span><br><span class="line">openclaw gateway restart</span><br></pre></td></tr></tbody></table>

#### [](#常用诊断命令与问题修复方法 "常用诊断命令与问题修复方法")常用诊断命令与问题修复方法

可以在与 OpenClaw 的对话中发送以下命令

+   `/feishu start`：确认是否安装成功
+   `/feishu doctor`：检查配置是否正常
+   `/feishu auth`：批量完成用户授权

插件中也内置了常见问题的解决方案，遇到问题都可以先问问小龙虾。如果不行，则执行以下指令查看问题，并尝试自动修复

<table><tbody><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br><span class="line">7</span><br><span class="line">8</span><br></pre></td><td class="code"><pre><span class="line"><span class="hljs-comment"># 查看问题</span></span><br><span class="line">npx @larksuite/openclaw-lark-tools doctor</span><br><span class="line"><span class="hljs-comment"># 尝试自动修复</span></span><br><span class="line">npx @larksuite/openclaw-lark-tools doctor --fix</span><br><span class="line"><span class="hljs-comment"># 查看版本信息</span></span><br><span class="line">npx @larksuite/openclaw-lark-tools info </span><br><span class="line"><span class="hljs-comment"># 查看详细配置信息</span></span><br><span class="line">npx @larksuite/openclaw-lark-tools info --all</span><br></pre></td></tr></tbody></table>

## [](#对接微信 "对接微信")对接微信

> **说明**  
> 2026.3.22 凌晨，[微信官方正式推出微信「ClawBot」插件](https://mp.weixin.qq.com/s/WUvOXV-XQcPv6SJjEEnOaQ)，支持接入 OpenClaw。用户扫码或复制命令，即可将 OpenClaw 接入微信。连接后，用户就能通过微信聊天的方式，快速调用自己的 “龙虾” 高效互动。

首先确认电脑和手机的微信版本是最新版本，在 “`我的`”-“`设置`”-“`插件`” 中能看到 “微信 ClawBot 插件” 即可

> **说明**  
> 微信 ClawBot 插件在逐步放量中。更新至最新版本，敬请期待。

在运行 OpenClaw 的设备上执行以下命令安装微信 ClawBot 插件

<table><tbody><tr><td class="gutter"><pre><span class="line">1</span><br></pre></td><td class="code"><pre><span class="line">npx -y @tencent-weixin/openclaw-weixin-cli@latest install</span><br></pre></td></tr></tbody></table>

安装完成后终端会弹出一个二维码，使用微信扫描二维码，直接点击 “连接”

[![](https://img.koenli.com/%E7%8E%A9%E8%BD%ACOpenClaw%20%7C%20OpenClaw%E5%AE%89%E8%A3%85%E4%B8%8EIM%E6%8E%A5%E5%85%A5%E6%96%87%E6%A1%A3_23.png)](https://img.koenli.com/%E7%8E%A9%E8%BD%ACOpenClaw%20%7C%20OpenClaw%E5%AE%89%E8%A3%85%E4%B8%8EIM%E6%8E%A5%E5%85%A5%E6%96%87%E6%A1%A3_23.png)

然后插件就会自动完成 Channel 的添加并重启 OpenClaw 生效

[![](https://img.koenli.com/%E7%8E%A9%E8%BD%ACOpenClaw%20%7C%20OpenClaw%E5%AE%89%E8%A3%85%E4%B8%8EIM%E6%8E%A5%E5%85%A5%E6%96%87%E6%A1%A3_24.png)](https://img.koenli.com/%E7%8E%A9%E8%BD%ACOpenClaw%20%7C%20OpenClaw%E5%AE%89%E8%A3%85%E4%B8%8EIM%E6%8E%A5%E5%85%A5%E6%96%87%E6%A1%A3_24.png)

现在你就可以在手机微信中与 “微信 ClawBot” 机器人聊天了。

## [](#参考文档 "参考文档")参考文档

* * *

+   [https://docs.openclaw.ai/install](https://docs.openclaw.ai/install)
+   [🔥玩转 OpenClaw｜云上 OpenClaw (Clawdbot) 快速接入 QQ 指南](https://cloud.tencent.com/developer/article/2626045)
+   [使用 QQBot 接入 OpenClaw 的” 养虾” 指南](https://docs.qq.com/doc/DS2FmdkJZZEJJWEFF)
+   [OpenClaw 接入企业微信智能机器人](https://open.work.weixin.qq.com/help2/pc/cat?doc_id=21657)
+   [一键创建 OpenClaw 机器人・即刻拥有钉钉 AI 助理](https://open.dingtalk.com/document/dingstart/build-dingtalk-ai-employees)
+   [OpenClaw 飞书官方插件使用指南（公开版）](https://bytedance.larkoffice.com/docx/MFK7dDFLFoVlOGxWCv5cTXKmnMh)
+   [刚刚，微信推出官方龙虾插件](https://mp.weixin.qq.com/s/o_FPvJ0tY6aGqGn4Ea7Rpw)
