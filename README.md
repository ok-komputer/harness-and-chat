# harness-and-chat

让弱模型本地 coding harness（opencode、claude code、codex 等）通过"用户复制粘贴"中转，借用强模型网页 chat（ChatGPT、Claude、Gemini 等任意网页 chat）的推理能力。

## 它解决什么问题

本地跑的 harness 用的是弱模型，遇到难任务搞不定。网页版 chat 用的是强模型，但没法直接操作你的代码。这个项目让弱模型 harness 生成一个"首问包"，你复制到网页 chat，chat 给出答案后你再复制回 harness，harness 自动解析。**你只做复制粘贴，不参与思考。**

## 文件说明

| 文件 | 用途 |
|------|------|
| `bridge/SKILL.md` | opencode skill 主体，自包含。opencode 用户只需这一个文件 |
| `PROTOCOL.md` | 纯协议文档，给非 opencode harness 用户参考 |
| `README.md` | 本文件 |

## 安装（opencode 用户）

**方式一：一键安装（推荐）**

```bash
npx skills add ok-komputer/harness-and-chat
```

这会自动把 `bridge` skill 安装到 opencode 的 skills 目录。

**方式二：手动复制**

把 `bridge/SKILL.md` 复制到：
- 全局：`~/.config/opencode/skills/bridge/SKILL.md`（Windows: `C:\Users\<用户名>\.config\opencode\skills\bridge\SKILL.md`）
- 项目级：`<项目目录>\.opencode\skills\bridge\SKILL.md`

安装后在 opencode 里打 `/bridge <任务描述>` 即可触发。

## 快速上手示例

**场景**：你遇到一个报错搞不定，想让强模型 chat 帮忙看看。

**第 1 步**：在 opencode 里打
```
/bridge 帮我看看这个报错 KeyError: 'user_id' in auth.py，代码大概是这样：
def get_user(req):
    return users[req['user_id']]
```

**第 2 步**：skill 会输出一个首问包，类似这样：
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
你是一个被本地 harness 借用的外部顾问...
（完整内容见 SKILL.md）

===BRIDGE_END===
════复制终点════

chat 回答后，把 ===BRIDGE_ANSWER=== 到 ===BRIDGE_ANSWER_END===
之间的内容复制回来，连同下面这行触发短语一起贴给我：

/bridge-back
<粘贴 chat 的完整回答>

harness 看到 /bridge-back 就会自动解析终答。
```

**第 3 步**：复制 `════复制起点════` 到 `════复制终点════` 之间的内容，粘贴到网页 chat（ChatGPT/Claude/Gemini 都行）

**第 4 步**：如果 chat 反问补充信息，直接在网页里跟它聊（harness 不参与中间轮）

**第 5 步**：chat 给出最终答案后，复制它的完整回答（含 `===BRIDGE_ANSWER===` 标记）

**第 6 步**：回到 opencode，打 `/bridge-back` 换行，粘贴刚才复制的内容
```
/bridge-back
===BRIDGE_ANSWER===
可能是 req 里没有 'user_id' 这个 key。建议先检查 key 是否存在：
def get_user(req):
    if 'user_id' not in req:
        raise ValueError('missing user_id')
    return users[req['user_id']]
===BRIDGE_ANSWER_END===
```

**第 7 步**：opencode 看到 `/bridge-back` 自动解析 `===BRIDGE_ANSWER===` 段，提取答案并应用

## 支持的任务类型

skill 自动识别任务类型，无需手动指定：

| 类型 | 触发关键词（部分） | 用途 |
|------|------------------|------|
| debug | 报错、不工作、失败、crash | 调试问题，分析根因 |
| design | 设计、架构、方案、选型 | 架构/方案设计 |
| review | 审查、review、检查 | 代码审查 |
| code | 写、实现、add、refactor | 写代码 |
| general | （无命中时 fallback） | 通用求助 |

优先级：debug > design > review > code > general。说"帮我看看这个实现为什么不工作"会优先归 debug（因为命中"不工作"），符合直觉。

## 命令说明

| 命令 | 作用 |
|------|------|
| `/bridge <任务描述>` | 发起求助：生成首问包，你复制到网页 chat |
| `/bridge-back` + 粘贴回答 | 接收答案：harness 解析 chat 的回答 |

两个命令语义相反，分开避免弱模型混淆。`/bridge` 生成包，`/bridge-back` 解析包。

## 非 opencode harness 使用建议

本项目主要面向 opencode，但也支持其他 harness。**适配由用户自行完成，不保证可用。**

### claude code

把 `PROTOCOL.md` 的内容贴进项目的 `CLAUDE.md`，让 claude code 理解协议。然后在对话里手动模拟 `/bridge` 和 `/bridge-back` 的语义：
- 发起求助时，让 claude code 按 PROTOCOL.md 的首问包格式输出
- 接收答案时，跟它说"这是 chat 的回答："然后粘贴，让它按协议解析

### codex

把 `PROTOCOL.md` 作为 prompt 模板手动引用。codex 的 prompt 机制各版本差异较大，具体接入方式请参考 codex 文档。

### 其他 harness

任何能读 markdown 的 harness 都能用本协议。核心是让 harness 理解 sentinel 协议（`===BRIDGE_START===` 等标记）和两个触发语义（发起求助 / 接收答案）。

## 协议要点

- **sentinel 标记**：`===BRIDGE_START===` / `===TASK===` / `===CONTEXT===` / `===INSTRUCTIONS===` / `===BRIDGE_END===` / `===BRIDGE_ANSWER===` / `===BRIDGE_ANSWER_END===`
- **无状态**：harness 不存对话历史，多轮对话在 chat 网页里完成
- **宽松兜底**：chat 不按格式回答也没关系，harness 会尽力解析整段文本
- **超长处理**：首问包超 8000 字符时自动截断 CONTEXT 段（阈值可调）

完整协议定义见 `PROTOCOL.md`。
