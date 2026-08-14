# DeepSeek Harness 可追溯性学习指南

> 面向初学者：理解 Harness 如何用「只追加会话日志」实现端到端可追溯性，
> 并把这套思路借鉴到你自己用 Claude Code 开发的项目中，实现跨会话可追溯。

---

## 0. 一句话总览

Harness 的可追溯性建立在一个简单但严格的不等式上：

> **磁盘上只追加（append-only）的事件日志 = 唯一真相源**
> **模型"看到"的消息列表 = 由日志投影（projection）出来的派生视图**

也就是说，模型每一次请求时看到的 system prompt、思维链、工具调用与结果、子 Agent 调度、每一次上下文注入，都**不是被直接存储的对象**，而是从一条只增不改的事件流里"折叠"出来的。任何到达模型的东西都能从日志重建；上下文压缩不会删除原始历史，只会追加一个"替换事件"来改变此后模型看到的表象。

这个不变量在仓库根 [AGENTS.md](file:///workspace/AGENTS.md) 里被明确写成一条规矩：

> **Model-visible ⟺ logged**: anything that reaches a model request must be reconstructable from the session log; a new model-visible input requires a session event.

---

## 1. 先建立心智模型：三层结构

理解 Harness 可追溯性，先把这三层分清楚：

```
┌──────────────────────────────────────────────────────────────┐
│  第 1 层：Append-only 事件日志  （source of truth，磁盘文件）   │
│  每一行是一个不可变事件 SessionEvent，seq 单调递增，永不删除    │
│  文件：~/.dsh/<project>/<session-id>/session.jsonl            │
└──────────────────────────────────────────────────────────────┘
                          │ foldSurface(events) 投影
                          ▼
┌──────────────────────────────────────────────────────────────┐
│  第 2 层：Surface（模型可见的有序消息视图）                      │
│  只包含三类"产生消息"的事件：user/message、assistant/message、  │
│  tool/result。压缩用 replace 操作在 surface 上"遮蔽"旧范围     │
└──────────────────────────────────────────────────────────────┘
                          │ deriveMessages()
                          ▼
┌──────────────────────────────────────────────────────────────┐
│  第 3 层：Message[]（发给 LLM 的请求体里的对话历史）             │
└──────────────────────────────────────────────────────────────┘
```

关键点：**第 2、3 层都是"算出来的"，不是"存的"**。只有第 1 层落在磁盘上，且只追加。所以无论中途压缩了多少次、模型后来看到的是什么"表象"，只要日志还在，原始历史就一字不差地还在。

---

## 2. 日志记录到什么文件？怎么找到它？

### 2.1 路径解析规则

Harness 把所有用户数据收在一个 "home" 目录下，解析优先级见 [home-paths/src/index.ts](file:///workspace/packages/util/home-paths/src/index.ts) 第 87-91 行的 `resolveDshHome()`：

1. 显式配置的路径（最高优先级）
2. 环境变量 `$DSH_HOME`（空白视为未设）
3. 默认值 `~/.dsh`

最终的会话日志路径由 [session-persistence-jsonl/src/format.ts](file:///workspace/packages/session/session-persistence-jsonl/src/format.ts) 第 201-208 行的 `logPath()` 拼出：

```
<home>/<projectDir>/<sessionDir>/session<suffix>
```

各段含义：

| 段 | 生成函数 | 例子 |
|---|---|---|
| `<home>` | `resolveDshHome()` | `~/.dsh` |
| `<projectDir>` | `projectDir(root, cwd)` 第 176-179 行 | `--Users-foo-myproject--` |
| `<sessionDir>` | `sessionDir(...)` 第 189-191 行 = `encodeSegment(sessionId)` | 一段安全编码后的 id |
| `<suffix>` | `logSuffix(compression)` 第 24-26 行 | `.jsonl` 或 `.jsonl.zstd` |

所以一条典型路径长这样：

```
~/.dsh/--Users-foo-myproject--/a1b2~003C.../session.jsonl
```

几个设计细节值得初学者注意（都在 `format.ts` 里）：

- `projectKey(cwd)`（第 147-167 行）把项目路径转成人类可读的目录键：分隔符 `/ \ :` 变成 `-`，不安全字符用 `~XXXX` 转义，前后加 `--`。所以你能用 `ls ~/.dsh` 一眼看出是哪个项目。
- `encodeSegment(id)`（第 121-136 行）把 SessionId 编码成**单一安全路径段**，杜绝 `../`、绝对路径、NUL、分隔符注入。
- `cwd === undefined` 时落到 `_no-cwd` 目录（第 177 行）。
- 文件可以是明文 `.jsonl`，也可以是 zstd 压缩的 `.jsonl.zstd`。

### 2.2 文件格式：NDJSON（每行一个 JSON 对象）

日志文件就是 **Newline-Delimited JSON**：第一行是 header，后面每一行是一个事件。

```jsonl
{"type":"session","version":0,"id":"<sessionId>","createdAt":0,"cwd":"<cwd>","delegationDepth":0}
{"type":"turn/start","seq":1,"time":1786123401614,"data":{"turn":1}}
{"type":"user/message","seq":4,"time":...,"data":{...},"surfaceOp":"append"}
{"type":"assistant/chunk","seq":8,"time":0,"data":{"turn":1,"step":1,"chunk":{"type":"block-start",...}}}
...
```

- 第 1 行是 `HeaderLine`（[format.ts](file:///workspace/packages/session/session-persistence-jsonl/src/format.ts) 第 33-44 行），`type: "session"`，携带 `version`、`id`、`createdAt`、`cwd` 等**会话级元数据**。元数据刻意放在事件流之外（见 [types.ts](file:///workspace/packages/core/session/src/types.ts) 第 59 行注释）。
- 从第 2 行起每行是一个 `SessionEvent`，按 `seq` 单调递增，`seq` 始终等于该行在事件流中的下标。

---

## 3. 实际使用：跑一个任务，然后看日志

### 3.1 跑一个任务

Harness 的 CLI 入口（见 [AGENTS.md](file:///workspace/AGENTS.md) 命令清单）：

```sh
# 从源码跑一个任务（需要 DEEPSEEK_API_KEY）
pnpm dsh --profile headless "你的任务描述"
```

任务跑完后，会话日志就落在了上节描述的路径下。

### 3.2 找到并查看日志

```sh
# 1. 看 harness home（默认 ~/.dsh，可用 $DSH_HOME 覆盖）
echo "${DSH_HOME:-$HOME/.dsh}"

# 2. 列出所有项目目录
ls ~/.dsh

# 3. 进入对应项目目录，列出会话
ls ~/.dsh/--Users-foo-myproject--

# 4. 进入某个会话目录，看日志文件
ls ~/.dsh/--Users-foo-myproject--/<session-id>/
# → session.jsonl  （或 session.jsonl.zstd）

# 5. 直接读（明文情况）。每行是一个 JSON 对象：
#    用 jq 美化第一行（header）
head -1 ~/.dsh/.../session.jsonl | jq .
#    看每个事件的 type 和 seq
cat ~/.dsh/.../session.jsonl | jq -c '{seq,type}'
```

> 压缩格式 `.jsonl.zstd` 需要先 `zstd -d` 解压再读。

### 3.3 真实示例：一次完整带压缩的会话长什么样

下面这段是从仓库测试快照里摘出来的真实日志（[examples/headless-agent/tests/snapshots/compaction-recovery/session.jsonl](file:///workspace/examples/headless-agent/tests/snapshots/compaction-recovery/session.jsonl)），我做了删减和注释，正好覆盖"用户提问 → 工具调用 → 上下文超限 → 压缩 → 恢复继续"全流程：

```jsonl
# 第 1 行：header，version:0 是 SESSION_FORMAT_VERSION
{"type":"session","version":0,"id":"...","createdAt":0,"cwd":"...","delegationDepth":0}

# 用户消息进入 surface（surfaceOp:"append"）
{"type":"turn/start","seq":1,"time":...,"data":{"turn":1}}
{"type":"step/start","seq":3,"time":...,"data":{"turn":1,"step":1}}
{"type":"user/message","seq":4,"time":...,"data":{"content":[{"type":"text","text":"..."}],...},"surfaceOp":"append"}

# 请求头快照：完整的 system prompt + tools 定义都被记录（仅日志，不进 surface）
{"type":"request/header","seq":6,"time":...,"data":{"header":{"config":{...},"system":"You are an AI agent...","tools":[...]},"reason":"initial"}}
{"type":"request/context","seq":7,"time":...,"data":{"provider":"deepseek-official","model":"deepseek-v4-flash","contextWindow":128000}}

# 助手流式输出的原始 chunk（token 级保真）
{"type":"assistant/chunk","seq":8,"time":0,"data":{"turn":1,"step":1,"chunk":{"type":"block-start","index":0,"blockType":"tool-call"}}}
{"type":"assistant/chunk","seq":9,"time":0,"data":{"turn":1,"step":1,"chunk":{"type":"tool-call-delta","index":0,"id":"call_compaction_marker","name":"bash","argumentsDelta":"{\"command\":\"printf 'alpha\\n'\"}"}}}
{"type":"assistant/chunk","seq":11,"time":0,"data":{"turn":1,"step":1,"chunk":{"type":"usage","usage":{"inputTokens":24,"outputTokens":6}}}}

# 组装好的助手消息（进 surface），usage 跟着消息一起走，没有独立 usage 记录
{"type":"assistant/message","seq":13,"time":...,"data":{"turn":1,"step":1,"message":{"role":"assistant","content":[{"type":"tool-call","id":"call_compaction_marker","name":"bash","arguments":"{...}"}],...},"usage":{"inputTokens":24,"outputTokens":6}},"sourceEventSeqs":[8,9,10,11,12],"surfaceOp":"append"}

# 工具调用与结果配对（callId 配对），结果也进 surface
{"type":"tool/call","seq":14,"time":...,"data":{"turn":1,"step":1,"callId":"call_compaction_marker","name":"bash","arguments":"{...}"}}
{"type":"tool/result","seq":15,"time":...,"data":{"turn":1,"step":1,"message":{"content":[{"type":"tool-result","toolCallId":"call_compaction_marker","content":[{"type":"text","text":"alpha\n"}],"isError":false}],"role":"user","id":"..."},"sourceEventSeqs":[14],"surfaceOp":"append"}

# 第二个 step：上下文超限！
{"type":"step/start","seq":17,"time":...,"data":{"turn":1,"step":2}}
{"type":"assistant/chunk","seq":18,"time":...,"data":{"turn":1,"step":2,"chunk":{"type":"finish","reason":{"kind":"error","failure":{"message":"snapshot request exceeded the model context window","code":"CONTEXT_WINDOW_EXCEEDED"}}}}}

# ★ 压缩开始：加锁
{"type":"compaction/start","seq":19,"time":...,"data":{"compactionId":"338e88fa-...","turn":1}}

# ★ 压缩摘要：记录被遮蔽的范围 {start:4,end:4}、shadowedSeqs:[4]、token 估算、模型调用 provenance
{"type":"compaction/summary","seq":20,"time":...,"data":{"compactionId":"338e88fa-...","summary":[{"type":"text","text":"The request established a durable compaction premise."}],"rawOutput":[...],"llmStreamCall":true,"shadowedRange":{"start":4,"end":4},"shadowedSeqs":[4],"shadowedTokenCount":266,"provider":"deepseek-official","model":"deepseek-v4-flash","maxTokens":32,"usage":{"inputTokens":20,"outputTokens":4}}}

# ★ 真正执行 surface 替换的事件：一个 user/message，携带 surfaceOp:{op:"replace",start:4,end:4}
#   注意 sourceEventSeqs:[19,20,4] —— 它引用了 compaction/start、compaction/summary 和被遮蔽的原始 seq 4
{"type":"user/message","seq":21,"time":...,"data":{"content":[{"type":"text","text":"This is an automatically generated checkpoint..."},{"type":"text","text":"The request established a durable compaction premise."},{"type":"text","text":"</compacted-summary>"}],"source":{"kind":"plugin","plugin":"compact","compactionId":"338e88fa-..."},"role":"user","id":"..."},"sourceEventSeqs":[19,20,4],"surfaceOp":{"op":"replace","start":4,"end":4}}

# ★ 压缩结束：释放锁
{"type":"compaction/end","seq":22,"time":...,"data":{"compactionId":"338e88fa-...","turn":1}}

# 重试请求成功，模型输出 "COMPACTION RECOVERED"
{"type":"assistant/chunk","seq":23,"time":...,"data":{"turn":1,"step":2,"chunk":{"type":"block-start","index":0,"blockType":"text"}}}
...
{"type":"assistant/message","seq":28,"time":...,"data":{"turn":1,"step":2,"message":{"role":"assistant","content":[{"type":"text","text":"COMPACTION RECOVERED"}],...},"usage":{"inputTokens":20,"outputTokens":4}},"sourceEventSeqs":[23,24,25,26,27],"surfaceOp":"append"}
{"type":"step/end","seq":29,"time":...,"data":{"turn":1,"step":2}}
{"type":"turn/end","seq":30,"time":...,"data":{"turn":1,"reason":{"kind":"completed"}}}
```

仔细看这段日志，你能亲眼看到可追溯性的精髓：**原始的 seq=4 那条用户消息从未被删除**，它还好好地躺在日志里；压缩只是追加了 `compaction/start`、`compaction/summary`、一条带 `surfaceOp:replace` 的 `user/message`、`compaction/end` 这四个事件。模型后来看到的是替换后的 checkpoint，但任何人重放日志都能看到原始的、完整的历史。

---

## 4. 事件类型：到底哪些东西会被记录？

事件类型表是"声明合并（declaration merging）"出来的：核心包在 [types.ts](file:///workspace/packages/core/session/src/types.ts) 第 236-333 行定义基础事件，各领域包再通过 `declare module '@deepseek-ai/dsh-session/types' { interface SessionEventMap { ... } }` 往里 merge 自己的事件。完整的权威目录见自动生成的 [docs/persistence-catalog.md](file:///workspace/docs/persistence-catalog.md)（由 `scripts/gen-persistence-catalog.ts` 生成，`pnpm run doc-sync` 校验）。

按用途分类：

### 4.1 边界标记（仅日志，不进 surface）

- `turn/start`、`turn/end` —— 一轮对话的边界，`turn/end` 带 `reason`（completed / interrupted / …）
- `step/start`、`step/end` —— 一次模型调用 + 其工具执行的边界
- `session/end-seed` —— 构造器 seed 结束标记（resume / fork / replay 的历史到此为止）

### 4.2 Surface 消息（模型可见，进 surface）

只有这三种事件类型能进 surface（[types.ts](file:///workspace/packages/core/session/src/types.ts) 第 343-347 行 `SurfaceEventType`）：

- `user/message` —— 用户角色消息。可能是：直接人类提示、合成的 `agent.inject()` 上下文（文件变更通知、子目录 AGENTS.md、skill 内容、cron 通知）、目标续作回合。三者用 `source` 字段区分，但**投影时 content 原样直通**，不加额外 framing。
- `assistant/message` —— 组装好的助手消息，`usage`（token 计费）跟着消息一起走，**没有独立的 usage 记录**。
- `tool/result` —— 完成的工具结果，`callId` 与 `tool/call` 配对。

### 4.3 原始流块（仅日志，token 级保真）

- `assistant/chunk` —— 模型流式输出的每一个原始块（text-delta、tool-call-delta、usage、finish 等）。有了它，你能逐 token 重放模型当时是怎么吐字的。

### 4.4 工具调用配对（仅日志）

- `tool/call` —— 模型请求的工具调用，`arguments` 是模型产出的**原始 JSON 字符串（未解析）**，`callId` 配对 `tool/result`。

### 4.5 请求头与上下文（仅日志）

- `request/header` —— 下一次请求的完整头快照（system prompt + tools 定义 + config），`reason: 'initial' | 'resume' | 'change'`。重建请求时取**最新**一条。**这一条直接回答了你的疑问：系统提示词是被完整记录的。**
- `request/context` —— 路由元数据（provider / model / contextWindow），仅路由或容量变化时记录。

### 4.6 子 Agent 调度（声明合并）

- `subagent/descriptor`（[subagent/src/descriptor.ts](file:///workspace/packages/subagent/subagent/src/descriptor.ts) 第 28-39 行）—— 持久化的子 Agent 身份与生命周期模式（`one-shot` / `continuable`），log-only，跨压缩保留。这就是"子 Agent 调度被完整记录"的落点。

### 4.7 压缩事件（声明合并，见 [compaction/src/types.ts](file:///workspace/packages/compaction/compaction/src/types.ts) 第 16-90 行）

- `compaction/start` —— 加锁
- `compaction/summary` —— 摘要内容 + 输入 + 模型调用事实（log-only，无 surfaceOp）
- `compaction/end` —— 释放锁
- `compaction/prune` —— 无模型调用的 prune 替换的"影子价格"

### 4.8 其它声明合并的事件

`todo/write`、`session/title*`、`approval/*`、`hook/*`、`llm/retry*`、`plan/mode`、`permission/preset`、`command/*`、`goal/change`、`agent-preset/selected`、`agent/inbox/spliced`、`tool-workflow/*` 等等——完整列表见 [known-event-types.ts](file:///workspace/packages/core/session/src/known-event-types.ts)。

---

## 5. 核心精髓：压缩如何"不删历史，只改表象"

这是整个可追溯性设计里最巧妙的地方，初学者一定要吃透。

### 5.1 SurfaceOp：surface 上的两种操作

[types.ts](file:///workspace/packages/core/session/src/types.ts) 第 372-374 行：

```typescript
export type SurfaceOp =
  | 'append'                                              // 加到尾部（正常路径）
  | { op: 'replace'; start: number; end: number }         // 替换 [start, end] 闭区间
```

只有上面说的三类 `SurfaceEventType` 事件能携带 `surfaceOp`。普通消息用 `'append'`；压缩用 `{ op: 'replace', start, end }`。

### 5.2 压缩事务的四步

一次完整压缩（见 [compaction-basic/src/region.ts](file:///workspace/packages/compaction/compaction-basic/src/region.ts) 的 `compactSurfaceRegion`）按顺序追加四个事件：

1. `compaction/start` —— 获取 durable 锁
2. `compaction/summary` —— 写入摘要内容 + 输入范围 + 模型调用 provenance（**log-only，无 surfaceOp**）
3. `user/message`（checkpoint 消息）—— 携带 `surfaceOp: { op: 'replace', start, end }`，**这才是真正执行 surface 替换的事件**
4. `compaction/end` —— 释放锁

注意第 2 步的 `compaction/summary` **本身不改变 surface**，它只是把"我这次压缩摘要了什么、用了哪个模型、花了多少 token"记进日志。真正动 surface 的是紧跟着的第 3 步那条 `user/message`。

### 5.3 为什么这么设计？

- **原始事件永不删除**：被 `replace` 的旧 `user/message` / `assistant/message` / `tool/result` 仍然完整留在磁盘日志里，只是不再出现在 `surface.nodes` 视图中。人类想看真实对话历史？用 `isAppendSurfaceEvent`（[surface.ts](file:///workspace/packages/core/session/src/surface.ts) 第 51-55 行）筛出"原始 append 来源"事件即可，第 44-48 行注释说得很清楚：model-visible surface 是人类 transcript 的**错误来源**（因为替换会抹掉用户已看过的对话），append-origin 事件才是正确来源。
- **锁覆盖整个事务**：`start` 先于 summary 与替换落地，**最后**才 `end`。这个反向设计让崩溃中段变成可检测的"孤儿锁"（有 `start` 无 `end`），而不是一个虚假声称完成的 `compaction/end`。详见 [docs/subsystems/compaction.md](file:///workspace/docs/subsystems/compaction.md) 第 19-21 行。
- **provenance 可追溯**：替换事件的 `sourceEventSeqs` 必须包含每个被遮蔽的 surface 节点（[surface.ts](file:///workspace/packages/core/session/src/surface.ts) 第 211-243 行 `assertProvenance` 校验）。所以你能从 checkpoint 反查到它替掉了哪些原始事件。

### 5.4 一图理解"压缩前 vs 压缩后"

```
压缩前磁盘日志（seq 数组）：
  0  1  2  3  [4:user/msg]  5  6  ...  13[asst/msg]  14[tool/call]  15[tool/result]  16
                ↑                                                                               ↑
              将被遮蔽                                                                      最新结果（保留）

surface 视图（压缩前）：
  [4, 13, 15, ...]   ← 模型看到完整历史

====== 压缩发生，追加 seq 19/20/21/22 ======

压缩后磁盘日志（只增不改）：
  0  1  2  3  [4]  5  6  ...  13  14  15  16  17  18  19[start]  20[summary]  21[replace-msg]  22[end]
  ↑原始 seq 4 仍在！                                              ↑这条 replace 遮蔽了 seq 4

surface 视图（压缩后）：
  [21, 13, 15, ...]   ← 模型看到的是 checkpoint（seq 21），不是原始 seq 4
                       但任何人重放日志都能看到 seq 4 的原始内容
```

---

## 6. 投影与重放：如何从日志重建模型视图

### 6.1 入口函数

[surface.ts](file:///workspace/packages/core/session/src/surface.ts) 第 387-395 行的 `foldSurface(events)` 是完整重放整个日志的入口：遍历所有事件，对每个调用 `applySurfaceEvent`，返回 `{ nodes: number[]; replacements: SurfaceFoldReplacement[] }`。

### 6.2 applySurfacePlan：替换的真正机制

[surface.ts](file:///workspace/packages/core/session/src/surface.ts) 第 362-379 行的 `applySurfacePlan` 是 append-only 日志与模型视图分离的真正机制：

```typescript
// append：把新 seq 加到尾部
state.nodes.push(plan.seq)

// replace：用 splice 把 [startIdx, endIdx] 闭区间替换为新 seq
state.nodes.splice(plan.startIdx, plan.endIdx - plan.startIdx + 1, plan.seq)
state.replaceGeneration += 1
```

`splice` 把旧 surface 节点从**视图**里移除，用新 seq 替代；但底层 `log` 数组不变——`log` 始终保留所有原始事件。

### 6.3 deriveMessages：把 surface 投影成 LLM 消息

[session/src/index.ts](file:///workspace/packages/core/session/src/index.ts) 第 726-741 行的 `Session.deriveMessages()` 遍历 `surface.nodes`，对每个 seq 调用 `deriveEventMessage`（[surface.ts](file:///workspace/packages/core/session/src/surface.ts) 第 83-114 行）投影出 LLM 消息：

```typescript
switch (event.type) {
  case 'user/message':     return event.data              // 原样直通
  case 'assistant/message': return /* 空内容跳过 */ event.data.message
  case 'tool/result':      return event.data.message
  default:                 return null                    // 非 surface 事件不投影
}
```

缓存按 `replaceGeneration` 失效：一旦发生 replace，整个 surface 重算（[index.ts](file:///workspace/packages/core/session/src/index.ts) 第 730-734 行）。这正是"压缩后模型看到压缩视图，但磁盘保留原始事件"的运行时实现。

### 6.4 增量验证：validateNext

`Session.append`（[index.ts](file:///workspace/packages/core/session/src/index.ts) 第 604-655 行）在事件进 `log` 之前调用 `SurfaceManager.validateNext`（[surface.ts](file:///workspace/packages/core/session/src/surface.ts) 第 421-429 行）做四重验证：seq 连续性、`surfaceOp` 形状与资格、替换范围合法性、provenance 完整性。**坏事件在进日志前就被拒绝**，保证日志永远是自洽的。

---

## 7. 版本管理与崩溃恢复

### 7.1 SESSION_FORMAT_VERSION

[types.ts](file:///workspace/packages/core/session/src/types.ts) 第 56 行：

```typescript
export const SESSION_FORMAT_VERSION = 0
```

这是一个**单一单调整数**（无 major/minor）。bump 的判定依据是 writer 发出的内容是否发生**结构性变更**（header 形状、envelope 结构、核心事件语义、surface 机制）；**添加普通事件类型不 bump**——词汇增长由 per-event 的 `ignorable` guard 覆盖。pre-release 阶段固定为 0，无兼容承诺，不兼容日志直接拒绝，无迁移路径。

### 7.2 ignorable: true 的语义

[types.ts](file:///workspace/packages/core/session/src/types.ts) 第 412-422 行：缺省为 **required**——reader 遇到未知 `type` 必须拒绝重建（而非静默跳过），因为未知 required 事件可能改变后续日志的解读方式。writer 仅对"丢失不影响重建的纯信息性记录"设 `ignorable: true`。默认 required 是**故意过度拒绝**：宁可让用户看到"升级 harness"，也不要静默恢复一个被掏空的 session。

### 7.3 崩溃恢复

- **torn-tail 截断**：[format.ts](file:////workspace/packages/session/session-persistence-jsonl/src/format.ts) 第 272-378 行的 `SessionLogScanner` 按换行符切分；崩溃留下的没有换行符的尾部被视为 torn tail 丢弃。
- **孤儿锁检测**：`compaction/start` 无匹配 `compaction/end` 即孤儿锁，可被识别。
- **孤儿 turn 关闭**：持久化后端在 reload 时关闭崩溃孤儿 turn，产生 `turn/end` 的 `reason: 'interrupted'`。
- **seq 连续性**：`consumeEventLine`（[format.ts](file:///workspace/packages/session/session-persistence-jsonl/src/format.ts) 第 347-377 行）校验 `event.seq !== this.events.length` 触发 seq gap 错误。

崩溃留下的是**可识别的痕迹**，而不是错误状态。

---

## 8. 跨会话能力：SessionHeader 与查询

### 8.1 SessionHeader 持久化元数据

[types.ts](file:///workspace/packages/core/session/src/types.ts) 第 61-99 行，每个会话的 header 携带：`version`、`id`、`createdAt`、`cwd`、`parentSession`（fork 血统）、`seedLength`、`origin`（如 `'subagent'`）、`delegationDepth`（递归预算，持久化以在 resume 后保留）、`agentPreset`（决定 session 工具与 prompt，持久化避免 resume 用不同组合重放历史）。

### 8.2 查询历史会话

[session-query/src/index.ts](file:///workspace/packages/session-query/session-query/src/index.ts) 提供 session 查询/枚举入口。关键优化：可以只读 header 行（`parseHeaderMeta`），无需解析整个 log，所以 session picker 的规模随**会话数**增长，而非**总会话大小**。

---

## 9. 借鉴到你的 Claude Code 项目：跨会话可追溯设计建议

你想在自己的 Claude Code 项目里实现跨会话可追溯性。下面是把 Harness 这套思路"移植"过去的最小可行设计，按重要性排序。

### 9.1 必须照搬的三个核心思想

1. **只追加的事件日志是唯一真相源**。无论你用什么存储（JSONL 文件、SQLite、还是别的），都不要原地修改或删除历史。所有变更都是"追加一个新事件"。
2. **模型可见消息是派生视图，不是存储对象**。不要把"发给模型的消息列表"存下来；要存"事件流"，然后写一个 `deriveMessages()` 函数从事件流投影出消息列表。
3. **压缩用"替换事件"，不用删除**。压缩时追加一个 `{ op: 'replace', range: [start, end] }` 事件，在投影层把旧范围遮蔽掉。原始事件永远留在磁盘上。

### 9.2 最小事件 schema（直接参考 Harness）

每个事件至少有这几个 envelope 字段：

```typescript
type SessionEvent = {
  type: string          // 判别标签，如 'user/message'、'assistant/message'、'tool/result'、'compaction/summary'
  seq: number           // 会话内单调递增，等于在日志里的行号
  time: number          // Unix epoch 毫秒
  data: unknown         // 类型化载荷（按 type 窄化）
  ignorable?: true      // 缺省 required；true 表示 reader 可安全跳过未知 type
  // 仅"产生消息"的事件携带这两个：
  surfaceOp?: 'append' | { op: 'replace'; start: number; end: number }
  sourceEventSeqs?: number[]   // 本事件引用的更早事件 seq（provenance）
}
```

判别联合（discriminated union）让 `switch (event.type)` 能无 cast 窄化 `event.data`——这是类型安全的关键。

### 9.3 跨会话可追溯的关键设计点

Harness 的"跨会话"主要靠以下几点，你可以按需取舍：

- **每个会话一个独立日志文件**，路径按项目分组（`<home>/<projectKey>/<sessionId>/session.jsonl`）。这样跨会话查询就是"列目录 + 读 header"，不需要全量扫描。
- **SessionHeader 放在事件流之外**（日志第一行）。跨会话列表只读 header，不解析事件流，规模随会话数增长。
- **`parentSession` 字段记录 fork 血统**。子 Agent 会话、resume 会话都能溯源到父会话，形成可追溯的会话树。
- **`request/header` 事件记录每次请求的完整 system prompt + tools**。这是"系统提示词被完整记录"的落点——跨会话对比 system prompt 变化只需读这类事件。
- **`subagent/descriptor` 事件记录子 Agent 调度**。跨会话追溯"谁调用了谁、用了什么模型、什么生命周期模式"全靠它。
- **`session/title*` 事件让会话可被命名检索**。

### 9.4 给 Claude Code 项目的落地步骤

1. **选定存储格式**。最简单是 NDJSON（每行一个 JSON 事件），像 Harness 一样。文件路径用 `<你的 home>/<projectKey>/<sessionId>/session.jsonl`。
2. **实现 append-only writer**。唯一写入入口是 `append(type, data, opts)`，在入口做：seq 连续性检查、JSON 可序列化校验、（对 surface 事件）surfaceOp 合法性校验。**事件进日志后深冻结**，防止运行时被改。
3. **实现 projection reader**。`foldSurface(events)` 遍历事件，维护一个 `nodes: number[]` 视图：遇 `append` 就 `push`，遇 `replace` 就 `splice`。然后 `deriveMessages()` 把 `nodes` 投影成 LLM 消息列表。
4. **实现压缩**。检测到上下文超限时，追加 `compaction/start` → `compaction/summary` → 一条带 `surfaceOp:replace` 的 `user/message` → `compaction/end`。**不要动旧事件。**
5. **实现崩溃恢复**。读取时按行切分，丢弃无换行符的尾部（torn tail）；校验 seq 连续性；检测孤儿锁（`start` 无匹配 `end`）。
6. **实现版本守卫**。日志 header 带 `version`，reader 遇到不匹配的版本直接拒绝并提示"升级"，绝不静默跳过。
7. **实现跨会话查询**。列出所有项目目录 → 每个项目下列出会话目录 → 只读每个会话的 header 行渲染列表。点进某个会话再完整重放事件流。
8. **接入 Claude Code**。在 Claude Code 的每次请求前，用 `deriveMessages()` 从事件日志投影出消息列表作为对话历史；请求结束后把 system prompt、user message、assistant response、tool calls/results 全部 append 成事件。压缩由你的 harness 在上下文超限时自动触发。

### 9.5 容易踩的坑（Harness 用代码注释和测试钉死过的）

- **不要在投影层给注入内容加 framing**。Harness 早期给 `agent.inject()` 的内容包过 `<context>` 之类的外壳，后来专门移除了（见 [.agents/notes/implemented/simplification/2026-07-20-unwrap-injected-content-envelopes.md](file:///workspace/.agents/notes/implemented/simplification/2026-07-20-unwrap-injected-content-envelopes.md)）。framing 应该由生产者烤进 `content`，投影层只做 verbatim 直通。
- **替换事件的 `sourceEventSeqs` 必须完整**。必须包含每个被遮蔽的 surface 节点，否则 provenance 断链。Harness 在 `assertProvenance` 里强制校验。
- **不要把 usage 单独存**。把 token 计费数据塞进 `assistant/message` 事件里跟消息一起走，避免重建时错位。
- **未知事件默认拒绝**，不要默认跳过。否则一个被遗忘的 marker 会让你的 session 被静默掏空而无人察觉。
- **锁要覆盖整个事务**，且 `end` 必须最后落盘。这样崩溃留下的是"孤儿 start"（可检测），不是"虚假完成的 end"（难发现）。
- **同进程类型边界不要重复校验**。在 `append` 入口校验一次 JSON 可序列化即可，存储端无需再防御性校验——这是 Harness 的 [AGENTS.md](file:///workspace/AGENTS.md) "Trust TypeScript at typed same-process boundaries" 原则。

---

## 10. 学习路径与参考资料

### 10.1 推荐阅读顺序

1. **本文** —— 建立心智模型
2. [docs/subsystems/session.md](file:///workspace/docs/subsystems/session.md) —— surface 顺序与 `deriveMessages()` 投影的官方文档
3. [docs/subsystems/persistence.md](file:///workspace/docs/subsystems/persistence.md) —— 持久化机制与版本立场
4. [docs/subsystems/compaction.md](file:///workspace/docs/subsystems/compaction.md) —— 压缩能力 seam 与锁语义
5. [docs/persistence-catalog.md](file:///workspace/docs/persistence-catalog.md) —— 所有持久化事件类型的权威目录（自动生成）
6. [.agents/notes/implemented/architecture/2026-08-10-session-log-version-mechanism.md](file:///workspace/.agents/notes/implemented/architecture/2026-08-10-session-log-version-mechanism.md) —— 版本机制的升级链与 migrate-on-continue 设计

### 10.2 关键源码文件速查表

| 关注点 | 文件 |
|---|---|
| 事件类型 / envelope / SurfaceOp / SESSION_FORMAT_VERSION | [packages/core/session/src/types.ts](file:///workspace/packages/core/session/src/types.ts) |
| Surface 折叠 / 替换机制 / deriveEventMessage | [packages/core/session/src/surface.ts](file:///workspace/packages/core/session/src/surface.ts) |
| Session 类（append / deriveMessages / envelope 校验） | [packages/core/session/src/index.ts](file:///workspace/packages/core/session/src/index.ts) |
| 已知事件类型集合 | [packages/core/session/src/known-event-types.ts](file:///workspace/packages/core/session/src/known-event-types.ts) |
| JSONL 路径 / 格式 / header / 崩溃恢复扫描器 | [packages/session/session-persistence-jsonl/src/format.ts](file:///workspace/packages/session/session-persistence-jsonl/src/format.ts) |
| 持久化协调器（版本检查 / ignorable 处理） | [packages/session/session-persistence/src/coordinator.ts](file:///workspace/packages/session/session-persistence/src/coordinator.ts) |
| harness home 路径解析 | [packages/util/home-paths/src/index.ts](file:///workspace/packages/util/home-paths/src/index.ts) |
| Compaction Service Definition | [packages/compaction/compaction/src/index.ts](file:////workspace/packages/compaction/compaction/src/index.ts) |
| Compaction 事件类型声明（merge 进 SessionEventMap） | [packages/compaction/compaction/src/types.ts](file:///workspace/packages/compaction/compaction/src/types.ts) |
| Compaction Provider（BasicCompactionEngine） | [packages/compaction/compaction-basic/src/index.ts](file:///workspace/packages/compaction/compaction-basic/src/index.ts) |
| 压缩事务编排与事件发射 | [packages/compaction/compaction-basic/src/region.ts](file:///workspace/packages/compaction/compaction-basic/src/region.ts) |
| 子 Agent 调度事件 | [packages/subagent/subagent/src/descriptor.ts](file:///workspace/packages/subagent/subagent/src/descriptor.ts) |
| 会话查询入口 | [packages/session-query/session-query/src/index.ts](file:///workspace/packages/session-query/session-query/src/index.ts) |
| 真实日志示例（含压缩全流程） | [examples/headless-agent/tests/snapshots/compaction-recovery/session.jsonl](file:///workspace/examples/headless-agent/tests/snapshots/compaction-recovery/session.jsonl) |

### 10.3 动手验证命令

```sh
# 校验自动生成的事件目录与源码一致
pnpm run doc-sync

# 重新生成事件目录
pnpm run gen-persistence-catalog

# 无 key 的 ACP/headless 重放 vs 预期输出（可加 -t <name> 过滤）
pnpm run test:snapshot
```

---

## 11. 一页纸总结

- **唯一真相源**：`~/.dsh/<project>/<sessionId>/session.jsonl`，NDJSON，只追加，永不删除。
- **三层结构**：事件日志（存储）→ Surface（有序视图）→ Message[]（发给 LLM 的历史）。后两层都是算出来的。
- **什么都被记录**：system prompt（`request/header`）、思维链（`assistant/chunk` + `assistant/message`）、工具调用与结果（`tool/call` + `tool/result`）、子 Agent 调度（`subagent/descriptor`）、上下文注入（`user/message` 带 `source`）、压缩（`compaction/*`）。
- **压缩不删历史**：追加 `compaction/start` + `compaction/summary` + 一条带 `surfaceOp:{op:'replace'}` 的 `user/message` + `compaction/end`。原始事件原样留在磁盘。
- **投影重放**：`foldSurface` 遍历事件，遇 `append` 就 `push`，遇 `replace` 就 `splice`；`deriveMessages` 把 surface 投影成 LLM 消息。
- **崩溃恢复**：torn-tail 截断 + 孤儿锁检测 + seq 连续性校验 + 版本守卫。崩溃留下可识别痕迹，不是错误状态。
- **移植到 Claude Code**：照搬"只追加日志 + 派生视图 + replace 压缩"三件套，每个会话一个日志文件，header 放事件流之外，跨会话查询只读 header。
