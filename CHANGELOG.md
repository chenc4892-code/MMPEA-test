# MMPEA 记忆管理 更新日志

## v5.5.1 (2026-02-19)

### UI/UX 改进

#### 面板遮罩与移动端适配
- 面板弹出时增加半透明暗色遮罩 + 毛玻璃模糊（`backdrop-filter: blur(4px)`），点击遮罩关闭面板
- 移动端（屏幕宽度 <500px）面板自动居中显示，宽度为屏幕宽 -24px，高度最大 480px

#### 初始化弹窗重做
- 原 `prompt()` 文本输入替换为自定义 HTML 弹窗
- 起始/结束改为两个独立 `type="number"` 输入框，预填 0 和 chatLen-1
- 消除手动输入格式导致的解析失败风险

#### 记忆体检增强：孤儿页面管理
- 健康检查不再只显示孤立日期数量，改为展示受影响的故事页列表
- 每个故事页标注状态标签：**全孤立**（红色，所有源日期均已失效）/ **部分孤立**（琥珀色，部分源日期失效）
- 支持复选框选择 + 全选按钮
- **删除选中页面**：从 `data.pages` 中移除选中页面及其 Embedding 向量
- **清理孤立日期**：从 `extractedMsgDates` 中移除已不存在的消息日期记录
- 操作完成后自动重新运行体检刷新结果

#### 未提取标记图标修正
- 替换手绘 inline SVG，改用扩展目录中实际的 `robot-svgrepo-com.svg` 路径数据
- SVG `fill` 改为 `currentColor`，跟随 CSS 颜色控制

#### 设置面板 UI 修复（模块拆分后回归修复）
- 知识卡片（已知角色/NPC/物品）布局与样式还原
- 故事页编辑表单：隐藏预览区、恢复日期字段
- 标签样式：分类标签（catTags）彩色药丸、关键词标签（kwTags）灰色药丸
- 故事页卡片布局对齐原始设计稿
- 实时指令聊天区域高度自适应

#### 移动端悬浮球修复
- 重新挂载 Popover API（`popover="manual"`）确保 top layer 渲染
- 修复触屏拖拽事件（touchstart/touchmove/touchend）
- Lottie 动画加载失败时降级为静态图标
- 拖拽位置边界约束（不超出视口）

### Bug Fixes

#### 删楼后自动隐藏范围未更新
- **问题**: 用户删除消息后，`keepRecentMessages` 对应的可见消息范围未重新计算，导致本应可见的消息仍处于隐藏状态（例如删了 6 楼后第 60 楼还是隐藏的）
- **修复**: 新增 `recalculateHideRange()` 函数，在 `onMessageDeleted` 事件中自动比较新旧隐藏边界，若新边界更小则调用 `hideChatMessageRange(unhideFrom, oldBoundary, true)` 取消隐藏差额区间

---

## v5.5.0 (2026-02-18)

### 架构重构：模块化拆分

将原 ~6000 行单体 `index.js` 拆分为 15 个职责单一的 ES 模块，依赖关系为有向无环图（DAG），无循环依赖。

#### 模块清单

| 模块 | 职责 |
|------|------|
| `src/constants.js` | 纯常量（零依赖） |
| `src/utils.js` | 工具函数（escapeHtml、generateId、cosineSimilarity 等） |
| `src/mood.js` | Lottie 动画心情系统 |
| `src/data.js` | 数据层：设置、记忆 CRUD、迁移链 |
| `src/auth.js` | 授权验证（SHA-256 授权码校验） |
| `src/api.js` | LLM API 层（主/副 API、工具调用） |
| `src/save.js` | 存档系统（saveToSlot / loadFromSlot） |
| `src/embedding.js` | Embedding 向量检索系统 |
| `src/formatting.js` | 提示词构建与记忆格式化 |
| `src/compression.js` | 渐进式压缩引擎 |
| `src/extraction.js` | 记忆提取引擎（UI 回调注入模式） |
| `src/retrieval.js` | Agent 检索引擎（MemGPT 工具调用） |
| `src/ui-browser.js` | 设置面板 UI（updateBrowserUI、CRUD 操作） |
| `src/ui-fab.js` | 悬浮球、召回面板、工具箱、批量初始化 |
| `src/commands.js` | /mm-* 斜杠命令注册 |

