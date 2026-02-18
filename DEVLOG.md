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
  ② 记忆Agent → 读候选 → 写叙事 or SKIP
  ③ 关键词匹配 (兜底)
  ↓
注入 [记忆闪回] 叙事 + 角色档案 (depth=2)
  ↓
主模型生成回复
```

```
每N条消息
  ↓
提取 → 更新时间线/角色/物品/故事页
  ↓
压缩周期 (按各自开关)
  ↓
自动保存存档
  ↓
自动隐藏旧消息
```

---

## 技术栈

- 纯浏览器端 JS（SillyTavern 第三方扩展）
- 副API通过 ST 服务端代理（避免CORS）
- Embedding 直接浏览器 fetch（中转站支持CORS）
- 向量存储在本地 chatMetadata/存档文件
- 余弦相似度纯JS计算
- 无需修改 SillyTavern 核心代码
