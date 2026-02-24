---
name: su_xing_fitness_app
overview: 为“塑型”App 设计一个基于 React Native 的跨平台移动应用架构，支持本地/云端存储、基于 OpenAI 兼容接口的大模型生成训练计划，以及多界面记录与可视化。
todos:
  - id: init-project
    content: 初始化 React Native + TypeScript 项目并配置导航和基础主题
    status: completed
  - id: data-model-storage
    content: 设计并实现本地数据模型和存储层（SQLite + 偏好存储）
    status: completed
  - id: llm-integration
    content: 封装 OpenAI 兼容大模型客户端和计划生成服务
    status: in_progress
  - id: ui-screens
    content: 实现主页、计划/历史、设置三个核心界面并串联业务逻辑
    status: pending
  - id: cloud-sync-config
    content: 实现云端同步配置（环境变量 + 设置页）和基础同步逻辑
    status: pending
isProject: false
---

## 总体目标

- **目标**: 使用 React Native（TypeScript）实现一款名为“塑型”的移动 App，支持：
  - 用户填写身高、体重、目标（增重/减重/维持/运动目标），由大模型生成个性化运动/体重管理计划。
  - 用户每日按计划打卡，记录每一项完成情况。
  - 数据可选择保存到本地（SQLite/AsyncStorage）或云端（通过可配置数据库连接的后端接口）。
  - 用户界面中可配置：是否启用云端、云端接口地址/密钥、大模型 API Key、模型名称等。
  - 2–3 个核心 Tab 页面：主页打卡+图表、计划/历史、用户设置。

---

## 架构与技术栈设计

- **前端技术栈**
  - 使用 `React Native` + `TypeScript`。
  - 路由与多页面使用 `@react-navigation`（Bottom Tab + Stack）。
  - 状态管理：优先使用 React Context + hooks，如果复杂度提升可预留升级到 Zustand/Redux 的空间。
- **本地存储方案**
  - 选择 `react-native-mmkv` 或 `AsyncStorage` 存储轻量配置（用户偏好、本地/云端开关、API Key 等）。
  - 选择 `react-native-sqlite-storage` 或 `expo-sqlite` 存储结构化数据（计划、每日打卡记录、体重记录）。
- **云端接口抽象**
  - 前端不直接连接数据库，而是通过 HTTP REST API（例如 `https://api.example.com`）与后端交互。
  - 云端地址、API Token 等通过环境变量注入（如 `.env` 文件，使用 `react-native-dotenv` 或 Expo 的 `app.config.js` 环境配置）。
  - 在代码中封装 `ApiClient`：根据配置决定是否调用云端，失败时自动降级为本地存储或做清晰的错误提示。
- **大模型调用封装**
  - 使用 OpenAI 兼容接口（如 `/v1/chat/completions`）。
  - 在前端封装 `LLMClient`：
    - 从环境变量 + 用户设置中读取：`LLM_BASE_URL`、`LLM_API_KEY`、`LLM_MODEL`。
    - 提供 `generatePlan(userProfile, goal)` 方法，返回结构化计划对象（每天/每周的运动/饮食建议与说明）。
  - 预定义提示模板，保证返回 JSON 结构可解析，前端根据此结构渲染计划和打卡项。

---

## 数据模型与业务流程设计

- **核心数据模型**
  - `UserProfile`：性别、身高、当前体重、目标体重、目标类型（增重/减重/维持/运动能力），时间周期等。
  - `Plan`：计划 ID、生成时间、目标信息、计划周期（天/周）、计划项列表。
  - `PlanItem`：某一天/某一时间段的任务，如“有氧 30 分钟”、“力量训练 A 组”等，包含预计消耗、大致难度等。
  - `DailyLog`：某天的整体记录（日期、体重、主观疲劳等）。
  - `TaskLog`：针对 `PlanItem` 的完成情况（完成/部分完成/未完成、备注、打卡时间）。
- **主要业务流程**
  - **新用户首次使用**：
    - 打开 App → 引导填写基础信息 → 选择目标 → 点击“生成计划” → 调用 LLM 接口 → 展示生成的计划 → 用户确认后保存为当前激活计划并落库（本地或云端）。
  - **每日打卡流程**：
    - 主页展示当天计划项列表与体重曲线 → 用户勾选完成项、输入体重/备注 → 点击“保存/同步”。
    - 若开启云端：先写入本地，再尝试同步云端，失败时在 UI 中展示同步状态。
  - **历史与统计查看**：
    - 通过图表展示体重变化曲线、完成率（周/月维度），可切换计划查看历史数据。
- **界面设计（3 个 Tab）**
  - **Tab1：主页（打卡 & 概览）**
    - 顶部：当前体重、目标体重、预计还需多少天（简单估算）。
    - 中部：当天计划列表（每个 `PlanItem` 可展开查看说明），带勾选/完成状态。
    - 底部：
      - 简单折线图/柱状图：最近 7/30 天体重变化、完成率。
      - 按钮：“记录体重”“生成新计划”。
  - **Tab2：计划 & 历史**
    - 计划列表：当前计划 + 历史计划（可查看详情但不可编辑）。
    - 计划详情：展示大模型生成的分天/分周安排、文字说明。
    - 操作：重新生成计划（基于当前最新体重/目标）、复制旧计划为新计划。
  - **Tab3：用户 & 设置（最右侧）**
    - 基础信息：身高、体重、性别、年龄等（可编辑）。
    - 存储策略：
      - 开关“启用云端同步”。
      - 云端配置字段：`API Base URL`、`API Token` 或 `Project ID`（存储到本地安全存储）。
    - 大模型配置：
      - `LLM Base URL`
      - `LLM API Key`
      - `Model Name`（如 `gpt-4.1`/`deepseek-chat` 等）
    - 数据管理：
      - 导出/导入本地数据（JSON 文件），为换机或调试预留。

