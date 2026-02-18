# MMPEA 记忆管理 — 开发日志

## v3: 基础框架 (Story Bible)

实现了：
- 两层架构：故事圣经（全量注入）+ 记忆条目（关键词匹配检索）
- 自动提取、副API支持、批量初始化、自动隐藏、JSON容错

问题：故事圣经全量注入，token随剧情线性增长无上限。本质上和手动 Plot Summary 没有根本区别。

---

## v4: PageIndex 架构

### 核心变化

从"注入一切"变成"索引 + 按需检索 + 压缩"：

```
v3: Story Bible (全量注入, 无上限) + Memories (关键词匹配)
v4: Story Index (~400-600 tokens, 有上限) + Pages (工具调用检索) + 压缩
```

### 架构设计

- **Layer 1 - Story Index**: 始终注入，只有时间线+物品（角色移出索引）
- **Layer 2 - Story Pages**: 副API通过 function calling 选页面
- **Layer 3 - Character Dossiers**: 副API通过 function calling 选角色
- **渐进压缩**: L0(详细) → L1(摘要) → L2(归档删除)
- **时间线压缩**: 超过20行自动合并旧条目
- **数据迁移**: v1→v2 自动迁移

---

## v5: Embedding + MemGPT Agent

### Session 6-8: 架构升级

#### 独立存档系统

解决"切换聊天=失忆"：
- 记忆通过 `/api/files/upload` 保存为独立JSON文件
- 存档索引在 `extension_settings` 中维护
- 支持多槽位（主线/IF线/分支）
- 切换聊天时自动检测并提示加载同角色存档
- 提取后自动保存

#### 语义分类标签

每个故事页自动标注 1-3 个分类（emotional/relationship/intimate/promise/conflict/discovery/turning_point/daily），用于分类检索和 UI 展示。

#### 数据结构 v3→v4 迁移

新增 `categories: []` 和 `embeddings: {}` 字段。

### Session 9-12: 检索引擎重写（多次迭代）

#### 迭代过程

1. **初版**: supervisor + reasoning agent + build_memory_chain（两个独立agent + 记忆链工具）
   - 问题：6次API调用（1向量+1supervisor+3agent轮+1主模型），注入的还是散装页面dump

2. **第二版**: 合并supervisor到agent，去掉build_memory_chain
   - 问题：agent输出被丢弃，只提取page_id后dump原文。"又绕远了"

3. **最终版**: 单agent，输出即注入内容
   - Embedding top-K 候选（完整内容）放进 agent 输入
   - Agent 直接写因果链叙事，输出即 [记忆闪回] 注入内容
   - 不需要时输出 SKIP
   - 4个辅助工具（search_by_category/timerange/keyword + read_story_page），仅在候选不够时使用
   - 角色档案通过 embedding 匹配，不需要 agent 工具调用

#### 关键设计决策

| 决策 | 选择 | 原因 |
|------|------|------|
| agent数量 | 1个 | supervisor职能可以合并到agent |
| 注入格式 | agent写的叙事原文 | 比dump散装页面更连贯，LLM更容易理解 |
| 记忆链工具 | 去掉 | agent本身就能做因果推理 |
| 角色检索 | embedding匹配 | 比工具调用更快，不需要额外API轮次 |
| 候选页面 | 完整内容放入prompt | agent需要读内容才能写叙事 |
| 工具调用 | 保留但非必须 | "候选能推理出因果链时，不需要工具调用" |

#### 数据清洗

发现发给agent的 recentText 包含大量元数据（Tidal Memory注释、`<details>`状态块）。修复：
- char消息：只提取 `<content>` 标签内容
- 其他消息：剔除 `<!-- -->` 注释和 `<details>` 块
- 不截断内容

### Session 13: Bug修复 + 压缩系统改进

#### 存档切换bug

`updateBrowserUI()` 从未调用 `refreshSlotListUI()`，导致切换角色后存档列表不刷新。修复：在 `updateBrowserUI` 末尾加 `refreshSlotListUI()`。

#### 自动隐藏触发时机

`hideProcessedMessages()` 只在提取后触发。修复：
- 保持"提取后触发"的主流程（累积N条 → 提取 → hide）
- 新增切换聊天时触发（对已提取消息）
- 新增设置变更时触发

#### think标签清洗

agent（DeepSeek等）输出包含 `<think>...</think>` 推理块。在注入前剔除。

