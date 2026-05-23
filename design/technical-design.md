# AI Chat Hub 技术实现与可行性方案

## 1. 当前项目基础

当前仓库是 uTools 空项目模板，技术栈为：

- 前端：Vue 3 + Vite
- UI 基座：建议采用 `shadcn-vue + AI Elements Vue`
- 样式体系：建议采用 Tailwind CSS + CSS Variables
- uTools 配置：`public/plugin.json`
- preload：`public/preload/services.js`
- 类型提示：`utools-api-types`

轻量化是本插件的硬约束：依赖数量、打包体积、后台浏览器实例数、本地历史数据规模都需要主动控制，避免把 uTools 插件做成常驻重型桌面应用。

模板已经具备 uTools 插件开发的关键入口：

- `plugin.json` 的 `main` 指向 `index.html`，开发态 `development.main` 指向 Vite 服务。
- `plugin.json` 的 `preload` 指向 `preload/services.js`。
- preload 脚本可以通过 CommonJS 使用 Node.js 能力，并把服务挂到 `window` 供 Vue 渲染层调用。

## 2. 官方能力调研结论

### 2.1 preload

uTools 基于 Electron，preload 机制允许插件在渲染进程中释放部分沙箱限制，使用 Node.js API 访问本地文件、跨域网络资源和本地存储。当前模板的 `services.js` 已经通过 `window.services` 暴露了读写文件能力，后续应扩展为 `window.aiHub` 服务层。

适合放在 preload 的能力：

- uTools DB / 加密存储封装。
- ubrowser 自动化编排。
- 平台适配器注册与调用。
- API Key、Cookie 这类敏感信息的读写封装。
- 日志、导入导出、摘要生成等本地服务。

### 2.2 数据存储

uTools 提供：

- `utools.db`：文档型本地数据库，支持 `put/get/remove/bulkDocs/allDocs`。
- `utools.dbStorage`：简单键值存储。
- `utools.dbCryptoStorage`：加密键值存储。

建议：

- 普通业务数据用 `utools.db`，按文档拆分，避免所有历史写入同一个大文档。
- 用户偏好用 `utools.dbStorage`。
- API Key、用户授权信息、可选 cookie 备份用 `utools.dbCryptoStorage`。

### 2.3 ubrowser

uTools 官方提供 `utools.ubrowser` 可编程自动化浏览器，支持：

- 打开 URL：`goto(url)`
- 设置视窗 / UA：`viewport(width, height)`、`useragent(ua)`
- 显示 / 隐藏：`show()`、`hide()`
- 等待元素或 JS 条件：`wait(selector)`、`wait(fn)`
- 输入 / 点击 / 按键：`input()`、`value()`、`click()`、`press()`
- 在网页执行 JS 并取回结果：`evaluate(fn)`
- 读取 / 设置 / 清理 Cookie：`cookies()`、`setCookies()`、`clearCookies()`
- 获取 Markdown / 截图：`markdown()`、`screenshot()`
- 启动新实例或复用空闲实例：`run(options)`、`run(ubrowserId)`、`utools.getIdleUBrowsers()`

关键限制：

- 隐藏窗口适合执行一次任务；官方类型定义显示，隐藏窗口任务结束后可能自动销毁，显示窗口才更容易保留实例 ID。
- `ubrowser` 不是完整 Playwright / Puppeteer，不应假设有稳定的网络拦截、CDP 调试协议或长期无头浏览器池。
- 自动化第三方 AI 网页依赖 DOM 结构，平台改版会导致适配器失效。

### 2.4 uTools AI

uTools 类型定义包含 `allAiModels()` 和 `ai()`，官方文档也说明支持流式 / 非流式调用 AI。它可以作为摘要生成的增强方案，但不能作为默认免费能力假设，因为用户是否配置可用模型、调用额度和成本不可控。

## 3. 总体架构

建议采用四层结构：

