# Standard 模式如何利用插件实现 Harness 控制

> 说明：deepseek-harness 中 `standard` preset（标准模式）的插件如何协同，实现对 Agent 的 Harness 控制。
> 对应声明文件：`packages/bundle/web-app/presets/standard.patch.yml`。

## 1. 控制平面总览

DSH 对 Agent 的控制分两层，standard 模式 = 一个固定的「底盘」+ 一套具体的「组合」：

- **Host 平面（底盘，始终开启，与 preset 无关）**：执行驱动、循环守卫（guard）、工具执行器、会话持久化。所有 preset 共享。
- **Preset 平面（standard 组合出来的）**：工具目录、系统提示词、计划模式、上下文压缩、委派编排、任务跟踪。由 `standard.patch.yml` 声明的插件列表提供。

关键设计点在 `packages/bundle/web-app/cordis.patch.yml` 第 509 行起的注释：**"the agent plane moves behind agent presets"** —— Web 界面把 host 层的 agent 相关行全部 `disabled: true`，改由每个会话挂载一个 preset 来提供 Agent 的组合。

## 2. Host 平面：执行驱动与守卫（底盘）

### 2.1 执行驱动：agent-loop

- 包：`@deepseek-ai/dsh-agent-loop`（`packages/core/agent-loop/`）。
- 具体实现 `ReactLoopAgent`：循环执行「组装上下文 → 调用模型 → 执行工具调用 → 把结果写回会话 → 下一步」。
- 这是唯一的执行驱动，standard 模式没有替换它，只是替换了它看到的**工具目录**和**系统提示词**。
- 每个 step 发送：会话派生历史 + 可见工具 schema；工具调用走 `ctx.tools.execute()`（受守卫的流水线）。

### 2.2 循环守卫：guard 家族

- 包组：`packages/guard/`，两个插件都在 `dsh` base bundle 里默认启用，**不属于 standard preset**，对所有模式生效：
  - `repeat-tool-reminder`（`@deepseek-ai/dsh-repeat-tool-reminder`）：检测模型连续用相同参数重复调用同一工具，在第 3 / 5 / 8 次重复时提醒它换个思路或结束，防止卡死循环。**纯提醒，不阻止**。
  - `timeout-policy`（`@deepseek-ai/dsh-tool-call-timeout-policy`）：给声明了超时的工具调用加协作式超时，超时后返回结构化 `TOOL_TIMEOUT` 错误。**协作式**：只通过 `exec.signal` 通知，不硬杀。

## 3. Preset 平面：standard 的组合（核心）

`standard.patch.yml` 里 `config.plugins` 列表按功能分 6 组。

### 3.1 身份与提示词

- `persona`（`@deepseek-ai/dsh-persona`）：`prefix: You are a coding agent powered by the {{model}} model.` —— 注册为系统提示词的 `deployment:persona-prefix` section。
- `agent-instructions`（`@deepseek-ai/dsh-agent-instructions`）：注入部署的 agent 指令（`maxBytes` 上限）。
- `time-context`（`@deepseek-ai/dsh-time-context`）：注入当前时间上下文。

### 3.2 工具目录（能力边界）

- shell：`tool-bash` / `tool-pwsh`
- 文件：`tool-fs` / `tool-fs-search`
- 后台任务：`tool-jobs`
- 日程：`tool-schedule`
- 技能：`skill-filesystem` / `tool-skill`
- 搜索：`tool-web`
- 自省与扩展：`tool-cordis`
- 交付：`present` / `tool-office`
- 插件管理：`tool-plugin-manager`

这些插件把工具注册进 `ctx.tools`，loop 每步把可见工具 schema 发给模型——**这就是"能力边界"的控制**。

### 3.3 引导式控制：plan-mode

- `planning` 组（`cordis:group`，`isolate: { planMode: true }`）里挂 `plan-mode`。
- 生效时注入一段 `section`：指示模型"先探索、只读、不改文件、产出完整计划、通过 `exit_plan_mode` 提交"。
- **工具目录不变**（为了 request-cache 稳定），但提示词覆盖了任何诱导使用变更工具的指引。
- 这是"引导式"控制：模型能力没变，行为被提示词约束。

### 3.4 上下文控制：compaction

- `compaction` 组：`compaction-basic` + `command-compact` + `tool-result-pruner`。
- 上下文接近窗口时压缩/总结历史，剪裁超大工具结果（`thresholdChars: 8192`），防止上下文溢出。

### 3.5 委派与编排：delegation

- `delegation` 组（`isolate: { workflowEngine: true }`）：
  - `tool-subagent-control` / `tool-subagent`（spawn）/ `tool-subagent-fork`（fork）：让 Agent 派生子 Agent。
  - `workflow-ptc` + `tool-workflow`：让模型写确定性 JS 编排脚本，引擎解析校验后执行到完成（`agent` / `parallel` / `pipeline`）。
- 这是"控制边界扩展"：Agent 不再单打独斗，可以并行、fork、编排。

### 3.6 任务跟踪与人机交互

- `tool-todo`（todo_write）：模型维护任务清单，跟踪实施进度。
- `command-goal` / `tool-goal`：会话目标。
- `tool-ask-user`：向用户提问，把必须由用户拍板的选择交回。

## 4. 控制如何协同（一个完整 turn）

1. 用户在会话选 standard → `AgentPresetRegistry.select()` 检查会话未开始 → 把 Agent 的 scope 通过 `bindScopeParent` 挂到 standard 的隔离子树 → 记录 `agent-preset/selected` 事件。
2. loop 组装请求：系统提示词（persona + agent-instructions + time-context + 当前 plan-mode section）+ 工具 schema（来自 preset 工具目录）。
3. 模型返回工具调用 → 走 `ctx.tools.execute()` 守卫流水线（timeout-policy 加超时，repeat-tool-reminder 在 post-execute 检测重复）。
4. 结果写回 session log，进入下一步，直到模型不再调用工具或 turn 结束。
5. 后台：compaction 组在上下文超限时压缩；guard 在模型卡循环时提醒；subagent / workflow 在模型选择委派时派生子 Agent。

## 5. 关键机制速查

| 控制维度 | 插件 / 包 | 作用 |
|---------|-----------|------|
| 执行驱动 | agent-loop（ReactLoopAgent） | 唯一循环驱动，host 平面 |
| 循环守卫 | repeat-tool-reminder / timeout-policy | 防卡死、防挂起，host 平面 |
| 能力边界 | preset 工具目录 | 决定 Agent 能调用哪些工具 |
| 行为引导 | persona / plan-mode / agent-instructions | 系统提示词约束行为 |
| 上下文控制 | compaction 组 | 压缩历史、剪裁结果 |
| 编排扩展 | subagent / workflow | 派生、并行、确定性流程 |
| 任务跟踪 | todo / goal | 任务清单与目标 |
| 人机交互 | ask-user | 关键决策交回用户 |