#### 压缩系统拆分

原来一个 `autoCompress` 开关控制所有压缩。拆成3个独立开关：
- **时间线压缩**（默认开）：超过20条自动合并
- **故事页压缩**（默认关）：故事页不占上下文，通常不需要压缩
- **归档日常页**（默认关）：只删 categories 仅含 "daily" 的页面，阈值50页，重要记忆永远不删

---

## v5.3.x: 提取系统重构 + Bug修复

### 消息级追踪系统

原来用水位线（`lastExtractedMessageId`）追踪提取进度，粗糙且不处理删除/隐藏场景。

新增 `extractedMsgDates: { [send_date]: true }` 精确追踪每条消息的提取状态。每条消息成功提取后用 `send_date` 打标。

**状态计数逻辑**（修复前后对比）：
- 修复前：有标记时只用日期计数，忽略水位线 → 混合数据下计数严重偏低
- 修复后：`Math.max(日期计数, 水位线)` → 兼容新旧数据

### 时间线保护

发现问题：LLM 提取本批消息后只输出本批的时间线，`data.timeline = result.timeline` 直接替换导致旧数据全丢。

新增 `mergeTimelines(old, new)` 函数：
- 解析 D-条目范围（处理 "D2-D4" 这类合并条目）
- 判断新时间线是否已完全覆盖旧时间线（新条目数 ≥ 旧的60% 且最大天数 ≥ 旧最大天数）
- 若已覆盖则信任新时间线；否则保留旧时间线中未被新时间线覆盖的条目

### 强制提取保护

发现问题：强制提取后 `autoSaveIfEnabled()` 自动保存，可能将空/少量记忆数据覆盖已有完整存档。

修复策略：
1. 强制提取前自动备份到 `{存档名}-备份`
2. 备份完成后恢复 `activeSlot`（`saveToSlot` 会修改它）
3. 如果当前记忆为空但存档存在，自动加载存档
4. 重新获取 `data` 引用（`loadFromSlot` 替换了 `ctx.chatMetadata.memoryManager` 对象引用）

### sourceDates 追踪

故事页新增 `sourceDates: string[]` 字段，记录生成该页面的批次消息的 `send_date` 列表。

修改 `applyExtractionResult(data, result, sourceDates = [])` 签名，全部6个调用点传入批次 `send_date`。

这是后续"孤立记忆检测"的基础数据。

---

## v5.4.0: UI 大升级 + 工具箱

### 小电视面板 Tab 化

原来面板只有一个页面（召回内容展示）。用户需要更丰富的交互入口。

将面板重构为三 Tab 结构：
- **召回**：保留原有召回内容展示
- **管理指令**：用户自定义 prompt 注入
- **工具箱**：记忆数据操作工具

Tab 懒加载（首次切入才执行 `renderDirectiveTab()` / `renderToolboxTab()`），避免无用渲染。

关闭面板时清空 `rtCommandMessages`（实时指令上下文），防止意外延续对话上下文浪费 token。

### 管理指令系统

需求：用户想给提取/召回/压缩过程加自定义要求，但现有提示词硬编码。

设计：
- 数据存在 `data.managerDirective: { global, extraction, recall, compression }` 中，绑定存档
- `getDirectiveSuffix(stage)` 按需返回附加文本，格式：`\n\n## 用户对记忆管理的特别要求\n...\n`
- 注入到 5 个提示词函数末尾（两个提取、一个召回、两个压缩）
- UI：四个 textarea + 直接保存 + LLM整理（让模型把乱写的指令重新分配到对应字段）

LLM整理实现：把四个字段内容拼接后请求 LLM 输出 `{global, extraction, recall, compression}` JSON，反填回 textarea。

### 故事页完整编辑

原来 `onEditPage` 只显示一个 content textarea。

重写后展开完整内联表单：title / day / date / content / keywords / categories(多选) / significance。

结构对齐已有的 `openEditKnownChar` / `openEditNpcChar` 内联编辑模式（隐藏展示区域，显示 `.mm-page-edit-panel`）。

修改页面卡片 HTML，添加 `<div class="mm-page-edit-panel" style="display:none"></div>`。

新增 `onAddPage()` 创建空白页并自动打开编辑表单。

### 工具箱实现

#### 记忆体检

