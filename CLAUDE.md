# CLAUDE.md

本文件为 Claude Code 在此仓库中工作提供指导。

## 概述

**test-review** 是一个 Claude Code 技能，提供 `/test-review` 命令，通过派遣专门审查代理并行工作，从多个维度审查测试质量。

## 仓库结构

```
skill/
  SKILL.md                  # 技能定义（user-invocable: true）
  references/               # 评分细则、反模式目录、报告模板
CLAUDE.md                   # 本文件
README.md                   # 项目说明
```

## 技能架构

多阶段流水线：

1. **模式选择** — 根据参数自动选择 Plan Review / Quick Mode / Full Mode
2. **上下文收集** — `git diff`、`git log`、`gh pr view`
3. **变更分类** — 扫描 diff，将文件映射到类别（storage-engine、concurrency、crash-durability 等）
4. **代理派遣** — 最多 5 个代理**并行**运行，每个使用 `model: opus`
5. **综合** — 去重、按严重程度排序（blocker > should-fix > suggestion）、生成统一报告

## 审查代理

| 代理 | 触发条件 |
|------|---------|
| `review-test-behavior` | 始终运行（非纯文档/配置变更） |
| `review-test-completeness` | 始终运行（非纯文档/配置变更） |
| `review-test-structure` | 有测试文件变更 |
| `review-test-concurrency` | 检测到并发原语 |
| `review-test-crash-safety` | 检测到 crash/durability 代码 |

## 关键设计决策

- 代理间无状态共享，每个代理使用相同的完整上下文独立运行
- 分类同时检查生产代码和测试代码
- 使用 `gh` CLI 进行 GitHub API 调用，不使用 WebFetch
- 综合过程只去重和排序，不添加新发现也不弱化代理结论
- 假阳性危害小于假阴性 — 有疑问时启动代理
