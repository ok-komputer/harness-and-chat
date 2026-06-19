---
name: bridge
description: Bridge a weak local harness model to a strong web-based chat model via copy-paste handoff. Use when the user wants to delegate a hard task (debug/design/code/review) to a stronger external chat by generating a sentinel-formatted prompt package, then parse the answer back. Triggers: /bridge, /bridge-back, "帮我问问 chat", "借用强模型", "这个问题我搞不定".
license: MIT
---

# Bridge：让弱模型 harness 借用强模型 chat 的脑力

## 概述

这个 skill 解决一个常见痛点：本地跑的 coding harness（比如 opencode、claude code、codex）用的是比较弱的模型，遇到难任务搞不定；但网页版 chat（ChatGPT、Claude、Gemini 等任意网页 chat）用的是很强的模型。

Bridge skill 让弱模型 harness 通过"用户复制粘贴"的人工中转，借用强模型 chat 的推理能力。流程是这样的：

1. 用户在 harness 里打 `/bridge <任务描述>` 触发 skill
2. skill 自动识别任务类型（debug/design/code/review），生成一个"首问包"——里面是一段给 chat 看的提示词，教 chat 怎么回答、怎么把答案格式化好
3. 用户把首问包复制到网页 chat
4. chat 在网页里跟用户多轮对话（harness 不参与中间轮），最终给出格式化好的答案
5. 用户把答案复制回 harness，**开头带一行 `/bridge-back`** 触发解析
6. harness 按 sentinel 协议解析答案，然后执行或应用

**用户只做复制粘贴的体力活，不参与思考。harness 只参与"首问"和"终答"两头，中间的多轮对话完全在 chat 网页里完成。** 这让 skill 保持无状态，简单可靠。

---

## Sentinel 协议

sentinel = 用特殊标记把一段文本分成几个段落，让人能读懂、机器能解析。格式是 `===全大写标记===`，每行一个。

### 首问包（harness → chat，用户复制走）

```
===BRIDGE_START===
session: <8位随机ID>
type: debug | design | code | review
harness: opencode | claude-code | codex

===TASK===
<任务描述，按类型模板填充>

===CONTEXT===
<相关上下文：代码片段、报错堆栈、约束条件等>
<如果超长，在这里标注"已截断，如需完整内容请指定 file:line">

===INSTRUCTIONS===
你是一个被本地 harness 借用的外部顾问。harness 用的是弱模型，
请你用你的强能力帮它完成下面任务。

要求：
1. 仔细阅读 TASK 和 CONTEXT
2. 如果信息不够，直接问我（用户），我会补充
3. 得出最终答案后，请严格按下面格式输出，方便 harness 解析。
   注意：下面这行标记必须**各占一行**、**原样输出**（不要加反引号、
   不要加 markdown 引用块 >、不要放进代码块里）：

   ===BRIDGE_ANSWER===
   <你的最终答案，可以是代码、方案、分析等，原样输出，不要加引用前缀>
   ===BRIDGE_ANSWER_END===

4. 如果无法按格式回答，直接回答即可，harness 会尽力解析
5. 答案内容里如果需要展示 ===BRIDGE_ANSWER=== 这个字符串本身，
   请用代码块包裹那一小段，避免与外层标记混淆

===BRIDGE_END===
```

### 终答包（chat → harness，用户复制回）

```
===BRIDGE_ANSWER===
<chat 的最终答案>
===BRIDGE_ANSWER_END===
```

### 解析规则（harness 端）

harness 端解析用户粘贴回来的内容时，按以下规则：

- **严格模式**：找到**最后一个** `===BRIDGE_ANSWER===` 到其后**最近的** `===BRIDGE_ANSWER_END===` 之间的内容
- **去引用前缀**：仅当**整段**每一行都以 `> ` 开头时（说明被 chat 用 markdown 引用块整体包裹），才去掉每行开头的 `> ` 和多余缩进；如果只有部分行有 `>`，保持原样不动（避免误伤 diff/代码块里的 `>`）
- **宽松模式（fallback）**：如果找不到任何 `===BRIDGE_ANSWER===` sentinel，把用户粘贴的整段文本当答案
- **session ID 不强制校验**：无状态设计，session ID 只做人眼对齐用（见下方说明）

### Session ID 用途

