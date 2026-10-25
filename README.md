# 智能旅游助手（Travel AI Helper）

## 1. 项目简介

智能旅游助手是一个 AI 驱动的全栈旅游规划应用。用户输入目的地城市、预算和旅行天数后，系统调用大语言模型自动生成详细的行程规划（每日上午/下午/晚上安排、景点介绍、交通建议、预算分配、温馨提示）；同时提供实时 AI 对话功能，可以随时咨询目的地相关的问题。

本项目最初基于 B 站教程（AI + Vue + Node.js + LangChain.js 全栈项目）实现，后续进行了全面优化：重设计了前端视觉、补全了「我的」模块（收藏、历史、设置）、加固了后端错误处理，并完善了文档。

### 核心功能

- **智能行程规划**：按城市 + 预算 + 天数生成结构化行程，支持每日时间轴展示与预算明细进度条
- **AI 旅游咨询**：基于 SSE（Server-Sent Events）的流式对话，回复逐字呈现（打字机效果）
- **行程收藏**：一键收藏/取消收藏行程，收藏列表可随时重新生成
- **历史记录**：自动保存最近 20 条规划记录（去重置顶），支持一键清空
- **个人设置**：自定义昵称与头像（6 款预设），设置后在「我的」页与对话气泡中同步展示
- **数据管理**：所有个人数据保存在浏览器 localStorage，支持一键清除，无需注册登录

## 2. 技术栈

### 前端（travel-h5）

| 技术 | 版本 | 作用 |
| --- | --- | --- |
| Vue | 3.5 | 前端框架，全部页面使用 Composition API（script setup）编写 |
| Vite | 8 | 开发服务器与打包工具 |
| Vant | 4.10 | 移动端 UI 组件库（导航栏、表单、列表、弹窗等），通过 unplugin-vue-components + VantResolver 按需自动引入，无需手动 import 组件 |
| Vue Router | 4 | 页面路由，使用 createWebHistory 模式（HTML5 History） |
| Axios | 1.x | 普通 HTTP 请求（行程推荐接口） |
| Fetch API | 原生 | 读取 SSE 流式响应，实现对话的打字机效果 |

> 注意：本项目**没有使用 Pinia**。个人数据（收藏/历史/设置）通过自封装的 localStorage 工具（src/utils/storage.js）管理，页面间跳转参数也经由它传递，架构更轻量。

### 后端（travel-server）

| 技术 | 版本 | 作用 |
| --- | --- | --- |
| Node.js | 24（ESM 模块） | 运行环境，package.json 中 type 为 module |
| Express | 4.19 | Web 服务器框架，提供 API 路由、CORS 跨域、错误处理中间件 |
| LangChain.js | @langchain/openai 1.x | 大模型集成框架，通过 OpenAI 兼容协议调用 DeepSeek |
| dotenv | 16.x | 读取 .env 环境变量（API 密钥等敏感配置） |
| nodemon | 3.x | 开发时监听文件变化自动重启服务 |
| SSE | 原生实现 | 对话接口的流式响应（无需额外依赖） |

### 外部服务

- **DeepSeek**（https://www.deepseek.com/）：大语言模型，通过 OpenAI 兼容接口调用。模型提供商通过环境变量 MODEL_PROVIDER 配置，理论上可替换为其他 OpenAI 兼容服务。

## 3. 项目结构与文件作用

```
旅行AI助手/
├── travel-h5/                    # 前端（Vue 3 移动端 H5）
├── travel-server/                # 后端（Node.js + Express）
├── .gitignore                    # Git 忽略规则（排除 node_modules、.env、dist 等）
└── 项目文档.md                   # 本文档
```

### 3.1 前端 travel-h5