```text
Vue Renderer
  - 页面、状态管理、消息渲染、设置表单

window.aiHub preload service
  - 对外暴露统一异步方法
  - 隔离 uTools API、Node.js、ubrowser、加密存储

Domain services
  - PlatformService
  - ChatOrchestrator
  - HistoryRepository
  - SummaryService
  - AuthService
  - LogService

Adapters
  - WebPlatformAdapter: DeepSeek / Qwen / Kimi / Doubao ...
  - ApiPlatformAdapter: OpenAI-compatible / platform official API ...
```

前端只调用 `window.aiHub`，不直接操作 `utools.ubrowser`。这样后续从网页自动化切换到 API 时，前端和历史数据结构不需要大改。

### 3.1 前端技术选型

推荐组合：

- 基础框架：Vue 3 + Vite，沿用当前 uTools 模板。
- UI 基座：`shadcn-vue`。
- AI 对话组件：`AI Elements Vue`。
- 样式：Tailwind CSS + CSS Variables。
- 图标：`lucide-vue-next`，与 shadcn 生态一致。
- 状态管理：Pinia，管理当前会话、平台配置、历史筛选和任务状态。
- 路由：Vue Router；如果插件只保留单窗口，也可以用轻量的本地 view state，等页面增多后再引入路由。
- Markdown 与代码渲染：优先使用 AI Elements Vue 的 `message`、`code-block` 等组件；如需自定义，再补 `markdown-it` / `shiki`。

选择 `shadcn-vue + AI Elements Vue` 的原因：

- 与当前 Vue 3 技术栈一致，不需要改成 React。
- AI Elements Vue 面向 AI 原生界面，提供 `conversation`、`message`、`prompt-input`、`code-block`、`reasoning`、`sources`、`tool`、`model-selector` 等组件，能覆盖对话页、对比回答、推理过程、引用来源、模型 / 平台选择等核心交互。
- shadcn-vue 采用“组件源码进入项目”的方式，适合 uTools 插件这种需要轻量、可控、易打包的桌面插件场景。
- Tailwind + CSS Variables 更适合后续做暗色模式、平台色标识、双栏对比、紧凑布局，不会被传统后台组件库的默认视觉强绑定。

不建议首期混用 Element Plus / Ant Design Vue / Naive UI：

- 这些库适合后台管理，但本产品的核心体验是 AI 对话，不是通用管理后台。
- 混用多套组件库会带来样式、体积、主题和交互一致性成本。
- 设置页需要的表单、表格、弹窗、开关、选择器，shadcn-vue 已能覆盖；缺口可以用本地组件补齐。

引入注意事项：

- 当前项目还没有 Tailwind，需要在第一阶段补齐 `tailwind.config`、全局 CSS 变量和基础主题。
- AI Elements Vue 依赖 shadcn-vue / Tailwind 生态，安装组件时会把组件源码加入项目，应纳入后续代码维护范围。
- uTools 插件运行在 Electron 渲染环境中，应避免依赖服务端渲染特性；所有 UI 组件应按纯客户端 Vue 组件使用。
- 对话消息的真实数据源来自 `window.aiHub`，不要直接使用 AI Elements Vue 内置的网络请求假设，组件只负责展示和交互。

### 3.2 轻量化约束

轻量化目标：

- 首屏打开快，插件唤起后不阻塞 uTools 主流程。
- 静态资源少，避免引入多个大型 UI 库和重复图标库。
- 后台任务可控，默认不常驻多个浏览器实例。
- 历史记录可搜索，但不无限制堆积大文本。

依赖策略：

- 只保留一套 UI 体系：`shadcn-vue + AI Elements Vue + lucide-vue-next`。
- 不引入 Element Plus / Ant Design Vue / Naive UI 等大型组件库。
- 不引入 Moment.js、Lodash 全量包等高体积工具库；日期格式化优先用原生 `Intl` 或小工具函数。
- Markdown、代码高亮、虚拟列表等能力按需引入；只有真实需要时再补依赖。
- 平台适配器按模块拆分，未来可做动态加载，避免所有平台逻辑一次性进入首屏路径。

