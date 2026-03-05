---
name: analyze-ci
description: |
  分析 PR 的 CI 失败原因，提供诊断报告和修复建议。
  当 PR 的 CI 检查失败、需要排查 GitHub Actions 问题时，优先使用本 skill。
---

# CI 失败分析与修复

## 适用场景

- PR 的 CI 检查失败，需要分析原因
- GitHub Actions 工作流报错或超时
- 测试本地通过但 CI 失败（flaky test）
- 需要理解 CI 日志并定位根因

## 核心原则

**先诊断，再修复** — 不要在没有充分分析的情况下盲目修改代码。

## 工作流程

### 1. 获取 CI 状态

首先确认 GitHub CLI 认证状态，然后获取 CI 失败信息：

```bash
# 检查认证
gh auth status

# 获取当前 PR 的检查状态
gh pr checks

# 获取详细检查信息（JSON 格式）
gh pr checks --json name,state,conclusion,detailsUrl

# 列出最近失败的工作流运行
gh run list --branch $(git branch --show-current) --status failure --limit 5
```

### 2. 获取失败日志

**重要**：始终限制日志输出量，避免信息过载。

```bash
# 获取失败运行的日志（限制最后 200 行）
gh run view <RUN_ID> --log-failed 2>&1 | tail -200

# 获取特定失败 job 的日志
gh run view <RUN_ID> --job <JOB_ID> --log-failed 2>&1 | tail -100

# 智能提取错误信息
gh run view <RUN_ID> --log-failed 2>&1 | grep -B 5 -A 10 -iE "error|fail|exception|traceback|fatal" | head -100
```

### 3. 分类失败类型

根据日志信息，将失败分为以下类别：

| 类型 | 症状 | 典型原因 | 修复方向 |
|------|------|----------|----------|
| **Code Issue** | 测试断言失败、类型错误、lint 错误 | 代码 bug | 修复代码逻辑 |
| **Infrastructure** | 超时、网络错误、OOM、服务不可用 | CI 环境问题 | 重试或调整资源配置 |
| **Flaky Test** | 间歇性失败、重试后通过、本地无法复现 | 竞态条件、时序依赖 | 修复测试稳定性 |
| **Configuration** | 命令未找到、版本不匹配、依赖缺失 | CI 配置与本地不一致 | 更新 workflow YAML |

### 4. 生成诊断报告

对于每个失败，输出以下信息：

```
## CI 失败分析报告

**Workflow**: [工作流名称]
**Run ID**: [运行 ID]
**失败类型**: [Code Issue / Infrastructure / Flaky / Configuration]

### 错误摘要
[关键错误信息，1-3 行]

### 根因分析
**类型**: [具体分类]
**位置**: [文件路径 / 工作流步骤]
**原因**: [详细解释]

### 修复建议
**操作**: [Fix / Retry / Config Change]
[具体修复步骤]

### 验证方法
- [ ] [本地验证步骤]
- [ ] [CI 重新运行命令]
```

### 5. 常见失败模式与修复

#### 5.1 Pre-commit / Lint 失败

```bash
# 本地运行 pre-commit
pre-commit run --all-files

# 或使用 prek（如果项目配置了）
prek

# 自动修复后提交
git add .
git commit -m "style: fix lint errors"
git push
```

#### 5.2 单元测试失败

```bash
# 本地运行失败的测试
pytest path/to/test_file.py::test_name -v

# 运行特定测试类
pytest path/to/test_file.py::TestClassName -v

# 带详细输出
pytest path/to/test_file.py -v --tb=long
```

#### 5.3 Flaky Test 排查

如果测试本地通过但 CI 失败，按以下步骤排查：

```bash
# 1. 多次运行测试确认稳定性
for i in {1..10}; do
  pytest path/to/test.py::test_name -v || echo "Failed on run $i"
done

# 2. 检查是否有竞态条件
#    - 查找 sleep / wait / timeout 相关代码
#    - 检查是否依赖外部服务或网络
#    - 确认测试间是否有状态污染

# 3. 如果无法本地复现，检查 CI 环境差异
#    - Python / CUDA / 依赖版本
#    - 资源限制（内存、CPU）
#    - 并发执行顺序
```

#### 5.4 CI 配置问题

```bash
# 查看 workflow 文件
cat .github/workflows/<workflow>.yml

# 检查环境变量
gh run view <RUN_ID> --json jobs --jq '.jobs[].steps[] | select(.conclusion == "failure")'

# 对比本地与 CI 环境
python --version
pip list | grep <package>
```

### 6. 重试与监控

```bash
# 重新运行失败的 job
gh run rerun <RUN_ID> --failed

# 监控 PR 检查状态
gh pr checks --watch

# 查看运行详情
gh run view <RUN_ID>
```

## 禁止事项

- **禁止通过降低质量门槛来通过 CI**（如降低覆盖率阈值、跳过 lint 规则、禁用安全检查）
- **禁止在没有分析的情况下盲目重试**（Infrastructure 问题除外）
- **禁止跳过测试来"修复" CI**（`@pytest.mark.skip` 不是修复方案）
- **禁止运行无限制的日志命令**（始终使用 `tail -N` 或 `head -N` 限制输出）

## 常用命令速查

| 操作 | 命令 |
|------|------|
| 查看 PR 检查 | `gh pr checks` |
| 列出失败运行 | `gh run list --status failure --limit 5` |
| 查看失败日志 | `gh run view <ID> --log-failed \| tail -200` |
| 重试失败 job | `gh run rerun <ID> --failed` |
| 监控检查状态 | `gh pr checks --watch` |
| 本地运行测试 | `pytest path/to/test.py -v` |
| 运行 pre-commit | `pre-commit run --all-files` |

## 输出规范

- 诊断报告使用中文，除非上游要求英文
- 命令和代码块保持原样（英文）
- 始终提供具体的修复命令，而非泛泛的建议
- 如果无法确定根因，明确说明需要更多信息

## 完成后

1. 确认修复后，使用 `paddle-pull-request` skill 创建或更新 PR
2. 等待 CI 重新运行并确认通过
3. 如果仍然失败，重复分析流程
