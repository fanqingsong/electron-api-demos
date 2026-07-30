# 项目架构与前后端划分

## 1. 架构概述

本项目是 Electron 桌面应用，不是传统的 Web 前后端分离项目。项目中没有独立的 HTTP 服务、REST API 或后端服务器，而是按照 Electron 的进程模型划分为：

- **主进程（Main Process）**：负责应用生命周期、窗口管理和原生系统能力，可视为“后端/系统能力层”。
- **渲染进程（Renderer Process）**：负责页面展示和用户交互，可视为“前端/UI 层”。
- **进程间通信（IPC）**：连接主进程和渲染进程。

## 2. 目录职责

```text
electron-api-demos/
├── main.js                 # Electron 主进程入口
├── index.html              # 主窗口的渲染进程入口
├── main-process/           # 主进程功能模块
├── renderer-process/       # 渲染进程交互逻辑
├── sections/               # 页面和功能演示模板
├── assets/                 # 样式、图片及前端公共脚本
├── test/                   # 测试代码
└── script/                 # 打包、发布等工程脚本
```

### 2.1 主进程

主进程代码位于：

- `main.js`
- `main-process/`

主要职责：

- 初始化 Electron 应用。
- 管理应用生命周期。
- 创建和管理 `BrowserWindow`。
- 调用菜单、对话框、托盘、快捷键等原生 API。
- 使用 `ipcMain` 接收渲染进程发送的消息。

`package.json` 将 `main.js` 声明为应用入口。`main.js` 启动时通过 `glob` 加载 `main-process/**/*.js`，随后在 Electron 的 `ready` 事件触发后创建主窗口，并加载 `index.html`。

### 2.2 渲染进程

渲染进程代码主要位于：

- `index.html`
- `sections/`
- `renderer-process/`
- `assets/`

各部分职责如下：

- `index.html`：主窗口的 HTML 壳页面，包含导航栏和内容容器。
- `sections/`：各项 Electron API 演示的 HTML 模板。
- `renderer-process/`：绑定 DOM 事件、处理用户输入，并调用 Electron 渲染进程 API。
- `assets/`：公共样式、图片、导航、页面导入和代码高亮逻辑。

`index.html` 使用 HTML Import 引入 `sections/` 中的模板。`assets/imports.js` 复制模板内容并插入主页面，各 section 再按需加载对应的 `renderer-process/` 脚本。

## 3. 启动与加载流程

```text
npm start
  │
  └── electron .
        │
        └── package.json 指定 main.js
              │
              ├── 加载 main-process/**/*.js
              │     └── 注册 IPC、菜单和原生功能
              │
              └── 创建 BrowserWindow
                    │
                    └── 加载 index.html
                          │
                          ├── 导入 sections/*.html
                          ├── assets/imports.js 注入页面模板
                          └── 加载 renderer-process/**/*.js
```

## 4. 主进程与渲染进程通信

主进程和渲染进程主要通过 Electron IPC 通信：

```text
用户操作
  │
  ▼
renderer-process/
  │ ipcRenderer.send(...)
  ▼
main-process/
  │ ipcMain.on(...)
  │ 调用系统或原生能力
  │ event.sender.send(...)
  ▼
renderer-process/
  └── 接收结果并更新页面
```

以异步消息为例：

- `renderer-process/communication/async-msg.js` 通过 `ipcRenderer.send` 发送消息。
- `main-process/communication/async-msg.js` 通过 `ipcMain.on` 接收消息并返回结果。
- 渲染进程监听回复，然后更新页面内容。

项目还演示了同步 IPC 和跨窗口 IPC。同步 IPC 会阻塞渲染进程，实际开发中应优先采用异步 IPC。

## 5. 功能模块的对应关系

一个完整演示通常由三部分组成：

```text
sections/<模块>/<功能>.html
├── 定义页面结构和演示按钮
├── 加载 renderer-process/<模块>/<功能>.js
└── 展示相关示例代码

renderer-process/<模块>/<功能>.js
├── 监听用户操作
├── 操作页面 DOM
└── 必要时向主进程发送 IPC 消息

main-process/<模块>/<功能>.js
├── 接收 IPC 消息
├── 调用 Electron 主进程 API
└── 将结果返回渲染进程
```

并非所有演示都必须同时包含三部分。只涉及页面或渲染进程 API 的功能可以没有主进程模块；应用级菜单、全局快捷键等功能也可能主要在主进程中完成。