前端渲染策略：

- 对话列表和历史列表首期可以分页 / 分段加载，历史量变大后再引入虚拟列表。
- 消息正文只保存和渲染必要文本；网页原始 HTML 默认不保存，除非开发调试开关开启。
- 长回答折叠非当前轮详情，避免一次性渲染大量 Markdown / 代码块。
- 对比模式最多同时展示两个平台；不做多平台无限并排。

ubrowser 资源策略：

- 默认同时最多运行 2 个 `ubrowser` 任务，对应普通模式 1 个、对比模式 2 个。
- 隐藏窗口只作为一次性任务执行，任务结束后释放；不把隐藏窗口设计为长期常驻池。
- 登录窗口使用 `show()` 让用户手动操作，完成后关闭或进入空闲状态，不长期保留。
- 对同一平台做任务队列，避免用户连续发送导致多实例抢占登录态或触发限流。

数据规模策略：

- `conversation` 保存标题、平台、URL、摘要、最近问题和回答预览。
- `message` 保存用户消息和最终回答文本；首期不保存截图、完整 DOM、网络包。
- 摘要默认本地生成，避免每条历史都调用 AI 产生额外延迟和成本。
- 历史记录提供归档、删除和导出；后续可增加保留策略，例如只展示最近 N 条、按需加载旧记录。

构建与验证：

- 每次引入新依赖后检查 `npm run build` 输出体积。
- 优先使用 Vite 的 tree-shaking，不引入 CommonJS 大包作为前端依赖。
- 打包资源中图片使用压缩后的 PNG/WebP；图标优先用 lucide 组件，不维护大图标集。
- 如果后续体积明显增长，需要增加 bundle 分析脚本，再决定是否拆包或替换依赖。

## 4. preload 服务接口设计

建议在 `public/preload/services.js` 暴露：

```js
window.aiHub = {
  platforms: {
    list,
    save,
    remove,
    setDefault,
    checkAuth,
    openLogin
  },
  chats: {
    start,
    send,
    stop,
    resume,
    compare
  },
  histories: {
    list,
    search,
    get,
    remove,
    export,
    rebuildSummary
  },
  settings: {
    get,
    set
  }
}
```

服务调用约定：

- 所有方法返回 Promise。
- 所有错误统一转成 `{ code, message, detail }`。
- 对话生成这种长任务用事件回调或轻量事件总线把增量状态推给前端。
- 如果暂不引入事件系统，V0.2 可以先用前端轮询任务状态。

## 5. 数据模型

### 5.1 Platform

文档 ID：`platform:{platformId}`

```ts
type PlatformDoc = {
  _id: string
  type: 'platform'
  platformId: string
  name: string
  icon?: string
  enabled: boolean
  isDefault: boolean
  allowCompare: boolean
  connector: 'web' | 'api'
  web?: {
    homeUrl: string
    newChatUrl: string
    loginUrl: string
    supportsConversationUrl: boolean
    selectorsVersion: string
  }
  api?: {
    baseUrl?: string
    model?: string
    stream?: boolean
    compatible?: 'openai' | 'custom'
    credentialKey?: string
  }
  auth?: {
    status: 'unknown' | 'logged_in' | 'logged_out' | 'expired'
    checkedAt?: number
  }
  createdAt: number
  updatedAt: number
}
```

### 5.2 Conversation

文档 ID：`conversation:{conversationId}`

```ts
type ConversationDoc = {
  _id: string
  type: 'conversation'
  conversationId: string
  platformId: string
  platformName: string
  mode: 'single' | 'compare-child'
  compareSessionId?: string
  title: string
  url?: string
  summary: string
  lastQuestion?: string
  lastAnswerPreview?: string
  messageCount: number
  status: 'active' | 'archived' | 'failed'
  createdAt: number
  updatedAt: number
}
```

