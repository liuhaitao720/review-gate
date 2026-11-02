# review-gate v1.1 — 评审发现核验与销账门禁 Skill

## 这是什么

对任意 diff/PR/存量代码执行「多 agent 找 bug → 独立核验逐条判定 → Important/Nit/Pre-existing 标签排序 → 销账台账四去向」的评审发现核验门禁（单文件 markdown skill）。

核心理念：**发现 ≠ 行动**——评审产出的发现不直接进修复循环：先经独立核验（防误报烧修复轮次），再经销账台账（每条 CONFIRMED 必须落四去向之一，未销账不放行）。

设计对照 [《How the Claude Code team does code review》](https://claude.dev/blog/solving-code-review-at-anthropic/)（claude.dev，2026-10-15）的核验/收敛/fix-at-source 机制，把它们从组织级评审基础设施翻译成单人 + agent 编队可执行的本地流程件；并在 Pre-existing 归因（基线实跑「三证一禁」）上比原文更严格。可作为 [a2a-coder](https://github.com/liuhaitao720/a2a-coder) v3.4+ 的评审核验协同件，也可独立使用。

## v1.0 基线

- **三档深度**：Quick（单发现 agent + 独立核验）/ Standard（≥2 异视角发现 agent 并行 + 逐条核验）/ Deep（+ 安全门 + 存量扫描 + 新旧分开归因）
- **独立核验三态**：CONFIRMED（复读 file:line 原文+失败场景推演成立）/ PLAUSIBLE（方向可疑、列阻断点）/ 误报（附理由）；纯风格/清理类默认丢弃（agent 作者修 nit 的成本 > 人：多一轮 commit+评审）
- **标签与摘要**：Important/Nit/Pre-existing，按严重度排序，摘要以计数开头；零发现必须声明已审范围（零发现 ≠ 没审）
- **销账台账四去向**：①已修复（附验收证据）②有理由关闭 ③Pre-existing（转存量缺陷台账）④升级用户裁决
- **Pre-existing 三证一禁**：基线证据（改动前快照实跑同样红，禁口头自证）+ 独立判定（执行者不得自标）+ 独立台账（不塞 OPEN_ASSUMPTIONS）；存量豁免不得覆盖本次新增接线的装配验收
- **REVIEW.md 约定**：项目级、用户维护，本 skill 只读+建议追加；同类发现 ≥2 次固化进 REVIEW.md（fix at source）
- **单Agent降级模式**：无 spawn 能力时明示次优退化（换焦点重读/多轮异焦点）
- **a2a-coder 协同契约**：版本握手、仅 CONFIRMED 进修复循环、不新增暂停点

## v1.1 新增：收敛与落盘（对照评审结论）

- **PLAUSIBLE 入台账观察区**：每条发现必须有归宿，不留豁免口——观察区留阻断点+复查方式，重跑必复查；复查 2 次仍无法实证 → 有理由关闭或用户裁决，不无限留观
- **重跑收敛**：修复后重跑只对**新 delta** 重新发现 + 复查 backlog，不全量重发现（防「过夜 50 轮、59% 发现为存量旧代码换皮重报」病理）；**轮次帽**——连续 ≥3 轮仍产出新 CONFIRMED → 停止打地鼠，升级用户裁决「换方法还是继续」
- **内容触发升档**：实际档 = max(声明档, 内容档)；diff 命中密钥/权限配置/网络/子进程/自升级脚本/系统提示词即 Deep——**分类看 diff 内容，不看任务声明**；用户显式降档须原话留痕
- **台账落盘与度量**：默认 `.review-gate/ledger.md` 三区（销账/观察/统计）；统计区每 run 一行计数（发现/CONFIRMED/误报/PLAUSIBLE/丢弃/轮次）+ **escaped defect** 记录（销账后仍翻车）作为效果 guardrail + 固化计数跨会话落点；无法落盘必须明示退化，不静默
- **依赖握手补全**：安全门优先 enterprise-secure-dev，未装或版本不足退化内置 /security-review 并明示；a2a-coder 术语加独立使用注解

## 安装（ZCode / Claude Code 类环境）

把 `SKILL.md` 放入目标环境的 skill 目录，例如：

- Windows: `C:\Users\<user>\.claude\skills\review-gate\SKILL.md`
- 或项目级: `<project>/.claude/skills/review-gate/SKILL.md`

重启会话后即可用 `/review-gate` 或在对话中引用调用。

独立使用：直接对任意 diff/分支/PR 调用；与 a2a-coder v3.4+ 协同自动生效（运行时版本握手，未装则显式警告退化，不静默）。

## 许可证

MIT License — 详见 [LICENSE](LICENSE) 与 SKILL.md 文末 License 节。

## 文件

- SKILL.md — skill 本体（单文件自包含，含 MIT 许可证）
