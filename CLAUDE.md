# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 概述

这是 Claude Code 的 **test-reviewer** 技能仓库。启动时会安装一个 `/test-review` 用户可调用命令，通过派遣专门的审查代理并行工作，从多个维度审查测试质量。

## 仓库结构

- `docs/SKILL.md` — 活跃的技能定义（`user-invocable: true`）。包含完整工作流：上下文收集、变更分类、代理派遣、结果综合。
- `docs/SKILL (1).md` 到 `(4).md` — 替代版本/草稿，记录了不同的审查哲学（突变杀伤、TDD/BDD 合规、通用覆盖）。

## 技能架构

该技能遵循一个多阶段流水线结构：

1. **上下文确定** — 解析 `$ARGUMENTS`（分支、提交范围、未提交内容或 PR 编号），自动检测 PR、回退到 `develop..HEAD`。
2. **基础分支检测** — 优先 PR 基础分支，否则默认 `develop`。
3. **上下文收集** — 通过 `git diff`、`git log` 和 `gh pr view` 收集 DIFF、CHANGED_FILES、COMMIT_LOG、PR_DESCRIPTION。
4. **分类** — 将文件变更分类（storage-engine、concurrency、crash-durability 等），并将类别映射到 5 个专门的审查代理。
5. **代理派遣** — 最多 5 个代理，必须**并行**运行，每个使用 `model: opus`：
   - `review-test-behavior` — 始终运行
   - `review-test-completeness` — 始终运行
   - `review-test-structure` — 当有测试文件变更时运行
   - `review-test-concurrency` — 当检测到并发原语时运行
   - `review-test-crash-safety` — 当检测到 crash/durability 代码时运行
6. **综合** — 去重、按严重程度排序（blocker > should-fix > suggestion）、跨维度归因、生成统一报告。

## 关键设计决策

- 每个代理都用相同的完整上下文独立运行 — 各代理之间没有跨代理状态共享。
- 分类同时检查**生产代码和测试代码** — 如果生产代码引入了 `synchronized` 但测试未涉及线程安全，并发代理仍会触发。
- 使用 `gh` 命令行进行 GitHub API 调用，不使用 WebFetch。
- 综合过程只去重和排序代理的输出结果 — 不会添加新的发现，也不会弱化代理的结论。
- 危险模式：非关键维度的假阳性比遗漏真实问题的假阴性危害更小。如有疑问，启动该代理。

## 规则文件

技能的定义和执行逻辑完全包含在 `docs/SKILL.md` 中。不存在独立的参考文件或模块 — 所有分类表、代理提示模板和综合逻辑都嵌入在这个文件中。