### 5.3 Message

文档 ID：`message:{conversationId}:{timestamp}:{shortId}`

```ts
type MessageDoc = {
  _id: string
  type: 'message'
  conversationId: string
  platformId: string
  role: 'user' | 'assistant' | 'system'
  content: string
  rawHtml?: string
  source: 'web' | 'api' | 'manual'
  status: 'sending' | 'streaming' | 'done' | 'failed'
  errorCode?: string
  createdAt: number
}
```

### 5.4 CompareSession

文档 ID：`compare:{compareSessionId}`

```ts
type CompareSessionDoc = {
  _id: string
  type: 'compareSession'
  compareSessionId: string
  title: string
  prompt: string
  platformIds: string[]
  conversationIds: string[]
  summary: string
  createdAt: number
  updatedAt: number
}
```

### 5.5 Secret

敏感数据不进 `utools.db`：

- `secret:apiKey:{platformId}` 存 API Key。
- `secret:cookies:{platformId}` 可选存 cookie 备份。
- 默认不主动持久化网页 cookie，优先依赖 ubrowser 自身 session。

## 6. 平台适配器设计

统一接口：

```ts
type WebPlatformAdapter = {
  id: string
  name: string
  urls: {
    home: string
    newChat: string
    login: string
  }
  selectors: {
    input: string
    sendButton?: string
    assistantMessages: string
    userMessages?: string
    generatingIndicator?: string
    loginMarker?: string
  }
  checkAuth(ctx): Promise<AuthStatus>
  openLogin(ctx): Promise<void>
  startConversation(ctx, input): Promise<SendResult>
  resumeConversation(ctx, url, input): Promise<SendResult>
  extractConversation(ctx): Promise<ExtractedConversation>
}
```

适配器职责：

- 维护平台 URL 和选择器。
- 判断是否登录。
- 打开新对话或历史 URL。
- 输入问题并提交。
- 等待生成完成。
- 抽取回答文本、标题和当前 URL。

不放入适配器的职责：

- 历史记录存储。
- UI 状态管理。
- 摘要生成。
- API Key 加密存储。

## 7. 网页自动化流程

### 7.1 登录

登录必须优先使用可视化窗口：

```js
await utools.ubrowser
  .goto(adapter.urls.login)
  .show()
  .run({ width: 1200, height: 800, center: true })
```

原因：

- 多数 AI 平台需要手机号、扫码、验证码或二次验证。
- 隐藏窗口里登录成功率低，也不利于用户理解授权行为。
- 不应尝试绕过验证码或风控。

登录完成后通过 `checkAuth` 检测登录态，必要时读取 cookie 状态，但不默认导出 cookie。

### 7.2 新对话

推荐任务链：

```js
const [result] = await utools.ubrowser
  .goto(platform.web.newChatUrl)
  .hide()
  .wait(adapter.selectors.input, { timeout: 30000 })
  .input(adapter.selectors.input, prompt)
  .press('enter')
  .wait(() => {
    // 页面内判断生成结束，具体逻辑由适配器提供
    return Boolean(window.__AI_HUB_DONE__)
  }, { timeout: 120000, interval: 1000 })
  .evaluate(() => {
    return {
      title: document.title,
      url: location.href,
      text: document.body.innerText
    }
  })
  .run({ show: false, width: 1200, height: 900 })
```

实际实现时不应使用 `document.body.innerText` 作为最终方案，需要平台级选择器只提取消息区。

### 7.3 继续历史对话

继续对话依赖三个条件：

- 历史记录里保存了可恢复的会话 URL。
- 当前平台登录态仍有效。
- 平台允许通过 URL 直接打开历史会话。

流程：

1. 打开 `conversation.url`。
2. 检查是否跳到登录页或首页。
3. 等待消息区和输入框出现。
4. 输入新问题并提交。
5. 抽取新增回答，更新本地历史。