- 8 位随机字母数字（如 `a1b2c3d4`），放在首问包开头
- 用途：**让用户在多任务并行时区分哪个首问包对应哪个终答**。比如用户同时开了 2 个求助任务（一个 debug、一个 design），session ID 帮用户对齐"这个终答回哪个首问"
- harness 端**不校验** session ID（无状态），它只是人眼标签

### Sentinel 冲突防护

- sentinel 用 `===全大写===` 格式，正常代码/文档里很少出现
- 如果用户代码里恰好有 `===BRIDGE_ANSWER===` 字符串，harness 端取**最后一个**匹配段（chat 的答案出现在回答末尾）
- 如果 chat 在答案内容里又用了 `===BRIDGE_ANSWER===`（比如它在解释协议本身），INSTRUCTIONS 段已要求 chat 把这种字符串用代码块包裹，让 harness 端能区分"标记"和"内容里的字符串"

---

## 终答回传入口（/bridge-back）

### 问题

harness 是弱模型，它怎么知道用户粘贴回来的是"终答包"，该按 sentinel 协议解析？如果不区分，harness 会把粘贴内容当普通文本处理，不会提取答案。

### 方案

用户粘贴终答时，**必须带一个触发短语** `/bridge-back` 让 harness 识别。skill 在首问包输出时就把这个短语告诉用户，让用户复制回的时候一起带上。

具体流程：

1. skill 生成首问包时，末尾提示用户："chat 回答后，请把答案**连同下面这行触发短语一起**复制回来贴给我："
   ```
   /bridge-back
   <chat 的回答，含 ===BRIDGE_ANSWER=== ... ===BRIDGE_ANSWER_END===>
   ```
2. 用户在 chat 网页复制 chat 的完整回答（含 sentinel 标记）
3. 用户回到 harness，输入 `/bridge-back` 然后换行粘贴 chat 的回答
4. harness 看到 `/bridge-back` 触发短语 → 进入"解析终答"模式 → 按 sentinel 协议提取 `===BRIDGE_ANSWER===` 段
5. 如果用户没带 `/bridge-back`，harness 把粘贴内容当普通文本处理（不解析 sentinel）

### 为什么不和 `/bridge` 复用

`/bridge <描述>` 是"发起求助"（生成首问包），`/bridge-back <回答>` 是"接收答案"（解析终答），语义相反。分开命令让弱模型 harness 不会混淆——看到 `/bridge` 就生成包，看到 `/bridge-back` 就解析包。

### 非 opencode harness 适配

`/bridge-back` 只是约定短语。claude code 用户可以自己定义等价命令或直接说"这是 chat 的回答："后粘贴。核心原则：**触发短语可按 harness 自定义，但必须有一个明确入口**区分"发起求助"和"接收答案"两种语义。

---

## 任务模板（4 类 + general fallback）

skill 读取用户输入的任务描述，按**优先级顺序**匹配关键词。优先级固定为 **debug > design > review > code > general**。

### debug 模板（调试问题）— 优先级 1

- **触发关键词**: `报错|错误|exception|stack trace|traceback|不工作|失败|bug|crash|panic|挂了|崩了`
- **TASK 段填充**: `请帮我调试以下问题：<用户描述的现象>`
- **CONTEXT 段填充**: 报错堆栈 + 相关代码片段 + 已尝试过的修复（由用户在触发时提供，skill 不主动读文件）
- **INSTRUCTIONS 补充**: `请分析根因，给出修复方案（代码 diff 或步骤）`

### design 模板（架构/方案设计）— 优先级 2

- **触发关键词**: `设计|架构|方案|怎么实现|如何设计|design|architecture|approach|选型`
- **TASK 段填充**: `请帮我设计以下方案：<用户描述的需求>`
- **CONTEXT 段填充**: 现有架构、约束条件、技术栈、性能要求（由用户提供）
- **INSTRUCTIONS 补充**: `请给出设计方案，包含权衡分析和推荐方案`

### review 模板（代码审查）— 优先级 3

- **触发关键词**: `审查|review|看看这段|检查|有没有问题|code review|评审`
- **TASK 段填充**: `请帮我审查以下代码：<用户指定的范围>`
- **CONTEXT 段填充**: 待审查的代码 + 审查重点（可选，由用户提供）
- **INSTRUCTIONS 补充**: `请指出问题（按严重程度分类）、给出改进建议`

### code 模板（写代码）— 优先级 4

