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

# 检查 develop/main 分支最近的 CI 状态
gh run list --branch develop --limit 5

# 如果 develop 分支也有相同失败，说明不是当前 PR 的问题
```

**判断依据**：

- 如果 develop 分支 CI 正常，当前 PR 失败 → **PR 引入的问题**
- 如果 develop 分支也失败，且错误相同 → **非 PR 问题，是已有问题**
- 如果是 flaky test → 需要进一步分析是否与 PR 改动相关

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

[说明判断依据，如 develop 分支 CI 状态、其他 PR 是否有相同问题等]

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

## Paddle 常见 CI 问题

### Pre-commit 失败

```bash
# 本地运行
pre-commit run --all-files
# 或
prek
```

### 单元测试失败

```bash
# 本地复现
pytest path/to/test_file.py::test_name -v --tb=long
```

### Flaky Test

多次运行确认稳定性：

```bash
for i in {1..10}; do
  pytest path/to/test.py::test_name -v || echo "Failed on run $i"
done
```

## 禁止事项

- **禁止通过降低质量门槛来通过 CI**
- **禁止在没有分析的情况下盲目重试**
- **禁止跳过测试来"修复" CI**

## 完成后

确认修复后，使用 `paddle-pull-request` skill 创建或更新 PR。
