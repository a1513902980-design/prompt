---
name: "flash-executor"
description: "把可蒸馏的写码/改写/摘要/结构化任务写成任务单，派给外部廉价模型执行器（4.0flash）多轮执行并落盘记账，调用方只验收；含最高铁律闸门与记忆分区。不用于需调用方自行取证的活。"
---

# flash-executor

把一件「用任务单 + 明确验收标准就能说清楚」的活，下沉给便宜的外部模型执行器；调用方只写任务单与验收。

## 什么时候用

- 写脚本、批量改写、摘要、结构化/翻译/抽取：交付物能写死成验收标准。
- 要把执行器的产出、记忆增量、逐次流水按固定契约落盘，可追溯、可重派。
- 要给多个 AI 各配一套互不写入的长期/短期/衔接记忆。

## 什么时候别用

- 活依赖调用方自己的判断、取证，或需要与用户来回澄清 —— 留在自己手里。
- 需要执行器直接读本机文件、跑命令或联网：**它没有这些能力**（纯对话补全）；它只能「点名要文件」，由驱动把内容贴回去。
- 只是想装外部仓库/包，或想改某个 Agent 的行为。

## 前置条件

- Node.js 可用（脚本只用内置模块，无第三方依赖）。
- 一个 OpenAI 兼容的 `chat/completions` 端点 + 一把 key。key **只经显式输入**：`--key-file <path>`，或环境变量 `SHNAGHIA_API_KEY` / `FLASH_KEY_FILE`。脚本只打印来源，绝不打印或落盘 key 值。
- 一个工作根（`--root`）：脚本只在它下面读写 `out\`、`memory\`、`ctxos\`。
- 调用方必须能为这一单写出**可验收标准**；写不出就别派。

## 怎么调

派一单（多轮，默认 4 轮）：

```text
"$ORKAS_NODE" "$ORKAS_PC_DIR/bin/run-skill.cjs" flash-executor dispatch -- --task <任务单> --root <工作根>
```

自检（不联网、不写盘）：

```text
"$ORKAS_NODE" "$ORKAS_PC_DIR/bin/run-skill.cjs" flash-executor dispatch -- --selftest
```

常用参数：

- `--rounds <n>` 默认 4：多轮上限；执行器可 `<plan>` 分批、`<checkpoint/>` 续跑。
- `--carry <n>` 默认 1：带上最近 n 单的极简上下文（跨单续接）。
- `--fs-root <dir>` 默认与 `--root` 相同：执行器 `<need>` 能点到的文件根（只读、限长、越界拒）。
- `--max-tokens <n>` 默认 4000：**别给小** —— 该端点默认开推理，预算太小时正文会被思考吃光、返回空（驱动会自动翻倍重试同一轮，最多 2 次）。
- `--no-reasoning`：纯格式化/改写类任务可省预算。`--timeout-ms <n>` 默认 180000（单次调用）。
- `--ai <id>` 默认 `executor`：决定读写哪一套记忆。`--memory <file>` 显式指定要注入的长期记忆（隔离测试用）。`--no-memory` / `--no-map` 关闭注入。

## 任务单契约

四段必备：**目标 / 具体要求（编号）/ 交付格式 / 边界**，并自带可验收标准。任务单控制在 2k tokens 以内最稳；**要源码就写路径让它自己 `<need path="..."/>` 要，别整份粘贴**。

交付格式硬约定：正文末尾两个块缺一不可 —— `<result>` 交付物全文（代码给完整文件，不许省略号），`<memory_update>` 1–5 行「以后还用得上」的结论（没有就写「无」）。

## 铁律优先（本 Skill 的核心约束）

驱动每轮都会在**最后一段**注入「最高铁律」，顺序即优先级：它压过执行器自己的长期记忆、自己写的守则、以及任务单。

- 铁律**只能由用户或管理者模型（调用方）修改**；执行器不能改、不能豁免、不能重新解释。
- 执行器可以用 `<self_prompt>` 给自己写工作守则，但只改「怎么干」，不能改交付契约（`<result>`/`<memory_update>`）与铁律。
- 越界的自写提示词会被**拒收**：原文存到 `ctxos\<ai>\SELFPROMPT.rejected.md`，`out\` 与 journal 都记下拒绝原因，并当场回告执行器重写。

## 验收（不能省）

驱动报 `status:"OK"` **只代表回复里有 `<result>` 块**，不代表东西能用。按任务单的验收标准逐条**亲自**跑：能执行的代码真跑，能对账的数字换一个独立实现交叉核对。不通过 → 带**失败原文**写新任务单重派；连续两次不成、或端点返回跑题内容时由调用方接手修，并在记录里写明是谁改的。

## 返回与落盘

- `out\<任务名>.<时间戳>.md`：交付物 + 记忆增量 + 每轮轨迹 + 原始回复。
- `memory\journal.jsonl`：追加一行流水（含 `selfPromptRejected`）。
- 记忆增量自动追加进该执行器自己的长期记忆文件；只由它自己写。
- 返回 JSON 键稳定：`ok / status / model / reportedModel / finish / ms / usage / out / memoryFile / memoryInjected / memoryWritten / selfPromptWritten / selfPromptRejected / rounds / needs / fedChars / resultChars / keySource / exitCode`。
- 退出码：0 成功；2 参数/任务单/密钥缺失；3 HTTP 或网络失败；4 回复里没有 `<result>`；5 轮数用尽仍未交付。

记忆怎么分区、每轮怎么接上、配额与体积口径，读 [记忆分区与衔接](references/context-os.md)。


---

## 在 DeepSeek Harness 下怎么调（本段由 DSH 会话追加，非原作者内容）

原「怎么调」一节写的是 Orkas 的包装器（`$ORKAS_NODE` / `$ORKAS_PC_DIR/bin/run-skill.cjs`）。
DSH 这边没有那套环境变量，**直接跑脚本本体**即可：

```text
node <本 skill 目录>/scripts/dispatch.mjs --task <任务单路径> --root <工作根> [--rounds 4] [--ai executor]
node <本 skill 目录>/scripts/dispatch.mjs --selftest        # 自检：不联网、只写系统临时目录
```

- 密钥解析顺序：`--key-file <path>` > `env SHNAGHIA_API_KEY` > `env FLASH_KEY_FILE` > `D:/deep seek/key.txt`。
  脚本只打印来源，不打印、不落盘 key 值。
- 默认端点 `https://discovery-api.intern-ai.org.cn/v1`、默认模型 `deepseek-v4-flash-0731`，
  与本机 DSH 的 `shnaghia / 4.0flash` 路由是同一个。
