---
name: paddle-ci
description: |
  分析 Paddle 相关仓库 PR 的 CI 失败原因，判断是否为 PR 引入的问题，提供诊断报告和修复计划。
---

# Paddle CI 失败分析与修复

## 核心原则

**先诊断，再修复** — 不要在没有充分分析的情况下盲目修改代码。

## 分析流程

### 1. 确定是否为 PR 引入的问题

首先判断 CI 失败是否由当前 PR 引入：

```bash
# 查看 CI 失败的具体 job
gh pr checks

# 检查 develop 分支最近的 CI 状态
gh run list --branch develop --limit 5
```

**判断依据**：

- develop 分支 CI 正常，当前 PR 失败 → **PR 引入的问题**
- develop 分支也失败，且错误相同 → **非 PR 问题，是已有问题**
- 间歇性失败 → **Flaky test，需进一步分析**

### 2. 检查其他 PR 是否有相似问题

```bash
# 查看最近的 PR 及其 CI 状态
gh pr list --state all --limit 10

# 查看特定 PR 的 CI 状态
gh pr checks <PR_NUMBER>
```

如果多个 PR 都有相同的 CI 失败，很可能是：

- 基础设施问题（CI 环境、网络等）
- develop 分支已有的问题
- Flaky test

### 3. 获取失败日志并分析

```bash
# 获取失败日志（限制输出量）
gh run view <RUN_ID> --log-failed 2>&1 | tail -200
```

### 4. 生成分析报告和修复计划

```markdown
## CI 失败分析报告

**PR**: #[PR_NUMBER]
**失败 Job**: [Job 名称]

### 问题归属

- [ ] PR 引入的问题
- [ ] develop 分支已有问题
- [ ] 基础设施/环境问题
- [ ] Flaky test

### 证据

[说明判断依据]

### 错误摘要

[关键错误信息]

### 根因分析

[详细解释]

### 修复计划

1. [具体步骤]
2. [具体步骤]

### 验证方法

- [ ] 本地验证
- [ ] CI 重新运行
```

## Paddle CI 检查项

### PR 模板检查 (CheckPRTemplate)

检查 PR 描述是否符合模板要求。

**失败处理**：

- 按照 [PR 模板](https://github.com/PaddlePaddle/Paddle/blob/develop/.github/PULL_REQUEST_TEMPLATE.md) 补充完整 PR 描述
- 参考 [PR 模板说明](https://github.com/PaddlePaddle/Paddle/wiki/PULL-REQUEST-TEMPLATE--REFERENCE)

### 代码风格检查 (Codestyle-Check)

运行 pre-commit 检查代码风格，包括 clang-format、cpplint、ast-grep 等。

**本地修复**：

```bash
# 安装依赖
pip install pre-commit==2.17.0 cpplint==1.6.0 clang-format==13.0.0

# 运行检查并自动修复
pre-commit run --files $(git diff --name-only origin/develop)

# 或检查所有文件
pre-commit run --all-files
```

### Approval 检查

某些目录的修改需要特定 reviewer 的 approve。

**处理方式**：

- 查看 CI 日志中提示的 required approvers
- 在 PR 中 @ 对应的 reviewer 请求 review

### 主 CI 流水线

| Job | 说明 |
|-----|------|
| **Linux-CPU** | CPU 编译和单测 |
| **Linux-XPU** | 昆仑芯 XPU 测试 |
| **Linux-DCU** | 海光 DCU 测试 |
| **Linux-NPU** | 昇腾 NPU 测试 |
| **Mac-CPU** | macOS 编译测试 |
| **PR-CI-SOT** | SOT (Symbolic OpTest) 测试 |
| **Distribute-stable** | 分布式训练测试 |

### 单元测试失败

```bash
# 本地复现（需要编译 Paddle）
ctest -R test_name -V

# 或使用 pytest
pytest path/to/test_file.py::test_name -v --tb=long
```

### Flaky Test

多次运行确认稳定性：

```bash
for i in {1..10}; do
  ctest -R test_name -V || echo "Failed on run $i"
done
```

## 禁止事项

- **禁止通过降低质量门槛来通过 CI**
- **禁止在没有分析的情况下盲目重试**
- **禁止跳过测试来"修复" CI**

## 参考文档

- [Paddle CI 手册](https://github.com/PaddlePaddle/Paddle/wiki/paddle_ci_manual)
- [PR 模板说明](https://github.com/PaddlePaddle/Paddle/wiki/PULL-REQUEST-TEMPLATE--REFERENCE)

## 完成后

确认修复后，使用 `paddle-pull-request` skill 创建或更新 PR。
