---
name: review-gate
description: 评审发现核验与销账门禁 skill——对任意 diff/PR/存量代码执行「多 agent 找 bug → 独立核验逐条判定（CONFIRMED 须附复读证据/误报须附理由）→ Important/Nit/Pre-existing 标签排序 → 销账台账四去向」；纯风格/清理类发现默认丢弃（agent 作者修 nit 成本>人）；Pre-existing 须基线证据+独立判定+独立台账；Deep 档含安全门与存量扫描；同类发现重复 ≥2 次固化进项目 REVIEW.md。作为 a2a-coder（v3.4+）的评审核验协同件（调用点=评审门问题清单/收口重跑核验，仅 CONFIRMED 进修复循环），也可独立使用；可复用内置 /code-review 作为发现引擎，不重建轮子。v1.1：PLAUSIBLE 入台账观察区（重跑必复查）、重跑 diff-only+轮次帽、安全敏感模式内容触发升 Deep、台账默认落盘 .review-gate/ledger.md（含运行统计与 escaped defect 基线）。
version: 1.1
author: Liu Haitao
license: MIT
---

# 评审发现核验与销账门禁（review-gate）v1.1

## 定位与协同

- **发现 ≠ 行动**：评审/核验产出的发现不直接进修复循环——先经**独立核验**（防误报烧修复轮次），再经**销账台账**（每条 CONFIRMED 必须落四去向之一），未销账 CONFIRMED = 流程不放行
- **与 a2a-coder 协同**：a2a-coder v3.4+ 在评审门问题清单与收口重跑核验处调用本 skill；输出 CONFIRMED 清单=其修复循环输入。版本握手：运行时读本文件 version，调用方未装/版本不足则显式警告「评审核验退化为主控/子任务自核」，不静默
- **与内置 /code-review 关系**：Standard/Deep 档的发现阶段可复用内置 code review 能力作为引擎之一；本 skill 加核验/标签/销账/固化层，不重建找 bug 轮子
- **独立使用**：用户对任意 diff/分支/PR 直接调用；销账单交用户裁决，或经用户授权由 agent 接手修复
- **术语约定**：OPEN_ASSUMPTIONS/L2.5/L3/终止线/拆解门等 a2a-coder 术语仅在协同场景生效，独立使用可忽略，不影响流程

## 三档深度

| 档 | 内容 | 适用 |
|---|---|---|
| Quick | 单发现 agent + 独立核验 | 小 diff、低风险、机械改动 |
| Standard | ≥2 异视角发现 agent 并行（互不读彼此产出）+ 逐条核验 | 默认档 |
| Deep | Standard + 安全门 + 受影响面存量扫描 + 新 diff 与存量分开归因 | 收口核验、敏感区（自升级/CI/发布脚本/权限文件/系统提示词） |

**档位来源与内容触发升档（v1.1）**：实际档 = max(声明档, 内容档)。

- 声明档：a2a-coder 调用时取其拆解门风险标注（敏感区至少 Deep）；独立使用默认 Standard，用户可指定
- 内容档：diff 命中任一安全敏感模式即 Deep——密钥/凭据读写、权限与安全配置（permissions/防火墙/sudoers）、网络监听与外部请求、子进程与 shell 执行、自升级/发布/CI 脚本、系统提示词。**分类看 diff 内容，不看任务声明**（任务书说「小改动」不豁免）
- 用户显式降档须原话留痕（如「我知道含敏感模式，仍强制 Quick」），记台账统计区；声明档高于内容档时不降

## 核心流程

1. **读入**：diff + 声明的意图（任务书/CORE_INTENT/PR 描述）+ 项目 REVIEW.md（缺失视为空，跳过不阻断）+ 台账观察区 backlog
2. **发现**（多 agent）：每个发现 agent fresh context 读 diff+受影响面，发现须附 file:line + 失败场景（具体输入/状态 → 错误输出）；发现代理间禁止互读产出
3. **核验**（独立核验 agent，与发现 agent 不同实例）：逐条判定——
   - **CONFIRMED**：复读 file:line 原文、失败场景推演成立（核验证据附后）
   - **PLAUSIBLE**：方向可疑但无法独立复现实证（列出阻断点）；落台账**观察区**，不进修复循环，重跑必复查（第 8 步）
   - **误报**：附理由（误读代码/已有防护/行为有意/与声明意图一致）
   - **纯风格/诊断/仅清理类**：默认丢弃（agent 作者会真去修 nit → 多一轮 commit+评审，成本>人）；仅当 REVIEW.md 显式纳入风格要求时保留
4. **标签与排序**：Important / Nit / Pre-existing（判定规则见下节）；按严重度排序；摘要以计数开头（如「3 Important / 1 Nit / 2 Pre-existing」）；**零发现必须明说「本轮零发现（已审范围：…）」——零发现≠没审**
5. **销账台账**：每条 CONFIRMED 必须落四去向之一——①已修复（附验收证据）②有理由关闭（理由留痕）③Pre-existing（转存量缺陷节）④升级用户裁决；修复可由 agent 接手（a2a-coder 场景=进其修复循环，受其终止线约束）。PLAUSIBLE 落观察区。台账默认落盘 `.review-gate/ledger.md`（项目根；是否 gitignore 由用户决定），a2a-coder 场景可由主控指定寄居 checkpoint、三区格式不变；无法落盘时必须明示「本 run 台账不持久，跨会话机制（≥2 固化计数/观察区复查）退化」。台账三区：
   - **销账区**：CONFIRMED 四去向 + 存量缺陷节（Pre-existing 专用）
   - **观察区**：PLAUSIBLE（阻断点 + 下次复查方式）；证据可得即升级 CONFIRMED 重走销账；复查 2 次仍无法实证 → 去向②有理由关闭（阻断点留痕）或④用户裁决，不得无限留观
   - **统计区**：每 run 一行 `发现 N / CONFIRMED x / 误报 y / PLAUSIBLE z / 丢弃 w / 轮次 r`；**escaped defect** 记录（销账①后仍翻车的条目，附教训）= 本 skill 效果的 guardrail；同类发现计数（≥2 触发固化）；用户显式降档原话