`index.js` 精简为薄入口：jQuery 初始化、auth 门控、事件绑定、回调注入。

#### 关键架构决策

- **回调注入模式**：`setExtractionUI(callbacks)` 将 `updateBrowserUI` / `updateInitProgressUI` 等 UI 函数注入提取引擎，避免 extraction → ui-browser/ui-fab 的循环依赖
- **状态 getter 模式**：retrieval 模块通过 `getLastRecalledPages()` / `getLastNarrative()` 等 getter 暴露状态，不直接共享变量
- **UI 职责分离**：`safeCompress()` / `retrieveMemories()` 不再自行调用 `updateBrowserUI()`，由调用方决定 UI 刷新时机
- **全局拦截器包装**：`window['memoryManager_retrieveMemories']` 在 index.js 封装 `retrieveMemories` + `updateRecallFab`，保持检索后 FAB 自动更新

---

## v5.4.1 (2026-02-18)

### Performance Optimizations

#### escapeHtml 纯字符串替换
- 原实现每次调用创建一个 `document.createElement('div')` DOM 节点，在 UI 渲染时（每张卡片调用 5-8 次 escapeHtml）开销显著
- 改为纯字符串 `.replace()` 链，零 DOM 操作，性能提升约 10 倍

#### updateBrowserUI 分区更新
- 原实现每次调用都重建**全部** 9 个 UI 区域（timeline / knownChars / npcChars / items / pageStats / pageList / embedding / status / slots），包括完整的 innerHTML 重写和事件监听重绑
- 新增 `sections` 参数，支持按需只更新特定区域
- 编辑/删除/新增单个实体（角色/NPC/物品/故事页/时间线）时，只更新对应的 1-2 个区域，避免全量 DOM 重建
- 无参调用保持向后兼容（更新全部区域）
- 内部拆分为 `_renderKnownChars()`、`_renderNpcChars()`、`_renderItems()`、`_renderPageList()` 四个独立渲染函数

#### updateUnextractedBadges 定向更新
- 原实现每次调用都 `querySelectorAll('.mes[mesid]')` 全量扫描所有聊天消息 DOM 节点
- 新增 `targetMesId` 参数，`onMessageRendered` 时只处理单条消息的 badge，避免 O(n) 全量扫描
- 全量扫描仅在 `onChatChanged` 等真正需要时执行

#### data.pages 排序优化
- `data.pages.sort()` 改为 `[...data.pages].sort()` 避免原地变异，防止其他代码依赖的数组顺序被意外修改
- 排序比较器内的 `parseInt(day.replace(/\D/g, ''))` 改为 Map 缓存，每个 page 只解析一次

#### FAB 拖拽监听器按需绑定
- 原实现在 FAB 创建时就将 `mousemove` / `mouseup` / `touchmove` / `touchend` 四个监听器永久挂载到 `document`，每次鼠标移动都会触发回调（即使未拖拽）
- 改为在 `mousedown` / `touchstart` 时才绑定 move/end 监听，拖拽结束后立即 `removeEventListener`，非拖拽状态下零开销

---

## v5.4.0 (2026-02-18)

### New Features

#### 小电视面板 Tab 化
- 原单一"召回"面板拆分为三个 Tab：**召回 / 管理指令 / 工具箱**
- Tab 懒加载（首次切入才渲染内容）
- 关闭面板或离开工具箱 Tab 时自动清空实时指令上下文（节省 API 消耗）

#### 管理指令 Tab（新功能）
- 支持按环节填写用户指令：**全局 / 提取 / 召回 / 压缩**
- 各环节带有极简说明（写给完全不懂的人看的那种）
- "直接保存"与"让管理员整理"（LLM重新格式化分配到各字段）两种保存方式
- 指令自动注入到对应 prompt 函数末尾（5个提示词函数全覆盖）
- 数据存储在 `managerDirective` 字段，绑定到当前存档