- **触发关键词**: `写|实现|add|implement|create|refactor|编码|函数|类|方法`
- **TASK 段填充**: `请帮我实现以下功能：<用户描述的目标>`
- **CONTEXT 段填充**: 相关现有代码、接口约定、风格规范（由用户提供）
- **INSTRUCTIONS 补充**: `请给出完整代码，遵循现有风格，包含必要注释`

### general 通用模板（fallback）— 优先级 5

当任务描述匹配不到任何一类，或匹配到 ≥2 类且无法按优先级唯一确定时使用：
- **TASK 段**: 直接放用户原始描述
- **CONTEXT 段**: 放用户提供的上下文
- **INSTRUCTIONS 段**: 不额外补充，走默认

---

## 自动识别逻辑

skill 读取用户输入的任务描述，按**优先级链**依次尝试匹配：

1. **先试 debug**：命中 `报错|错误|exception|stack trace|traceback|不工作|失败|bug|crash|panic|挂了|崩了` 任一 → 归为 debug
2. **再试 design**：命中 `设计|架构|方案|怎么实现|如何设计|design|architecture|approach|选型` 任一 → 归为 design
3. **再试 review**：命中 `审查|review|看看这段|检查|有没有问题|code review|评审` 任一 → 归为 review
4. **再试 code**：命中 `写|实现|add|implement|create|refactor|编码|函数|类|方法` 任一 → 归为 code
5. **fallback**：以上都未命中，或匹配到 ≥2 类且无法按优先级唯一确定 → general 通用模板

**为什么是这个优先级**：用户说"帮我看看这个实现为什么不工作"会同时命中 code（"实现"）和 debug（"不工作"）。按优先级 debug 优先，归为 debug。这符合直觉——用户说"不工作"时最需要的是先定位问题根因，而不是直接改代码。

**示例判断**：

| 用户输入 | 命中 | 归类 |
|---------|------|------|
| "帮我看看这个报错 KeyError in auth.py" | debug（报错） | debug |
| "帮我设计一个用户认证方案" | design（设计、方案） | design |
| "审查一下这段代码" | review（审查） | review |
| "帮我写一个 JWT 中间件" | code（写） | code |
| "帮我看看这个实现为什么不工作" | debug（不工作）+ code（实现） | debug（优先） |
| "今天天气怎么样" | 无命中 | general |

---

## 用户操作手册

### 触发方式

在 harness 里打：
```
/bridge <任务描述>
```

任务描述可以包含：问题现象、报错信息、需求、待审查的代码片段等。skill 会自动识别任务类型并生成首问包。

### 完整流程示例（debug 类）

用户输入：
```
/bridge 帮我看看这个报错 KeyError: 'user_id' in auth.py，代码大概是这样：
def get_user(req):
    return users[req['user_id']]
```

skill 输出：
```
已识别任务类型：debug
请复制以下内容到网页 chat：

════复制起点════
===BRIDGE_START===
session: a1b2c3d4
type: debug
harness: opencode

===TASK===
请帮我调试以下问题：帮我看看这个报错 KeyError: 'user_id' in auth.py

===CONTEXT===
报错：KeyError: 'user_id'
文件：auth.py
相关代码：
def get_user(req):
    return users[req['user_id']]

===INSTRUCTIONS===
你是一个被本地 harness 借用的外部顾问。harness 用的是弱模型，
请你用你的强能力帮它完成下面任务。

要求：
1. 仔细阅读 TASK 和 CONTEXT
2. 如果信息不够，直接问我（用户），我会补充
3. 得出最终答案后，请严格按下面格式输出，方便 harness 解析。
   注意：下面这行标记必须各占一行、原样输出（不要加反引号、
   不要加 markdown 引用块 >、不要放进代码块里）：

   ===BRIDGE_ANSWER===
   <你的最终答案，可以是代码、方案、分析等，原样输出，不要加引用前缀>
   ===BRIDGE_ANSWER_END===

4. 如果无法按格式回答，直接回答即可，harness 会尽力解析
5. 答案内容里如果需要展示 ===BRIDGE_ANSWER=== 这个字符串本身，
   请用代码块包裹那一小段，避免与外层标记混淆

请分析根因，给出修复方案（代码 diff 或步骤）

===BRIDGE_END===
════复制终点════

chat 回答后，把 ===BRIDGE_ANSWER=== 到 ===BRIDGE_ANSWER_END===
之间的内容复制回来，连同下面这行触发短语一起贴给我：

/bridge-back
<粘贴 chat 的完整回答>

harness 看到 /bridge-back 就会自动解析终答。
```