如果 URL 失效：

- 标记历史为 `failed` 或 `url_invalid`。
- 提供“复制原问题到新对话”。
- 不删除原历史。

### 7.4 对比模式

使用两个独立任务并发：

```js
const jobs = selectedPlatformIds.map(platformId =>
  chatOrchestrator.sendToPlatform({ platformId, prompt, compareSessionId })
)

const results = await Promise.allSettled(jobs)
```

注意：

- 每个平台有独立超时、错误状态和重试。
- 并发不要无限制，默认最多 2 个实例。
- 同一平台不建议同时开多个任务，容易触发平台限流或会话覆盖。

## 8. 抓取回答的可行性分析

### 8.1 可行能力

| 能力 | 可行性 | 说明 |
| --- | --- | --- |
| 打开 AI 网页 | 高 | `ubrowser.goto` 支持外部 URL |
| 可视化登录 | 高 | `ubrowser.show` 允许用户直接操作网页 |
| 复用登录态 | 中高 | 可依赖 ubrowser session / cookie，但不同平台策略不同 |
| 自动输入问题 | 中 | 依赖输入框选择器、contenteditable 行为和平台快捷键 |
| 提取最终回答 | 中 | 可用 DOM 选择器和 `evaluate`，但平台改版会失效 |
| 流式展示回答 | 中低 | 需要轮询 DOM 差异；没有官方网络流拦截能力 |
| 打开历史链接继续问 | 中 | 取决于平台是否有稳定会话 URL 和登录态 |
| 双平台对比 | 中 | 技术可做，但资源、限流和登录态冲突要控制 |
| 后台长期常驻实例 | 低 | 隐藏窗口任务结束后可能销毁，不宜作为核心假设 |

### 8.2 主要风险

DOM 变化：

- 第三方 AI 平台频繁改版，选择器可能失效。
- 需要适配器版本、健康检查和快速修复机制。

反自动化与风控：

- 频繁自动发送、并发、多账号切换可能触发风控。
- 不做验证码绕过，不做规避检测逻辑。

登录限制：

- 扫码、短信、二次验证必须用户手动完成。
- Cookie 可能过期或绑定设备 / UA。

内容提取质量：

- 网页可能使用虚拟列表，只保留可视区域 DOM。
- Markdown、代码块、表格、引用、图片等结构可能丢失。
- 首期建议先提取纯文本，后续再做 Markdown 结构化。

法律与平台条款：

- 自动化访问和抓取第三方网页可能受平台服务条款限制。
- 产品应定位为用户本地自用的自动化助手，不提供绕过限制、批量爬取或未授权采集能力。

## 9. 摘要生成方案

默认免费摘要算法：

1. 取最近一轮用户问题和 AI 回答。
2. 清理空白、代码块、重复提示词和免责声明。
3. 提取前 300-500 字作为候选。
4. 识别结论句关键词：`总结`、`建议`、`因此`、`优先`、`步骤`、`原因`。
5. 生成 80-160 字摘要。
6. 存入 `conversation.summary`，用于历史搜索。

增强摘要：

- 如果 `utools.allAiModels()` 返回可用模型，且用户启用“AI 生成摘要”，可调用 `utools.ai()`。
- 如果平台配置了 API Key，也可以调用用户自己的模型生成摘要。
- 增强摘要必须可关闭，并显示可能产生调用成本。

## 10. API 接入预留

平台管理中预留 API 配置，技术上建议实现 `Connector` 抽象：

```ts
type ChatConnector = {
  type: 'web' | 'api'
  checkAuth(input): Promise<AuthStatus>
  send(input): Promise<ChatResult>
  stream?(input, onChunk): Promise<ChatResult>
}
```

API 模式优先支持 OpenAI-compatible 格式：

- `baseUrl`
- `apiKey`
- `model`
- `messages`
- `stream`