#### 故事页完整编辑表单（新功能）
- 点击"编辑"展开完整内联表单，可编辑所有字段：标题、天数（D1）、日期（251017）、内容、关键词（逗号分隔）、分类标签（多选）、重要程度
- 原来只能编辑 content 文本框，现在全字段可改
- 保存后如启用了 Embedding 自动重新建向量

#### 手动新增故事页（新功能）
- 故事页列表底部增加"+ 新增故事页"按钮
- 点击后创建空白页并自动打开编辑表单

#### 工具箱 Tab（新功能）

**记忆体检**
- 比对 `extractedMsgDates` 与当前聊天记录，找出孤立（已删消息）的提取日期
- 显示受影响的故事页及其状态（全孤立=建议删除，部分孤立=建议审查）
- 支持选择性删除 + 清理孤立日期记录

**快捷操作**
- 提取指定范围（输入起止编号 → 调用完整提取 pipeline）
- 标记已提取（只打标不提取，用于跳过不需要记忆的消息段）
- 重建向量库（快捷入口，复用设置面板的同名功能）

**实时指令**（聊天式 agent）
- 和记忆管理员用自然语言对话，直接操作记忆数据
- 支持 7 个工具：搜索故事页、编辑字段、删除页面、提取范围、标记已提取、压缩页面、重建向量
- 最多 10 轮 tool calling，末轮禁用工具强制返回文字总结
- 离开 Tab 或关闭面板自动清空上下文

#### 删楼提醒 Toast（新功能）
- 用户删除消息后自动检测是否有孤立提取记录
- 发现孤立数据时弹出 toast，引导使用工具箱→记忆体检进行清理

### Bug Fixes
- **实时指令 tool call 崩溃**: `callSecondaryApiChat` 返回的 `toolCalls` 已解析为 `{name, arguments}`，但代码按原始 OpenAI 格式 `tc.function.name` 访问导致崩溃。修复：改用 `rawToolCalls` 迭代，保留 `tc.id` 和原始格式
- **副 API 不可用无反馈**: `agentRetrieve` 调用失败时只打日志，召回静默失败。修复：加 try-catch，失败时弹出红色 toast，显示错误信息并提示检查 API 状态，自动 fallback 到关键词检索

---

## v5.3.1 (2026-02-18)

### Bug Fixes
- **时间线被新提取覆盖**: `data.timeline = result.timeline` 直接替换，LLM 只输出本批内容的时间线时旧数据全部丢失。修复：新增 `mergeTimelines()` 函数，解析 D-条目范围，智能判断新旧时间线的覆盖程度，只覆盖已被新时间线包含的旧条目
- **强制提取覆盖存档**: 强制提取后 `autoSaveIfEnabled()` 自动保存，导致当前记忆数据（可能为空）直接覆盖已有存档。修复：强制提取前自动备份到 `{存档名}-备份`，如果当前记忆为空且存档存在则自动加载，确保不丢数据
- **故事页缺少 sourceDates**: 正常提取和强制提取创建的故事页 `sourceDates` 始终为空，无法追踪来源消息。修复：`applyExtractionResult` 新增 `sourceDates` 参数，全部 6 个调用点均传入对应批次的 `send_date` 列表
- **处理计数显示错误**: 存在 `extractedMsgDates` 标记时只用日期计数，忽略水位线；无标记时只用水位线。导致新旧混合数据下计数严重偏低（如"已处理4条"而实际已处理上千条）。修复：取日期计数和水位线计数的较大值

---

## v5.3.0 (2026-02-17)