用户操作：
1. 复制 `════复制起点════` 到 `════复制终点════` 之间的所有内容
2. 粘贴到网页 chat（ChatGPT/Claude/Gemini 等）
3. 如果 chat 反问补充信息，直接在网页里跟它聊（harness 不参与）
4. chat 给出最终答案后，复制 `===BRIDGE_ANSWER===` 到 `===BRIDGE_ANSWER_END===` 之间的内容（或整段回答）
5. 回到 harness，打 `/bridge-back` 换行，粘贴刚才复制的内容
6. harness 解析答案，执行或应用

### 其他模板示例（简略）

**design 类**：
```
/bridge 帮我设计一个高并发的限流方案，QPS 预计 10000
```
TASK: `请帮我设计以下方案：帮我设计一个高并发的限流方案，QPS 预计 10000`
INSTRUCTIONS 补充: `请给出设计方案，包含权衡分析和推荐方案`

**review 类**：
```
/bridge 审查一下这段代码有没有竞态条件：<粘贴代码>
```
TASK: `请帮我审查以下代码：审查一下这段代码有没有竞态条件`
INSTRUCTIONS 补充: `请指出问题（按严重程度分类）、给出改进建议`

**code 类**：
```
/bridge 帮我实现一个 LRU 缓存，要求 O(1) 读写
```
TASK: `请帮我实现以下功能：帮我实现一个 LRU 缓存，要求 O(1) 读写`
INSTRUCTIONS 补充: `请给出完整代码，遵循现有风格，包含必要注释`

---

## Guardrails 与边缘情况

### 边缘情况处理

| 情况 | 处理 |
|------|------|
| 用户输入空任务描述 | skill 提示"请描述你的任务"并退出 |
| 自动识别置信度低（≥2 类无法按优先级唯一确定） | fallback 到 general 通用模板 |
| chat 回答没有 sentinel 包裹 | harness 端宽松解析，整段当答案 |
| 首问包超长（>8000 字符，保守阈值） | CONTEXT 段截断 + 标注"已截断，如需完整内容请在 chat 里指定 file:line 让我补"。阈值是保守值，用户可根据目标 chat 的输入上限自行调 |
| 用户代码里恰好有 sentinel 字符串 | harness 取最后一个匹配段；sentinel 用全大写降低冲突概率 |
| chat 网页给回答加了 `>` 引用块 | harness 解析时仅当**整段**每行都以 `> ` 开头才去引用前缀；部分行有 `>` 保持原样（避免误伤代码/diff） |
| chat 不按格式回答 | INSTRUCTIONS 段已写"无法按格式回答就直接回答"，harness 端宽松解析兜底 |

### Guardrails（必须遵守）

- skill **不发送任何网络请求**——纯文本生成，不调用 chat API
- skill **不读写用户代码文件**——所有上下文（报错堆栈、代码片段、约束条件）由用户在触发 `/bridge` 时**手动提供**，skill 只负责打包。skill 永远不主动读文件系统
- skill 输出的包必须有**明确的"复制起点"和"复制终点"提示**（`════复制起点════` / `════复制终点════` 标记）
- 自动识别置信度低时**必须 fallback**，不能硬选（fallback 规则见自动识别逻辑章节）
- sentinel 协议**不做版本协商**——v1 固定，未来有 v2 再说
- 终答解析入口固定为 `/bridge-back` 触发短语，harness 见此短语才进入解析模式

---

## 安装

### opencode 用户

把本文件复制到以下任一路径：

- 全局：`~/.config/opencode/skills/bridge/SKILL.md`（Windows: `C:\Users\<用户名>\.config\opencode\skills\bridge\SKILL.md`）
- 项目级：`<项目目录>\.opencode\skills\bridge\SKILL.md`

### 其他 harness 用户

本项目还提供 `PROTOCOL.md`——纯协议文档，不含 opencode 特有的 frontmatter。claude code / codex 用户可把协议内容贴进各自的系统提示（如 CLAUDE.md）、自定义命令、或 prompt 模板。**适配由用户自行完成**，PROTOCOL.md 只提供协议定义参考。
