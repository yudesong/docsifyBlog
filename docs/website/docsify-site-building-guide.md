# Docsify 建站指南

本文整理自“快速入门”“多页文档”“定制导航栏”三篇内容，按从初始化到导航配置的顺序，串成一份 docsify 建站速查。

## 1. 初始化项目

推荐先安装 `docsify-cli`，它可以快速初始化项目并启动本地预览。

```bash
npm i docsify-cli -g
```

在目标目录初始化文档项目：

```bash
docsify init ./docs
```

初始化后，`docs` 目录通常包含：

- `index.html`：docsify 入口文件。
- `README.md`：默认首页内容。
- `.nojekyll`：用于阻止 GitHub Pages 忽略下划线开头的文件。

日常写作时，直接编辑 `README.md` 或新增 Markdown 页面即可。

## 2. 本地预览

使用 docsify 启动本地服务：

```bash
docsify serve docs
```

默认访问地址是：

```text
http://localhost:3000
```

如果不使用 docsify-cli，也可以用 Python 启动静态服务：

```bash
cd docs
python -m http.server 3000
```

## 3. 手动创建入口文件

不通过 `docsify init` 时，也可以手动创建 `index.html`：

```html
<!DOCTYPE html>
<html>
  <head>
    <meta http-equiv="X-UA-Compatible" content="IE=edge,chrome=1" />
    <meta name="viewport" content="width=device-width,initial-scale=1" />
    <meta charset="UTF-8" />
    <link rel="stylesheet" href="//cdn.jsdelivr.net/npm/docsify/themes/vue.css" />
  </head>
  <body>
    <div id="app">加载中</div>
    <script>
      window.$docsify = {
        loadSidebar: true,
      };
    </script>
    <script src="//cdn.jsdelivr.net/npm/docsify/lib/docsify.min.js"></script>
  </body>
</html>
```

如果修改挂载元素，需要同步配置 `el`，并给对应元素加上 `data-app`：

```html
<div data-app id="main">加载中</div>

<script>
  window.$docsify = {
    el: "#main",
  };
</script>
```

## 4. 多页文档

docsify 通过文件路径映射路由。比如创建 `guide.md` 后，对应访问路径是：

```text
/#/guide
```

示例目录：

```text
docs
├── README.md
├── guide.md
└── zh-cn
    ├── README.md
    └── guide.md
```

对应访问关系：

```text
docs/README.md        => http://domain.com
docs/guide.md         => http://domain.com/#/guide
docs/zh-cn/README.md  => http://domain.com/#/zh-cn/
docs/zh-cn/guide.md   => http://domain.com/#/zh-cn/guide
```

子目录中的 `README.md` 会作为该目录路由的默认页面。

## 5. 侧边栏

开启侧边栏需要配置 `loadSidebar`：

```html
<script>
  window.$docsify = {
    loadSidebar: true,
  };
</script>
```

然后在 `docs` 目录创建 `_sidebar.md`：

```markdown
* [首页](README.md)
* [指南](guide.md)
```

侧边栏文件会按目录向上查找。例如访问 `/zh-cn/more-pages` 时，docsify 会优先找 `/zh-cn/_sidebar.md`，找不到再回退到 `/_sidebar.md`。

如果希望所有路径都使用根目录侧边栏，可以配置 `alias`：

```html
<script>
  window.$docsify = {
    loadSidebar: true,
    alias: {
      "/.*/_sidebar.md": "/_sidebar.md",
    },
  };
</script>
```

## 6. 页面目录

开启 `subMaxLevel` 后，docsify 会把页面内标题自动追加到侧边栏中：

```html
<script>
  window.$docsify = {
    loadSidebar: true,
    subMaxLevel: 2,
  };
</script>
```

如果某个标题不想出现在目录里，可以加：

```markdown
## Header <!-- {docsify-ignore} -->
```

如果整页标题都不想加入目录，可以在第一个标题上加：

```markdown
# Getting Started <!-- {docsify-ignore-all} -->
```

## 7. 导航栏

导航栏可以直接写在 `index.html` 中，链接需要以 `#/` 开头：

```html
<body>
  <nav>
    <a href="#/">EN</a>
    <a href="#/zh-cn/">中文</a>
  </nav>
  <div id="app"></div>
</body>
```

也可以使用 `_navbar.md` 管理导航。先开启 `loadNavbar`：

```html
<script>
  window.$docsify = {
    loadNavbar: true,
  };
</script>
```

再创建 `_navbar.md`：

```markdown
* [首页](/)
* [简体中文](/zh-cn/)
```

导航内容较多时，可以写成嵌套列表，docsify 会渲染成下拉菜单：

```markdown
* 入门
  * [快速开始](quickstart.md)
  * [多页文档](more-pages.md)
  * [定制导航栏](custom-navbar.md)

* 配置
  * [配置项](configuration.md)
  * [主题](themes.md)
  * [插件](plugins.md)
```

如果使用 GitHub Pages，请保留 `.nojekyll`，否则 `_sidebar.md`、`_navbar.md` 这类以下划线开头的文件可能不会被正确发布。