核心逻辑：
1. 从 `extractedMsgDates` 取出所有已提取的 `send_date`
2. 从当前 `ctx.chat` 取出所有消息的 `send_date`（构成 chatDates Set）
3. 差集 = 孤立日期（在提取记录中但已不在聊天中）
4. 遍历故事页的 `sourceDates`，判断被孤立的比例 → 全部孤立/部分孤立

UI：checkbox 列表 + "删除选中" + "清理孤立日期"。

#### 快捷操作

`quickExtractRange(start, end)`：从 `ctx.chat` 提取第 start~end 条消息，调用 `buildExtractionPrompt` + `callLLM` + `applyExtractionResult` 完整流程。

`quickMarkExtracted(start, end)`：只打 `extractedMsgDates` 标记，不实际提取。用于跳过已人工处理或不需要记忆的消息段。

#### 实时指令（聊天式 agent）

设计：维护 `rtCommandMessages[]` 上下文，用 `callSecondaryApiChat` 支持 tool calling。

7个工具：
- `search_pages`：按关键词/分类/天数范围搜索故事页
- `edit_page_field`：修改故事页的单个字段
- `delete_page`：删除故事页
- `extract_range`：调用快捷提取（复用 `quickExtractRange`）
- `mark_extracted`：标记已提取
- `compress_page`：压缩单页（L0→L1）
- `rebuild_embeddings`：重建向量库

轮次控制：
- 最多10轮 tool calling
- 末轮禁用工具（`tools = []`），强制拿到文字总结
- 修复：不再误报"已达最大轮次限制"，改为"操作已执行完毕（共N轮）"

**关键 Bug 修复**：`callSecondaryApiChat` 返回的 `toolCalls` 是已解析格式 `{name, arguments}`，没有 `tc.id`。实时指令代码误用 `rawToolCalls` 需求的 `tc.function.name` 格式，导致崩溃。修复：始终用 `rawToolCalls` 迭代（有 `tc.id` 和原始 `tc.function`）。

### 副 API 失败报错

原来 `agentRetrieve` 调用失败只打 console warning，用户看不到任何提示，以为召回正常（实际是空的）。

修复：在调用层（不在 `agentRetrieve` 内部）加 try-catch，失败时弹红色 toast（10秒），显示错误信息并提示检查副API状态。失败后自动 fallback 到关键词检索，不中断主流程。

### 删楼提醒

注册 `MESSAGE_DELETED` 事件的第二个监听器 `onMessageDeleted`（原有的 `onChatEvent` 监听器触发重新提取，新监听器独立检测孤立记忆）。

检测逻辑同体检，只是只看孤立日期数量，不展开具体页面分析。发现孤立数据时弹 toast 引导用户去工具箱。

---

## 当前架构概览

```
用户发消息
  ↓
generate_interceptor
  ↓
Layer 1: Story Index 始终注入 (depth=9999, ~400-600 tokens)
  时间线 + 角色态度 + NPC列表 + 物品
  ↓
Layer 2+3: 统一检索
  ① Embedding → top-K候选 (含角色档案)
  ② 记忆Agent → 读候选 → 写叙事 or SKIP   [副API失败 → toast + fallback]
  ③ 关键词匹配 (兜底)
  ↓
注入 [记忆闪回] 叙事 + 角色档案 (depth=2)
  ↓
主模型生成回复 [附加管理指令: getDirectiveSuffix('recall')]
```

```
每N条消息
  ↓
提取 → 更新时间线(merge)/角色/物品/故事页(+sourceDates)
  [附加管理指令: getDirectiveSuffix('extraction')]
  ↓
压缩周期 (按各自开关)
  [附加管理指令: getDirectiveSuffix('compression')]
  ↓
自动保存存档
  ↓
自动隐藏旧消息

用户删楼
  ↓
onMessageDeleted → 检测孤立日期 → toast提醒（如有）
```

小电视面板：
```
召回 Tab  ←→  管理指令 Tab  ←→  工具箱 Tab
  ↑                                  ↑
召回内容展示          记忆体检 / 快捷操作 / 实时指令agent
```

---

## 技术栈

- 纯浏览器端 JS（SillyTavern 第三方扩展）
- 副API通过 ST 服务端代理（避免CORS）
- Embedding 直接浏览器 fetch（中转站支持CORS）
- 向量存储在本地 chatMetadata/存档文件
- 余弦相似度纯JS计算
- 无需修改 SillyTavern 核心代码