### New Features
- **消息级提取标记系统** (`extractedMsgDates`): 每条消息提取成功后用 `send_date` 打标，精确追踪哪些消息已提取、哪些遗漏。替代原来仅靠水位线 (`lastExtractedMessageId`) 的粗略判断
- **强制提取重写**: 强制提取不再走水位线逻辑，而是扫描全部聊天消息（包括已隐藏的），找出所有未打标的消息进行分批提取。初始化失败遗漏的消息也能被捞回
- **强制提取自动重试**: 失败批次会自动重试一轮，与初始化的重试逻辑对齐
- **强制提取进度条**: 复用初始化进度 UI，显示分批进度和重试状态
- **精确待处理计数**: "未处理消息"数量改为基于 `extractedMsgDates` 精确统计，包含已隐藏的消息
- **初始化范围选择**: 初始化弹窗显示全部消息总数（含隐藏），用户可指定提取的消息范围
- **未提取消息标记图标**: 未被记忆管理器提取的消息在用户名旁显示一个小机器人图标（🤖），提取成功后自动消失。让用户一眼看出哪些消息还没被处理

### Bug Fixes
- **iOS 召回面板变扁**: Popover API 的浏览器默认样式 (`height: fit-content`) 导致面板在 iOS Safari 上塌缩。修复：在 `[popover]` 重置中增加 `height: auto`、`min-height: 200px`、`overflow: visible`
- **隐藏消息不可见**: `hideChatMessageRange` 将消息标记为 `is_system=true`，导致初始化/强制提取/计数用 `!is_system` 过滤时排除了这些消息。修复：在初始化、强制提取和计数中改用 `m.mes` 作为过滤条件，不再排除已隐藏的消息
- **旧数据兼容**: 旧数据 `extractedMsgDates` 从空开始，不做假设性迁移。待处理计数在无标记时 fallback 到水位线逻辑。用户可通过强制提取重新扫描全部消息补上遗漏

### Other
- 添加 `.gitignore`，排除 `auth-codes.txt` 和 `generate-auth-codes.cjs`，防止授权码明文和生成器泄露到仓库

---

## v5.2.4 (2026-02-17)

### Bug Fixes
- **初始化重试按钮不显示**: `updateInitProgressUI` 中重试按钮的渲染条件要求 `!initializationInProgress`，但调用时该标志仍为 `true`（在 `finally` 中才重置），导致按钮永远不会出现。修复：在 `finally` 块中重置标志后重新渲染 UI
- **强制提取无反馈**: 点击强制提取后没有任何提示或进度显示，用户不知道请求已发出会反复点击。修复：添加开始/完成/失败的 toast 提示，以及各种边界情况的反馈（正在提取中、无聊天记录、消息发送中）
- **强制压缩虚假成功**: 压缩条件不满足时实际没有执行任何操作，但仍显示"压缩完成"。修复：`runCompressionCycle` 返回实际工作量统计，根据结果显示具体信息（"3 页压缩，2 页归档"）或"当前没有需要压缩的内容"

---

## v5.2.2 (2026-02-17)

### Bug Fixes
- **NPC/物品消失修复**: 提取结果改用合并逻辑（merge），不再整体替换角色和物品数组。已有NPC不会因为某次提取未输出而丢失
- **重投骰记忆污染修复**: 新增提取缓冲区，autoHide开启时跳过最近N-2条消息，避免用户重投骰前旧回复被写入记忆
- **悬浮球/召回面板被主题遮挡修复**（三轮迭代）:
  - 第一轮: z-index 提升至 99990/99991 + CSS `!important` → 部分主题仍被挡
  - 第二轮: JS 内联 `setProperty(..., 'important')` → 部分主题仍被挡
  - 第三轮（最终方案）: **Popover API** (`popover="manual"`) 将面板渲染到浏览器 top layer，彻底免疫所有 z-index/stacking context 问题
- **面板样式**: 改回半透明白底 + 黑字 + 磨砂玻璃效果，不再依赖主题CSS变量

### New Features
- **知识卡片式布局**: 已知角色态度、NPC档案、物品信息改为卡片式展示，显示全部字段（外貌/性格/态度/状态等）
- **YYMMDD日期标记**: 故事页新增 `date` 字段，提取时从消息状态栏/时间描述中提取具体日期（格式如 "251017"）
- **初始化批次重试**: 初始化失败的批次会被记录，提供"重试失败批次"按钮
- **强制提取自动分批**: 待处理消息超过25条时自动按每批20条分批提取，避免单次请求过大导致失败