## 6. 前后端概念映射

| 传统 Web 概念 | 本项目中的对应部分 |
| --- | --- |
| 前端页面 | `index.html`、`sections/` |
| 前端交互逻辑 | `renderer-process/`、`assets/` |
| 后端/系统能力层 | `main.js`、`main-process/` |
| HTTP/REST 通信 | Electron IPC |
| 浏览器窗口 | Electron `BrowserWindow` |

因此，本项目可以简化理解为：

```text
main-process/       ≈ 后端/系统能力层
renderer-process/   ≈ 前端交互逻辑
sections/           ≈ 前端页面模板
assets/             ≈ 前端公共资源
```

## 7. `nodeIntegration` 与 `contextIsolation`

这两个参数都配置在 `BrowserWindow` 的 `webPreferences` 中，但控制的是不同的安全边界：

- `nodeIntegration` 决定普通页面能否直接使用 Node.js 和 Electron API。
- `contextIsolation` 决定 preload 脚本与普通页面脚本是否运行在不同的 JavaScript 上下文中。

### 7.1 `nodeIntegration`

启用 `nodeIntegration` 后，渲染页面可以直接使用 `require()`：

```js
new BrowserWindow({
  webPreferences: {
    nodeIntegration: true
  }
})
```

页面代码可以直接访问 Node.js 和 Electron：

```js
const fs = require('fs')
const {ipcRenderer} = require('electron')
```

这使开发更加直接，但也会扩大攻击面。如果页面发生 XSS，注入的脚本可能获得读取文件、执行系统命令等能力。因此，生产应用通常应设置：

```js
nodeIntegration: false
```

当前项目在 `main.js` 中设置了 `nodeIntegration: true`，所以 `index.html`、`sections/` 和相关渲染进程脚本可以直接使用 `require()`。这是旧版 Electron 示例项目常见的架构。

### 7.2 `contextIsolation`

启用 `contextIsolation` 后，preload 脚本与页面脚本运行在相互隔离的上下文中：

```js
new BrowserWindow({
  webPreferences: {
    contextIsolation: true
  }
})
```

隔离后，preload 中的 `window` 和页面中的 `window` 不再是同一个对象。页面不能直接读取 preload 的变量，也不能随意篡改 preload 使用的 JavaScript 原型和全局对象。

需要向页面提供能力时，应在 preload 中通过 `contextBridge` 暴露范围有限的接口：

```js
// preload.js
const {contextBridge, ipcRenderer} = require('electron')

contextBridge.exposeInMainWorld('electronAPI', {
  sendMessage: (message) => ipcRenderer.send('message', message)
})
```

页面只能调用明确暴露的 API：

```js
window.electronAPI.sendMessage('hello')
```

不应直接向页面暴露整个 `ipcRenderer`，否则页面可以任意发送 IPC 消息，削弱接口边界。

### 7.3 参数组合

| `nodeIntegration` | `contextIsolation` | 效果 | 建议 |
| --- | --- | --- | --- |
| `true` | `false` | 页面直接拥有 Node.js 权限，且没有上下文隔离 | 风险最高 |
| `true` | `true` | preload 与页面隔离，但页面仍拥有 Node.js 权限 | 仍不推荐 |
| `false` | `false` | 页面没有 Node.js 权限，但 preload 与页面共享上下文 | 容易被页面篡改 |
| `false` | `true` | 页面无 Node.js 权限，通过 preload 暴露有限能力 | 推荐 |

需要注意：`contextIsolation: true` 不能抵消 `nodeIntegration: true` 带来的风险。只要普通页面仍能直接访问 Node.js，页面代码就具有较高系统权限。

### 7.4 推荐配置

现代 Electron 应用通常使用以下配置：

```js
const path = require('path')

new BrowserWindow({
  webPreferences: {
    nodeIntegration: false,
    contextIsolation: true,
    preload: path.join(__dirname, 'preload.js')
  }
})
```

同时还应：

- 仅通过 `contextBridge` 暴露业务所需的最小 API。
- 不向页面暴露整个 Node.js 模块或 `ipcRenderer`。
- 在主进程中校验 IPC 消息的来源和参数。
- 不直接加载不可信的远程内容。

这些建议不改变本项目当前的代码划分，但在基于该项目开发新的生产应用时需要特别注意。