这样 DeepSeek、通义千问、Kimi、豆包等未来如果开放兼容接口，可以复用同一套 API 连接器。

## 11. 开发落地顺序

### Step 1: 清理模板与基础路由

- 修正 `plugin.json` 中文乱码。
- 将功能入口改成“AI Chat Hub”和“设置”。
- 初始化 Tailwind CSS、shadcn-vue、AI Elements Vue。
- 建立 Vue 页面骨架：对话页、设置页。
- 建立基础 UI 规范：暗色模式、平台色、消息气泡、双栏对比布局、紧凑设置页布局。
- 建立轻量化基线：记录首版依赖清单、构建产物体积、首屏组件数量。

### Step 2: preload 服务层

- 新增 `window.aiHub`。
- 封装 `utools.db`、`dbStorage`、`dbCryptoStorage`。
- 实现平台和历史 CRUD。

### Step 3: 平台配置与历史管理

- 内置默认平台。
- 设置页支持启用、默认、对比候选、API 配置预留。
- 历史页支持新增、搜索、删除、打开。

### Step 4: 单平台 ubrowser 验证

- 选择一个平台做适配器。
- 实现登录检测、打开登录窗口。
- 实现发送问题、抓取最终回答、保存历史。

### Step 5: 对比模式

- 实现两个平台并发任务。
- 对话区支持双列渲染和小屏 Tab。
- 保存 compare session。

### Step 6: 稳定性与维护工具

- 适配器健康检查。
- 错误日志。
- 选择器调试入口。
- 摘要重建。

## 12. 验证计划

本地验证：

- `npm run build` 能通过。
- 构建产物体积在预期范围内，新增依赖需要说明用途。
- uTools 开发者工具加载插件目录。
- `window.aiHub.platforms.list()` 返回内置平台。
- 新增 / 修改 / 删除平台数据后重启插件仍保留。
- 历史记录搜索覆盖标题、平台、摘要。

ubrowser 验证：

- 未登录时能打开可视化登录窗口。
- 登录后 `checkAuth` 能正确判断。
- 新对话能发送问题并提取回答。
- 历史 URL 能恢复或明确失败。
- 对比模式中一个平台失败不影响另一个平台。

回归验证：

- 平台 DOM 变更后适配器失败要有明确错误。
- 长回答、代码块、中文、英文混排能正常保存。
- 插件退出后未完成任务不会破坏历史记录。

## 13. 推荐技术判断

结论：

- 使用 uTools `ubrowser` 做后台网页访问和数据抓取是可行的，适合 MVP。
- 但它应被设计为“平台适配器的一种实现”，而不是产品长期唯一内核。
- 前端选型建议固定为 `Vue 3 + Vite + shadcn-vue + AI Elements Vue + Tailwind CSS`，与当前 uTools 模板一致，同时能获得更好的 AI 对话体验。
- 轻量化优先级高于功能堆叠：首期只引入一套 UI 体系，限制并发浏览器实例，历史数据按需加载，不保存不必要的大对象。
- 对话继续能力应优先依赖平台会话 URL + 登录态 + 本地历史，而不是长期持有隐藏浏览器实例。
- 摘要默认用本地免费规则，AI 摘要作为可选增强。
- 后续只要平台 API 可用，应优先走 API，以降低 DOM 变动和风控风险。

## 14. 参考资料

- uTools 快速开始：https://www.u-tools.cn/docs/developer/basic/getting-started.html
- uTools preload 预加载脚本：https://www.u-tools.cn/docs/developer/information/preload-js/preload-js.html
- uTools ubrowser 可编程自动化浏览器：https://www.u-tools.cn/docs/developer/utools-api/ubrowser.html
- uTools 本地数据库：https://www.u-tools.cn/docs/developer/api-reference/db/local-db.html
- uTools AI API：https://www.u-tools.cn/docs/developer/utools-api/ai.html
- 本地类型定义：`node_modules/utools-api-types/utools.api.d.ts`
