# test-review

测试评审技能 — Claude Code 多维度测试质量审查工具。

## 功能

从多个维度审查测试用例和测试计划，评估覆盖完整性、断言深度、边界条件、错误处理、突变杀伤强度、不变量保护和外部依赖故障场景。生成结构化报告及可执行的改进建议。

## 三种模式

| 模式 | 触发条件 | 说明 |
|------|---------|------|
| **Plan Review** | 提供测试计划 / 测试点列表（`.md`、`.csv`、`.xlsx`） | 分析 AI 生成的测试点，检查依赖故障覆盖、权重分布、历史缺陷覆盖 |
| **Quick Mode** | 单个测试文件路径 | 单文件深度评审，产出 7 维度评分卡 |
| **Full Mode** | 分支 / PR / 提交范围 / "uncommitted" | 多代理并行评审，基于 triage 的代理调度 |

## 用法

```
/test-review [参数]
```

| 参数 | 效果 |
|------|------|
| `test-file` | 单文件深度评审 (Quick Mode) |
| `branch` / `PR-number` | 多代理并行评审 (Full Mode) |
| `commit-range` | 指定提交范围评审 (Full Mode) |
| `uncommitted` | 评审未提交变更 (Full Mode) |
| `plan-file` | AI 生成测试点走查 (Plan Review) |
| 留空 | 自动检测：有 PR → Full，≤5 测试文件 → Quick，>5 → Full |

## 审查维度 (Quick/Full Mode)

| 维度 | 核心问题 |
|------|---------|
| **Assertion Depth** | 断言是否针对具体值，而非仅检查存在性/真值？ |
| **Input Coverage** | 是否覆盖规范输入、空值、边界、无效和依赖故障场景？ |
| **Error Testing** | 源码中每个可抛出/拒绝的路径是否有对应的错误测试？ |
| **Mock Health** | Mock 是否局限在外部边界，还是泄漏到了内部模块？ |
| **Specification Clarity** | 仅通过测试名称是否能理解完整契约？ |
| **Independence** | 测试是否可以任意顺序运行且互不影响？ |
| **Invariant Protection** | 测试是否编码了架构边界、数据约束和设计不变量？ |

## Plan Review 三项检查

1. **外部依赖故障覆盖** — 对每个外部依赖检查不可用、超时、数据损坏三种故障模式
2. **业务权重分布** — 核心路径测试深度应至少 3 倍于边界场景
3. **历史缺陷交叉验证** — AI 生成的测试点仅覆盖 23-41% 的历史缺陷场景

## 架构

多阶段流水线：上下文收集 → 变更分类 → 代理派遣（最多 5 个并行）→ 结果综合

五个专项代理：
- `review-test-behavior` — 始终运行
- `review-test-completeness` — 始终运行
- `review-test-structure` — 测试文件变更时运行
- `review-test-concurrency` — 检测到并发原语时运行
- `review-test-crash-safety` — 检测到 crash/durability 代码时运行

## 仓库结构

```
skill/
  SKILL.md                  # 技能定义（用户可调用）
  references/
    dimensions.md            # 7 维度评分细则
    anti-patterns.md         # 反模式目录
    report-format.md         # 三种模式报告模板
CLAUDE.md                    # 技能配置
README.md                    # 本文件
```

## 核心原则

- **先理解后评判** — 同时阅读测试代码和生产代码
- **具体化** — 每条发现必须引用具体行号和场景
- **宁可误报不可漏报** — 假阳性危害小于假阴性
- **突变思维** — 测试必须对生产代码的任何有效变更保持敏感
- **依赖故障心态** — 生产故障更多来自依赖失败而非功能缺陷