---

## 技术笔记：SillyTavern 扩展中浮动UI的渲染层级问题

### 问题
扩展创建的 `position: fixed` 浮动元素（悬浮球、弹出面板）被某些主题遮挡，用户看不到或无法点击。

### 踩过的坑（从低到高）

#### 1. 提高 z-index（无效）
```css
.my-panel { z-index: 99991; }
```
**为什么不够**: SillyTavern 的弹窗系统用原生 `<dialog>` + `.showModal()`，dialog 进入浏览器的 **top layer**，这是一个独立于 z-index 的渲染层。任何 z-index 值（哪怕是 999999999）都在 top layer 之下。

#### 2. CSS `!important`（无效）
```css
.my-panel { z-index: 99991 !important; display: flex !important; }
```
**为什么不够**: 主题的 `custom_css` 如果用了更高特异性的选择器 + `!important`，会覆盖扩展的样式表规则。

#### 3. JS 内联 `setProperty(..., 'important')`（部分有效）
```javascript
el.style.setProperty('display', 'flex', 'important');
el.style.setProperty('z-index', '99991', 'important');
```
**进步**: 内联 `!important` 是 CSS 优先级最高的，任何样式表规则都无法覆盖。
**仍然不够**: 解决了 CSS 覆盖问题，但没解决 top layer 问题。如果 ST 的 dialog 系统或其他 top layer 元素在前，面板仍然被挡。

#### 4. Popover API（最终方案 ✓）
```javascript
panel.setAttribute('popover', 'manual');
// 显示时:
panel.showPopover();  // 进入 top layer
panel.style.setProperty('display', 'flex', 'important');
// 隐藏时:
panel.hidePopover();  // 离开 top layer
panel.style.setProperty('display', 'none', 'important');
```
**为什么有效**: `popover="manual"` + `.showPopover()` 将元素放入浏览器的 **top layer**，与 `<dialog>` 同级。Top layer 中最后加入的元素在最上面。不需要 z-index，不受任何 CSS 影响。

### CSS 优先级速查（从低到高）

```
普通样式表规则          →  可被更高特异性覆盖
样式表 !important       →  可被内联 !important 覆盖
内联 style="..."        →  可被内联 !important 覆盖
内联 setProperty+important →  CSS 层面的最高优先级
──────────────────────────────────────────────
top layer (dialog/popover)  →  独立渲染层，z-index 无关
```

### 浮动UI最佳实践（SillyTavern 扩展）

1. **面板/弹窗**: 用 `popover="manual"` + `showPopover()`/`hidePopover()`
2. **定位**: 仍然用 `position: fixed` + 内联 `setProperty` 设置 top/left/right/bottom
3. **样式**: 不要依赖主题 CSS 变量（`--SmartThemeXxx`），用硬编码颜色，因为面板在 top layer 里跟主题是隔离的
4. **降级**: 检测 `typeof HTMLElement.prototype.showPopover === 'function'`，不支持时回退到 z-index 方案
5. **Popover 默认样式重置**: 浏览器给 `[popover]` 元素有默认样式（margin/padding/border/inset），需要重置：
   ```css
   .my-panel[popover] { margin: 0; padding: 0; inset: auto; }
   ```

### 浏览器兼容性

Popover API 支持: Chrome 114+, Edge 114+, Firefox 125+, Safari 17+ (2023年底起全面支持)

---

## v5.2.1

### Features
- 授权码系统（SHA-256哈希验证）
- 已知角色/NPC/物品的编辑和删除功能
- crypto.subtle 降级兼容修复

## v5.0.0

### Major Release
- PageIndex + Embedding + MemGPT Agent 架构
- 独立存档系统（跨聊天记忆持久化）
- Embedding 向量语义检索
- 语义分类标签系统
- 增强记忆代理（6工具集 + 两轮工具调用）
- 统一检索流（Embedding预筛选 → Agent检索 → 关键词降级）
- 副API支持
- 渐进式压缩（时间线/故事页/归档）