```
travel-h5/
├── index.html                    # HTML 入口模板（中文标题、移动端 viewport 设置）
├── vite.config.js                # Vite 配置：@ 路径别名、Vue 插件、Vant 按需自动引入
├── package.json                  # 前端依赖与脚本（dev/build/preview）
└── src/
    ├── main.js                   # 应用入口：创建 Vue 实例、注册路由、按序引入三层样式
    ├── App.vue                   # 根组件：路由出口 + 底部标签栏（首页/对话/我的）
    ├── style.css                 # 全局基础重置样式（移动端适配、去默认边距）
    ├── styles/
    │   ├── theme.css             # 设计令牌：品牌色、渐变、圆角、阴影等 CSS 变量，
    │   │                         #   并覆盖 Vant 主题色变量
    │   └── common.css            # 公共组件类：卡片、渐变主按钮、页面骨架、弹窗文案
    ├── router/
    │   └── index.js              # 路由表：6 个页面全部懒加载，路径统一小写
    ├── utils/
    │   ├── request.js            # 网络层：axios 实例（普通请求）+ fetchStream（SSE 流式请求），
    │   │                         #   后端地址只在 API_BASE 常量处配置一次
    │   └── storage.js            # 本地存储工具：收藏/历史/设置/行程参数的读写封装，
    │                             #   含头像预设列表与异常兜底
    ├── views/
    │   ├── Home.vue              # 首页：渐变头图 + 规划表单（城市/预算/天数）+ 热门目的地，
    │   │                         #   提交后跳转详情页生成行程
    │   ├── Detail.vue            # 详情页：行程概览卡（含收藏按钮）、每日行程折叠面板 + 时间轴、
    │   │                         #   预算明细、温馨提示/注意事项、底部「咨询 AI」入口
    │   ├── Chat.vue              # 对话页：空态欢迎页 + 快捷问题、流式聊天气泡、毛玻璃输入区；
    │   │                         #   从详情页进入时自动带入该城市的问题
    │   ├── Profile.vue           # 我的：渐变头卡（昵称/头像）+ 功能菜单入口 + 关于我们
    │   ├── Favorites.vue         # 我的收藏：收藏行程列表，点击可重新生成，支持删除（带确认弹窗）
    │   ├── History.vue           # 历史记录：最近 20 条规划记录，支持一键清空（带确认弹窗）
    │   └── Settings.vue          # 设置：修改昵称、选择头像、保存；清除全部本地数据
    └── components/
        ├── ChatBubble.vue        # 聊天气泡：AI 左侧渐变飞机头像，用户右侧（读取设置的头像）
        ├── SpotItem.vue          # 景点信息卡：景点名、时长、门票、交通、介绍
        └── BudgetTable.vue       # 预算明细：分类金额 + 渐变进度条动画
```

### 3.2 后端 travel-server

```
travel-server/
├── package.json                  # 后端依赖与脚本（start/dev，dev 使用 nodemon）
├── nodemon.json                  # nodemon 监听配置（只监听 src 下 js/json 变化）
├── .env.example                  # 环境变量示例：复制为 .env 后填写真实密钥
├── .env                          # 真实环境变量（含 API 密钥，已被 .gitignore 排除，不会提交）
└── src/
    ├── index.js                  # 服务入口：CORS、JSON 解析、请求日志中间件、
    │                             #   健康检查接口、404 兜底、全局错误处理中间件
    ├── routes/
    │   └── travel.js             # 旅游路由：/recommend（一次性 JSON）、/chat（SSE 流式），
    │                             #   均带 try/catch，异常返回结构化错误而不是挂起请求
    ├── service/
    │   └── travelService.js      # 核心业务：初始化 LangChain 大模型客户端（惰性重试）、
    │                             #   构造行程规划/对话提示词、解析大模型返回的 JSON、参数校验
    └── utils/
        └── streamUtils.js        # SSE 工具：统一设置流式响应头，提供 send/error/end 三个方法
```

## 4. 工作原理（数据流）

### 4.1 行程规划流程

1. 用户在首页填写城市、预算、天数，点击「开始规划」；表单校验通过后，参数写入 localStorage（供详情页刷新时恢复）并跳转 /detail。
2. 详情页通过 axios 向后端 `POST /api/travel/recommend` 发送参数。
3. 后端 travelService 用「专业旅游规划师」提示词要求大模型按固定 JSON 结构输出，调用 LangChain 的 ChatOpenAI（OpenAI 兼容协议接 DeepSeek）。
4. 后端从模型返回文本中提取 JSON（兼容 markdown 代码块包裹的情况），解析后一次性返回给前端。
5. 前端渲染行程概览、每日时间轴、预算进度条，同时把本次规划写入历史记录，并同步收藏状态。

### 4.2 AI 对话流程（SSE 流式）

