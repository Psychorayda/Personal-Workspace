# 语音打卡助手 设计文档

日期：2026-08-17
范围：`workbench-desktop.html` 的「猫咪日常」(checkin) 模块
目标：通过语音交互完成 新增 / 删除 / 完成 打卡记录，最小代码增量、最小成本、架构预留扩展

## 1. 背景与约束

- 工作台为单文件静态 HTML，数据在 localStorage（`cat-press-workbench-v1`），无后端
- checkin 数据结构：`data.checkin = [{id, title, log:{ "YYYY-MM-DD": true }}]`
- 所有增删改最终都收敛为「改 `data.checkin` → `persist()` 保存并刷新」
- 用户环境：Chrome / Edge 桌面浏览器 → 可使用免费的 Web Speech API（`webkitSpeechRecognition`）
- 决策记录：
  - LLM 选型：DeepSeek（默认），provider 做成可换配置（Gemini 免费档为 0 元备选）
  - 范围：只做「猫咪日常」，但执行器写成通用结构，未来扩展到其他模块只需加配置
  - 删除必须二次确认（用户明确要求）；新增、完成为即时执行
  - API Key 存 localStorage，仅发往 provider 官方接口；单用户本地使用，接受该风险

## 2. 架构与数据流

```
[悬浮麦克风按钮] → webkitSpeechRecognition（zh-CN，单次识别）
  → 识别文本 + 当前 checkin 条目清单 → POST DeepSeek /chat/completions
  → 模型返回纯 JSON：{"action":"add|delete|complete","title":"条目名"}
  → VoiceExec 执行器：模糊匹配条目 → 操作 data.checkin
      - add / complete → 立即 persist()
      - delete → 先弹确认条，用户点「确认」后才 persist()
  → toast 反馈（编辑部口吻）
```

## 3. 组件设计（全部为新增代码，现有代码零改动）

集中放置在文件末尾：一个 `<style>` 块（约 40 行）+ 一个 `<script>` 块（约 150 行）。

### 3.1 VOICE_CONFIG

```js
const VOICE_CONFIG = {
  baseURL: "https://api.deepseek.com/chat/completions",
  model: "deepseek-chat",
  keyStoreKey: "cat-press-llm-key",   // localStorage 里的 Key 位
  moduleKey: "checkin",               // 本期只接猫咪日常；未来扩展改这里+动作表
};
```

### 3.2 UI

- 悬浮麦克风按钮：fixed 右下角，橘黄（`var(--module-3)`）圆形，使用 Lucide `mic` 图标（新增进 ICONS）
- 录音中状态：墨绿（`var(--accent)`）脉冲描边动画
- 设置弹窗：首次使用或无 Key 时弹出，输入 Key 存 localStorage；复用现有 modal 样式
- 删除确认：**复用模板已内置的 `confirmDelete("checkin", id)` 弹窗**（规划阶段确认存在），零新增 UI
- toast：模板未内置 toast，新增一个轻量 toast 函数（底部居中淡入淡出，复用 token 配色）

### 3.3 函数

| 函数 | 职责 |
|---|---|
| `listen()` | 封装语音识别；不支持/无权限时给出明确提示 |
| `parseCommand(text)` | 调 LLM；system prompt 注入当前条目清单，要求只输出 JSON；解析失败返回 null |
| `execCommand(cmd)` | 分发三个动作；delete 走确认条流程 |
| `fuzzyFind(title)` | 包含匹配 + 编辑距离，返回唯一最佳条目或 null |
| `ensureKey()` | 无 Key 时弹设置窗，返回是否有 Key |

### 3.4 LLM Prompt（精简版）

system：
```
你是「猫咪生活报」打卡助手的指令解析器。当前打卡条目：{titles}。
把用户的话解析为 JSON：{"action":"add|delete|complete","title":"..."}。
- add：title 用用户说的新条目名
- delete/complete：title 用清单里最匹配的条目标题
只输出 JSON，不要解释。无法理解时输出 {"action":"unknown"}。
```

### 3.5 动作映射

| action | 操作 | 确认 |
|---|---|---|
| add | `data.checkin.push({id:Date.now(), title, log:{}})` | 否 |
| complete | `x.log = x.log||{}; x.log[today()] = true` | 否 |
| delete | 从 `data.checkin` 移除匹配项 | **是（确认条）** |

执行后统一 `persist()`。

## 4. 错误处理

| 场景 | 行为 |
|---|---|
| 浏览器不支持语音识别 | 点击按钮提示「请使用 Chrome / Edge」 |
| 无 API Key | 弹设置窗引导填写 |
| 识别无结果 / LLM 返回 unknown / JSON 解析失败 | toast「没听清，请再说一次」 |
| delete/complete 找不到条目 | toast 列出当前条目名 |
| 网络或 API 报错（401/429/超时） | toast 提示检查网络或 Key 余额 |
| delete 匹配多条 | 取相似度最高者进确认条；确认条上显示完整标题 |

## 5. 成本

- 语音识别：0 元（Chrome 内置）
- DeepSeek：每条指令约 300 tokens 输入 + 50 输出 ≈ 0.0004 元；每天 30 条，月成本 < 0.5 元
- 充值 1 元约可用 2~3 个月；换 Gemini 免费档则 0 元（需可访问 Google）

## 6. 测试（手动验收清单）

1. 「新增一个梳毛打卡」→ 列表出现新条目
2. 「喝水打卡完成了」→ 「喝够 8 杯水」今日已勾选，连续天数 +1
3. 「把晒太阳删掉」→ 弹确认条，点确认后条目消失；点取消不删
4. 模糊说法：「水喝完了」→ 命中「喝够 8 杯水」
5. 不存在条目：「把遛弯删掉」→ toast 列出当前条目
6. 清空 Key 后点麦克风 → 弹设置窗
7. 全程控制台无报错