6. **安全门**（Deep 档）：安全类 CONFIRMED 未销账不放行（优先调 enterprise-secure-dev；未装或版本不足则退化内置 /security-review 并明示降级）；销账即自动清除
7. **固化（fix at source）**：同类发现累计出现 ≥2 次（计数记统计区，跨 run 持久）→ 输出「建议 REVIEW.md 条目」，经用户确认写入（本 skill 不代写核心判断）
8. **重跑收敛（v1.1）**：修复后/收口重跑核验——只对**新 delta** 重新发现 + 复查未销账项与观察区 backlog，**不全量重发现**（防过夜循环病理：重复轮次里大半发现是首轮已存在的旧代码换皮重报）；全量发现仅 Deep 收口或用户显式要求时执行。**轮次帽**：连续核验 ≥3 轮仍产出新 CONFIRMED → 停止逐条修，升级去向④（用户裁决「换方法还是继续」）；a2a-coder 场景与其终止线取更早触发者

## Pre-existing 硬约束（三证一禁）

1. **基线证据**：同一验收命令对**改动前快照**跑出同样红（stash/checkout 基线实跑），原始输出附后——禁止口头自证
2. **独立判定**：执行者/被评审方不得自标 Pre-existing；由核验 agent 或上层（子任务 Agent/主控）判定
3. **独立台账**：转「存量缺陷台账」（`.review-gate/ledger.md` 销账区存量节，或寄居项目 checkpoint 的存量缺陷节），**不塞 OPEN_ASSUMPTIONS**（其语义是假设/降级方案，混入污染销账口径）
4. **一禁**：存量豁免**不得覆盖本次新增接线/装配点的 L2.5/L3 验收**——新接线盲区必须自身过装配验证，与存量无关

## REVIEW.md 约定（项目级，用户维护）

- 内容：该仓库什么算失败（失败长什么样）/ nit 上限 / 跳过路径（生成物/vendored/测试夹具等）/ 敏感区清单 / 证据要求（file:line）
- 本 skill 对 REVIEW.md **只读 + 建议追加**，核心判断（什么算失败）必须用户确认后写入
- 缺失时视为空文件：风格类丢弃规则退化为「默认全丢」，不阻断流程

## 单Agent降级（无 spawn 能力时，明示次优）

- 核验步退化=同 agent 换焦点重读（落盘发现清单→重读→以「假如我是另一位核验者」逐条证伪），明示不等价独立实例
- 发现步退化=多轮异焦点（≥2 轮：正确性轮/边界轮），轮间落盘+重读
- Pre-existing 独立判定退化=上层（主控/用户）判定，执行者仍不得自标
- 台账三区与落盘规则不退化（不依赖 spawn）

## a2a-coder 协同契约

- **调用点**：主控评审门各焦点产出的问题清单、子任务评审发现、收口重跑核验（敏感区子任务用 Deep 档；内容触发升档规则同样生效）
- **输出对齐**：CONFIRMED 清单（含核验证据）→ 修复循环输入；Pre-existing → 存量缺陷台账；PLAUSIBLE → 观察区（不转发，重跑复查）；误报/风格类不转发
- **不新增暂停点**：本 skill 的「不放行」指 a2a-coder 修复循环继续，非暂停等人工；唯一升级例外=销账去向④（用户裁决），且仅在修复循环触发终止线后到达；轮次帽升级并入此例外（先到先触发）

## 反模式

- 发现未核验直进修复循环（误报烧轮次）
- 执行者/被评审方自标 Pre-existing（自我豁免）
- Pre-existing 无基线原始输出（口头豁免红验收）
- agent 修纯风格 nit 触发额外 commit+评审轮
- 重复报已销账发现
- 重跑全量重发现（旧发现换皮重报；应只审新 delta + backlog）
- 连续多轮新 CONFIRMED 仍逐条修（触轮次帽应升级换方法，不打地鼠）
- PLAUSIBLE 不落观察区 / 重跑不复查 backlog（静默蒸发或无限重复报）
- 报 bug 只报一例、不指认同类（修全类才销账）
- 零发现不声明审过范围
- 敏感内容按声明档走（分类看 diff 不看任务书；显式降档须原话留痕）
- 本 skill 代写 REVIEW.md 核心判断
- 把存量缺陷塞进 OPEN_ASSUMPTIONS
- 台账不落盘且不明示（跨会话机制静默退化）

## 适用边界

- 纯对话/无 diff 产出：不适用
- 小 diff 用 Quick 即可；里程碑收口建议 Deep
- 组织级评审制度（merge queue/灰度/on-call/审批人数制度）不在本 skill 范围——那是基础设施与团队制度层

## License

MIT License — Copyright (c) 2026 Liu Haitao

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
