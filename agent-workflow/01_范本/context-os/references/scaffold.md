# 骨架与模板

一个项目根 = 下面这些文件。**幂等**：文件已存在就跳过，别覆盖（记忆只能由主人写）。

```text
<项目根>/
  ctxos/
    QUOTA.json          配额与口径（唯一事实源）
    INDEX.md            总索引：哪个 AI 的记忆在哪、怎么恢复 ← 开场第一个读
    <ai>/LTM.md         长期记忆
    <ai>/STM.md         短期记忆（六问）
    <ai>/MEMTEXT.md     记忆文本（索引）
  memory/
    journal.jsonl       只读流水（只追加）
```

## QUOTA.json（逐字用，把 `<ai>` 换成真实 AI 名）

```json
{
  "schema": 1,
  "version": "v1.1",
  "date": "<YYYY-MM-DD>",
  "unit": "estimated_tokens",
  "estimator": "ceil(cjk_chars * 1.0 + other_chars / 4)",
  "estimator_note": "cjk_chars = 码位 > 0x2E80 的字符（含全角标点）；粗估，只用于配额判断",
  "total_context": 1000000,
  "partitions": {
    "LTM": {
      "quota": 12000,
      "quota_soft": 8000,
      "quota_base": 10000,
      "zh": "长期记忆（基石·像硬盘索引）",
      "file_pattern": "LTM.md",
      "admission": "只收：红线与裁决、跨会话结论、接口与调用方式、踩过的坑、去哪找的指针。不收：流水账、代码、产物原文、可再查到的细节",
      "compress_order": ["合并同类条目", "删已失效条目（带日期）", "细节搬进产物文件，这里只留指针"]
    },
    "STM": { "quota": 100000, "zh": "短期记忆（工作现场六问）", "file_pattern": "STM.md" },
    "MEMTEXT": { "quota": 50000, "zh": "衔接层·记忆文本（索引）", "file_pattern": "MEMTEXT.md" },
    "EXEC": { "quota": 840000, "zh": "执行空间（用完即忘）", "file_pattern": null }
  },
  "status_rule": "tokens <= quota_soft → ok；quota_soft < tokens <= quota → warn；tokens > quota → over。没写 quota_soft 时按 quota 的 80% 兜底。",
  "close_protocol": [
    "1. 原地覆盖迭代 MEMTEXT.md（索引）",
    "2. 回填 STM.md（六问）",
    "3. 只把『以后还用得上』的结论进 LTM.md",
    "4. 记账：任务板状态 + journal.jsonl 追加一行"
  ],
  "iteration_rule": "单份、原地迭代：不建 .bak、不带时间戳副本、不做版本目录、不回滚。",
  "ais": {
    "<ai>": { "zh": "<该 AI 的角色>", "root": "ctxos/<ai>" }
  }
}
```

要点：
- `EXEC.file_pattern` 必须是 `null`（执行空间不落文件、不计量）；它的配额＝`total_context` 减去所有被计量分区之和（计量结果里的 `summary.exec_space`）。
- 某个 AI 的 LTM 实体在别处时（例如执行器沿用既有路径），在该 AI 的配置里加 `"ltm_real_path": "memory/<文件>.md"`，计量器会改去读它。
- LTM 用「软线/硬顶」两档而不是单一配额：这是「动态容量」的落地方式。

## INDEX.md（总索引，骨架）

```markdown
总索引 · 记忆在哪（Context OS 入口）
版本 v1.1 / <日期> / 维护：<谁>
读法：**每轮开场第一个读的文件**。读完按索引只加载需要的那一段，不要整目录读。

一、有哪些 AI
| AI | 角色 | 记忆根 | 备注 |
|---|---|---|---|
| <ai> | <角色> | ctxos/<ai>/ | <备注> |

二、分类（每个 AI 同样三件套）
| 代号 | 中文名 | 一句话 |
|---|---|---|
| LTM | 长期记忆 | 资产：红线、裁决、跨会话结论、接口与坑。慢写、慎改 |
| STM | 短期记忆 | 现场：六问 |
| MEMTEXT | 记忆文本 | 路标：记忆在哪、怎么分类、怎么恢复。每轮结束覆盖写 |

三、当前状态指针
| 想知道什么 | 去哪看 |
|---|---|
| 各分区占用/是否该压缩 | 跑 ctx_status（--root <项目根>） |
| 某 AI 的现场与交接 | ctxos/<ai>/STM.md 、 ctxos/<ai>/MEMTEXT.md |

四、硬规则
1. 谁的记忆谁写。2. 交换只走显式通道。3. 单份、原地迭代。4. 开场先读 MEMTEXT，收场最后一步是迭代 MEMTEXT。
```

## 三件套模板

`ctxos/<ai>/LTM.md`

```markdown
# <ai> · 长期记忆（LTM）
> 只有 <ai> 自己写这一份。配额：软线 8000 / 硬顶 12000 tokens。
> 只收结构性、最重要的东西，像硬盘索引：红线与裁决、跨会话结论、接口与调用方式、踩过的坑、去哪找的指针。
> 不收：流水账、代码、产物原文、随时能再查到的细节。
> 过软线就压缩：① 合并同类 ② 删已失效（带日期） ③ 细节搬进产物文件、这里只留指针。

（尚未写入任何内容）
```

`ctxos/<ai>/STM.md`

```markdown
# <ai> · 短期记忆（STM，工作现场）
> 配额 100k tokens。每轮结束回填变化的部分。

一、你知道什么（环境事实、已探明结论，带日期与实测证据）
- 待填
二、我们要干什么（当前目标与边界）
- 待填
三、上一轮怎么做的（关键实现与决策）
- 待填
四、去哪找（索引：代码/文件/任务单的路径与定位方法）
- 待填
五、现在做到哪
- 待填
六、还剩什么（待办、阻塞、未拍板）
- 待填
```

`ctxos/<ai>/MEMTEXT.md`

```markdown
# <ai> · 记忆文本（MEMTEXT，衔接层·索引）
> 配额 50k tokens。**不是内容，是路标**。每次对话结束**原地覆盖重写**。
> 目标：下一个会话读到这份就能恢复状态（我是谁、在干什么、东西在哪、还剩什么）。

一、开场 3 步
1. 读本文件 2. 按下面索引只加载需要的段 3. 需要更早细节才查上游记录

二、索引（记忆在哪、怎么分类）
| 想知道什么 | 去哪看 |
|---|---|
| 我的长期记忆 | ctxos/<ai>/LTM.md |
| 我的工作现场 | ctxos/<ai>/STM.md |
| 用量与压缩建议 | ctx_status --root <项目根> |

三、当前状态（每次收场更新）
- 正在干什么：待填
- 做到哪了：待填
- 下一步 / 阻塞：待填

四、恢复要点（下次最容易踩的坑）
- 待填
```

## 建完自检

```text
node ctx_status.mjs --root <项目根>
```

期望：`ok:true`，每个 AI 每个分区都有 `tokens/quota/status`，新建的模板文件是 `ok`（模板体积很小），`summary.exec_space` ≈ 1000000 − 已计量之和。若报 `missing`，就是该文件没建对路径。

## 建骨架的那条命令

```text
node ctx_init.mjs --root <项目根>            # 幂等；已存在的文件不动
node ctx_init.mjs --root <项目根> --force    # 才覆盖已有记忆（慎用）
```