1. 用户在对话页发送问题，前端 fetchStream 用 Fetch API 向后端 `POST /api/travel/chat` 发起请求。
2. 后端将响应头设为 `text/event-stream`，把大模型的流式输出逐块以 `data: {...}` 事件下发。
3. 前端按行解析流数据，每收到一个 chunk 就追加到当前 AI 气泡内容，实现打字机效果；流关闭时结束加载状态。
4. 若大模型调用失败，后端通过 error 事件下发错误信息，前端在气泡中展示降级文案，不会让请求无限挂起。

### 4.3 后端健壮性设计

- Express 4 不会自动捕获 async 路由异常，所有路由均手动 try/catch，保证任何失败都会返回结构化错误。
- 启动时校验必需环境变量，缺失只告警不阻断（便于无密钥时预览接口）。
-
- 每个请求记录方法、路径、状态码与耗时，便于排查问题。

## 5. API 接口

| 方法 | 路径 | 说明 | 响应形式 |
| --- | --- | --- | --- |
| GET | /api/health | 健康检查，返回服务状态、时间戳与当前模型提供商 | JSON |
| POST | /api/heartbeat | 心跳测试接口 | JSON |
| POST | /api/travel/recommend | 行程规划。请求体：city（城市）、budget（预算，≥100）、days（天数，1-30） | JSON（一次性返回完整行程） |
| POST | /api/travel/chat | AI 对话。请求体：message（用户问题） | SSE 流式（chunk / complete / error 事件） |

参数非法时返回 400 + `{ success: false, error: 原因 }`；服务异常返回 500 + 同样结构，不会出现请求挂起。

## 6. 环境要求与运行步骤

### 6.1 环境要求

- Node.js ≥ 18（开发时使用 v24）
- 一个 DeepSeek API 密钥（https://platform.deepseek.com 注册创建）

### 6.2 启动后端

1. 进入 `travel-server` 目录，执行 `npm install` 安装依赖。
2. 复制 `.env.example` 为 `.env`，填入 `DeepSeek_API_KEY`（其余项保持默认即可）。
3. 执行 `npm run dev`（nodemon 热重启）或 `npm start`。
4. 浏览器访问 `http://localhost:3300/api/health`，返回 success 即启动成功。

### 6.3 启动前端

1. 进入 `travel-h5` 目录，执行 `npm install`。
2. 执行 `npm run dev`，浏览器访问 `http://localhost:5173`。
3. 生产构建：`npm run build`，产物在 `dist/` 目录。

> 前端接口地址配置在 `src/utils/request.js` 的 `API_BASE` 常量（默认 `http://127.0.0.1:3300/api/travel`）。若后端端口或部署地址变化，只需改这一处。后端已开启 CORS，允许前端跨域访问。

### 6.4 推荐开发顺序

先启动后端（3300），再启动前端（5173），使用浏览器移动端模式（F12 → 设备模拟）体验效果最佳。

## 7. 常见问题（FAQ）

- **对话/规划报「大模型调用失败」**：检查 `.env` 中 `DeepSeek_API_KEY` 是否填写、密钥是否有效、账户余额是否充足。
- **前端请求一直失败**：确认后端已启动且端口为 3300；修改过 `request.js` 的 API_BASE 后需与后端实际地址一致。
- **收藏/历史/设置数据哪来的**：全部存在浏览器 localStorage（键名以 travel_ 开头），清除浏览器数据或设置页「清除全部数据」会将其删除，不依赖任何账号体系。
- **刷新详情页显示「缺少行程参数」**：正常情况下会回退读取 localStorage 中保存的最近一次规划参数；若从未提交过表单则会提示缺少参数，回首页重新填写即可。
- **为什么 Vant 提示组件不使用 showToast 函数**：在本项目 Vue 3.5 环境下，函数式 API 存在响应式兼容问题，因此统一改用模板组件 `<van-toast>` / `<van-dialog>`。

## 8. 相关资源

- Vue 3：https://cn.vuejs.org/
- Vite：https://vitejs.cn/
- Vant 4：https://vant.pro/vant/#/zh-CN
- Vue Router：https://router.vuejs.org/zh/
- Express：https://www.expressjs.com.cn/
- LangChain.js：https://js.langchain.com/
- DeepSeek 开放平台：https://platform.deepseek.com/
- SSE 流式响应参考：https://developer.mozilla.org/zh-CN/docs/Web/API/Server-sent_events