---

## 环境变量与配置设计

- **移动端侧环境变量**（开发/打包阶段）
  - `.env` 或 `app.config.js` 中配置：
    - `API_BASE_URL`：后端服务地址，用于云端存储同步。
    - `LLM_BASE_URL`：大模型兼容接口地址。
    - `DEFAULT_LLM_MODEL`：默认模型名。
  - 使用配置库：
    - `react-native-config` 或 Expo `expo-constants` + 自定义 `config.ts`。
- **运行时用户可覆盖**
  - 在设置页中，用户可以覆盖环境中的默认值（如更换模型服务商），优先级：用户设置 > 编译时默认值。
- **安全考虑**
  - 不在仓库中提交真实的 API Key，仅示例变量名和 `.env.example`。
  - 文档说明如何通过环境变量/CI 注入真实密钥。

---

## 模块划分与关键文件

- **入口与导航**
  - `[src/App.tsx]`：应用入口、导航容器。
  - `[src/navigation/RootNavigator.tsx]`：Tab 导航 + Stack 导航配置。
- **上下文与状态管理**
  - `[src/context/UserSettingsContext.tsx]`：存储用户基础信息、存储策略、API 配置等。
  - `[src/context/PlanContext.tsx]`：当前计划、历史计划、打卡状态，提供业务操作函数。
- **存储层**
  - `[src/storage/localDb.ts]`：SQLite 初始化与 CRUD 封装（计划、日志）。
  - `[src/storage/preferences.ts]`：MMKV/AsyncStorage 封装用户偏好与敏感配置。
- **网络与服务封装**
  - `[src/services/apiClient.ts]`：HTTP 客户端，包含云端同步逻辑、错误处理与重试策略（简化版）。
  - `[src/services/llmClient.ts]`：OpenAI 兼容大模型调用封装（接收 prompt 参数，返回计划对象）。
  - `[src/services/planGenerator.ts]`：构造 prompt、调用 `llmClient`，并将 LLM 响应转换为内部 `Plan` 模型。
- **UI 界面组件**
  - `[src/screens/HomeScreen.tsx]`：主页打卡与图表。
  - `[src/screens/PlansScreen.tsx]`：计划与历史记录。
  - `[src/screens/SettingsScreen.tsx]`：用户 & 配置界面。
  - `[src/components/PlanItemCard.tsx]`：单个计划项的展示与勾选。
  - `[src/components/Charts/ProgressChart.tsx]`：体重/完成率图表（基于 `react-native-svg-charts` 或 `victory-native`）。

---

## LLM 计划生成的 Prompt 与结构约定

- **Prompt 设计要点**
  - 明确告知模型：
    - 用户目标（增重/减重/维持/提升体能）。
    - 用户当前身体数据与训练经验。
    - 期望计划周期（例如 8 周）。
    - 输出必须为严格 JSON，不返回多余说明文字。
  - 例如要求 JSON 结构：`{ "weeks": [...], "summary": "..." }`，其中 `weeks` 内包含按天列出的 `tasks`。
- **前端解析逻辑**
  - `planGenerator.ts` 中负责校验与兜底：
    - 若解析 JSON 失败，提示用户“生成失败，请重试”；
    - 若字段缺失，使用默认值或简化展示。

---

## UX 细节与可扩展功能建议

- **交互细节**
  - 打卡动作提供轻微动画/反馈，让用户有成就感。
  - 在主页顶部展示简短激励语句（可由 LLM 生成或内置文案）。
  - 离线可用：即使没网，仍可本地记录，待联网后自动提示可同步。
- **可扩展功能（后续迭代）**
  - 加入饮食记录与热量估算。
  - 引入计划难度调整（如“强度+10%”重新生成计划）。
  - 社区/分享功能：导出一周成果截图。

---

## 简要开发步骤

1. 初始化 React Native + TS 项目，配置导航、环境变量读取和基础主题。
2. 实现 `UserSettingsContext` 和 `PlanContext`，并打通设置页的基础信息编辑（先用本地存储）。
3. 搭建本地数据库层（SQLite）和偏好存储（MMKV/AsyncStorage），实现计划与打卡记录的基本 CRUD。
4. 实现 `llmClient` 与 `planGenerator`，使用假数据或测试 API Key 调通大模型生成计划的完整流程。
5. 完成 3 个 Tab 界面：主页打卡 + 图表、计划/历史、设置；联通上下文与存储。
6. 增加云端同步开关和配置字段，封装 `apiClient` 与同步逻辑（先以 Mock/假服务器为例）。
7. 打磨 UI/UX、增加错误提示和加载状态，准备打包/发布。