- `--max-tokens` 不要给小：该端点默认开推理，预算太小正文会被思考吃光而返回空。
- 退出码：0 成功 / 2 参数或密钥缺失 / 3 HTTP 或网络失败 / 4 没有 `<result>` 块 / 5 轮数用尽。

### 驱动改造：流式 + 空闲超时（本段由 DSH 会话追加）

原驱动用 `stream:false` + **总时长超时**（默认 180000ms）。这个组合在小任务上没事，
但在「生成一份几百行的完整文件」时必然出问题 —— 实测同一个任务：

| 尝试 | 超时设置 | 结果 | 实际耗时 |
| --- | --- | --- | --- |
| 第 1 次 | 180s × 2 次尝试 | 失败，两次都被中止 | 364 秒（= 180×2 + 1.5s） |
| 第 2 次 | 300s | 成功 | 555 秒 |

**根因**：`stream:false` 时服务器在整段生成完之前**一个字节都不发**，客户端没有任何
「它还活着」的信息源，只能退化成「死等固定秒数」—— 而固定秒数是在跟任务大小赌博。

**改法**：`callModel` 改为 `stream:true` 逐块读 SSE，并引入**两层计时器**：

- `--idle-ms`（新增，默认 **60000**）：**空闲超时 / 看门狗**。每收到一个 SSE 事件就重置计时器，
  只有「连续 idleMs 没有新数据」才判定卡死并中止。模型慢慢想十分钟没关系，只要它在吐 token。
- `--timeout-ms`（默认从 180000 提到 **900000**）：**绝对上限**，纯防呆，
  防止对端一直滴水导致永不结束。不再是主力手段。

配套改动：

1. **失败路径保留真实耗时**。原代码失败时返回硬编码的 `ms: 0`，导致「超时」和「秒挂」无法区分
   （这正是当初诊断困难的原因）；现在返回真实毫秒数，并写进错误文本。
2. **429 / 5xx 尊重 `Retry-After`**，没有这个头才用退避（原来是无条件等 1.5 秒）。
   4xx（429 除外）直接返回，不浪费一次重试。
3. 返回里新增 `chunks` 字段 —— **流式事件计数**，可用于确认「心跳真的在跳」。
4. SSE 里 `delta.reasoning_content` 与 `delta.content` 分开累积，
   返回的 `reasoningText` 是模型的思考过程（不计入交付物）。

> 实测：改造后真打一发 `ok=true / ms=731 / rounds=1`，usage 正常返回
> （该端点无需 `stream_options` 就会在流里带用量）。

**参数速查**：`--idle-ms 60000`（活体检测）、`--timeout-ms 900000`（绝对上限）。