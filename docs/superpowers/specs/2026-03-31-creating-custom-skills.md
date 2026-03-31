# 创建自定义 Skills - 完整指南

**日期**: 2026-03-31
**版本**: 1.0
**作者**: Claude Code Research

## 执行摘要

Skills 是可重用的工作流程指南，帮助 AI 遵循最佳实践。本文档提供创建、测试和部署自定义 Skills 的完整指南，基于 Test-Driven Development 原则。

---

## 1. Skill 基础概念

### 1.1 什么是 Skill？

**Skill 是**:
- ✅ 可重用的技术、模式或工具的参考指南
- ✅ 帮助未来的 AI 实例找到并应用有效方法
- ✅ 结构化的工作流程文档

**Skill 不是**:
- ❌ 一次性问题解决方案的叙述
- ❌ 项目特定的配置（应放在 CLAUDE.md）
- ❌ 可通过自动化工具强制执行的规则（应使用 Hooks）

### 1.2 Skill 类型

| 类型 | 描述 | 示例 |
|------|------|------|
| **Technique** | 具体的步骤方法 | condition-based-waiting, root-cause-tracing |
| **Pattern** | 思考问题的方式 | flatten-with-flags, test-invariants |
| **Reference** | API 文档、语法指南 | office-docs, api-reference |

### 1.3 目录结构

```
skills/
├── skill-name/
│   ├── SKILL.md              # 主文件（必需）
│   ├── supporting-file.js    # 辅助脚本（可选）
│   └── references/
│       └── advanced.md       # 深度参考（可选）
```

**原则**:
- 扁平命名空间 - 所有 skills 在一个可搜索的命名空间
- 保持简单 - 只在必要时创建额外文件
- 内联优先 - 小于 50 行的代码模式应内联

---

## 2. TDD 方法创建 Skills

### 2.1 核心原则

**Writing skills IS Test-Driven Development applied to process documentation.**

```
┌─────────────────────────────────────────┐
│   TDD for Skills                        │
├─────────────────────────────────────────┤
│                                         │
│  1. RED: 运行压力场景（没有 skill）      │
│     ↓ 观察失败                          │
│  2. GREEN: 编写 skill                   │
│     ↓ 验证通过                          │
│  3. REFACTOR: 关闭漏洞                  │
│     ↓ 重新验证                          │
│                                         │
└─────────────────────────────────────────┘
```

### 2.2 TDD 映射表

| TDD 概念 | Skill 创建 |
|---------|-----------|
| **测试用例** | 使用 subagent 的压力场景 |
| **生产代码** | Skill 文档 (SKILL.md) |
| **测试失败 (RED)** | Agent 没有 skill 时违反规则（基线） |
| **测试通过 (GREEN)** | Agent 有 skill 时遵守规则 |
| **重构** | 在保持合规的同时关闭漏洞 |

### 2.3 创建流程

**步骤 1: 编写测试场景（先写测试）**
```markdown
测试场景：
任务："实现一个函数来验证邮箱地址"
预期失败行为：
- AI 可能会使用简单的正则表达式
- 没有考虑国际化域名
- 没有处理边界情况
```

**步骤 2: 运行基线测试（观察失败）**
- 使用 subagent 执行任务
- 记录所有违规行为
- 文档化 agent 使用的合理化借口

**步骤 3: 编写 Skill（最小代码）**
- 只解决观察到的具体违规
- 不要过度设计
- 保持简洁

**步骤 4: 验证通过（观察通过）**
- 重新运行测试场景
- 确认 agent 现在遵守规则
- 检查是否引入新问题

**步骤 5: 重构（关闭漏洞）**
- 寻找新的合理化借口
- 堵塞漏洞
- 重新验证

---

## 3. SKILL.md 结构详解

### 3.1 Frontmatter (YAML)

**必需字段**:
```yaml
---
name: skill-name-with-hyphens
description: Use when [specific triggering conditions]
---
```

**命名规则**:
- 只使用字母、数字和连字符
- 不使用括号或特殊字符
- 保持简洁和描述性

**Description 关键原则**:

⚠️ **CRITICAL**: Description = When to Use, NOT What the Skill Does

```yaml
# ❌ 错误: 总结了工作流程 - Claude 可能会遵循这个而不是读取 skill
description: Use when executing plans - dispatches subagent per task with code review between tasks

# ❌ 错误: 太多过程细节
description: Use for TDD - write test first, watch it fail, write minimal code, refactor

# ✅ 正确: 只有触发条件，没有工作流程总结
description: Use when executing implementation plans with independent tasks in the current session

# ✅ 正确: 只有触发条件
description: Use when implementing any feature or bugfix, before writing implementation code
```

**为什么这很重要**:
> 测试表明，当 description 总结了 skill 的工作流程时，Claude 可能会遵循 description 而不是读取完整的 skill 内容。Description 说"在任务之间进行代码审查"导致 Claude 只做一次审查，即使 skill 的流程图清楚地显示了两次审查。

**Description 最佳实践**:
- 以 "Use when..." 开头
- 描述具体的触发条件、症状和上下文
- 包含特定的情况和场景
- 不要总结 skill 的过程或工作流程
- 保持在 500 字符以内

### 3.2 Skill 主体结构

```markdown
---
name: example-skill
description: Use when [specific triggering conditions and symptoms]
---

# Skill Name

## Overview
1-2 句话说明这是什么和核心原则。

## When to Use
[如果决策不明显，可以放小的内联流程图]

- 症状和用例的要点列表
- 何时**不**使用

## Core Pattern (用于 techniques/patterns)
前后代码对比

## Quick Reference
常见操作的表格或要点，便于扫描

## Implementation
简单模式的内联代码
重度参考或可重用工具的文件链接

## Common Mistakes
常见错误 + 修复方法

## Real-World Impact (可选)
具体的结果和影响
```

---

## 4. Claude Search Optimization (CSO)

### 4.1 Rich Description Field

**目的**: Claude 读取 description 来决定为给定任务加载哪些 skills

**关键**: Description 应回答 "我应该现在读取这个 skill 吗？"

**示例对比**:

```yaml
# ❌ 错误: 太抽象，模糊，不包含何时使用
description: For async testing

# ❌ 错误: 第一人称
description: I can help you with async tests when they're flaky

# ❌ 错误: 提到技术但 skill 不是特定的
description: Use when tests use setTimeout/sleep and are flaky

# ✅ 正确: 以 "Use when" 开头，描述问题，没有工作流程
description: Use when tests have race conditions, timing dependencies, or pass/fail inconsistently

# ✅ 正确: 技术特定的 skill 有明确的触发器
description: Use when using React Router and handling authentication redirects
```

### 4.2 Keyword Coverage

**原则**: 使用自然语言描述问题，而不是特定技术

```yaml
# ❌ 错误: 技术特定的关键词
description: Use when you see setTimeout or sleep in tests

# ✅ 正确: 问题导向的描述
description: Use when tests have race conditions or timing dependencies
```

**原因**: 
- 技术会变化（setTimeout → requestAnimationFrame）
- 问题模式持久（race conditions 永远存在）

### 4.3 内容格式化

**使用标题**:
```markdown
## Core Pattern
## Common Mistakes
## Quick Reference
```

**原因**: Claude 会扫描标题来定位相关内容

**使用表格**:
```markdown
| Scenario | Approach |
|----------|----------|
| Race condition | Use condition-based waiting |
| Flaky test | Add retry logic |
```

**原因**: 表格便于快速扫描和理解

---

## 5. 实战案例

### 5.1 案例 1: 创建一个简单的 Skill

**场景**: 团队经常在代码中留下 console.log()

**步骤 1: 创建测试场景**
```markdown
任务: "添加一个函数来计算两个数的平均值"
预期失败: AI 可能会添加 console.log() 进行调试
```

**步骤 2: 编写 Skill**

```markdown
---
name: no-console-log
description: Use when writing JavaScript/TypeScript code for production
---

# No Console.log in Production Code

## Overview
Remove all debugging statements before considering code complete.

## When to Use
- Writing production JavaScript/TypeScript code
- Before marking a task as complete
- When code will be committed to version control

## What to Remove
- `console.log()` statements
- `console.debug()` statements
- `console.warn()` for non-critical warnings
- `debugger;` statements

## Quick Reference

| Statement | Action |
|-----------|--------|
| `console.log()` | Remove |
| `console.error()` | Keep (for error handling) |
| `console.warn()` | Evaluate (keep if important) |
| `debugger;` | Remove |

## Common Mistakes

❌ **Wrong**: Leaving debug code with comment
```javascript
// TODO: Remove this later
console.log('debug info', data);
```

✅ **Right**: Remove completely
```javascript
// No debug statements
```

## Implementation

Before committing:
1. Search for `console.log` in codebase
2. Remove all instances
3. Verify tests still pass
4. Commit clean code
```

**步骤 3: 测试验证**
- 运行相同的任务
- 验证 AI 不再添加 console.log()
- 确认代码质量提升

### 5.2 案例 2: 创建复杂的工作流 Skill

**场景**: 团队需要标准化的 API 错误处理

```markdown
---
name: api-error-handling
description: Use when implementing API endpoints or handling HTTP errors
---

# API Error Handling Standard

## Overview
Consistent error handling across all API endpoints for better debugging and user experience.

## When to Use
- Implementing new API endpoints
- Handling HTTP errors in frontend
- Building error responses
- Logging API failures

## Error Response Format

### Standard Structure
```json
{
  "success": false,
  "error": {
    "code": "ERROR_CODE",
    "message": "Human-readable message",
    "details": {},
    "timestamp": "2026-03-31T10:00:00Z",
    "requestId": "req_abc123"
  }
}
```

## Error Codes Hierarchy

| Code | HTTP Status | Description |
|------|-------------|-------------|
| `VALIDATION_ERROR` | 400 | Invalid input data |
| `AUTHENTICATION_ERROR` | 401 | Missing/invalid auth |
| `AUTHORIZATION_ERROR` | 403 | Insufficient permissions |
| `NOT_FOUND` | 404 | Resource doesn't exist |
| `RATE_LIMIT_ERROR` | 429 | Too many requests |
| `INTERNAL_ERROR` | 500 | Server-side failure |

## Implementation Pattern

```typescript
// Error handler middleware
export function handleAPIError(error: unknown, req: Request, res: Response) {
  const requestId = req.headers['x-request-id'];
  
  if (error instanceof ValidationError) {
    return res.status(400).json({
      success: false,
      error: {
        code: 'VALIDATION_ERROR',
        message: error.message,
        details: error.details,
        timestamp: new Date().toISOString(),
        requestId
      }
    });
  }
  
  // ... handle other error types
}
```

## Common Mistakes

❌ **Wrong**: Exposing internal errors
```typescript
res.status(500).json({ error: err.message });
```

✅ **Right**: Generic message + detailed logs
```typescript
logger.error('Internal error:', err);
res.status(500).json({
  error: {
    code: 'INTERNAL_ERROR',
    message: 'An unexpected error occurred'
  }
});
```

## Real-World Impact
- Reduced debugging time by 60%
- Improved error tracking
- Better user experience with actionable error messages
```

### 5.3 案例 3: 创建参考型 Skill

**场景**: 团队需要快速查阅常用的 Git 命令

```markdown
---
name: git-quick-reference
description: Use when working with Git operations, branches, or version control
---

# Git Quick Reference

## Overview
Common Git commands and workflows for daily development.

## Quick Reference Table

### Basic Operations
| Task | Command |
|------|---------|
| Initialize repo | `git init` |
| Clone repo | `git clone <url>` |
| Check status | `git status` |
| Stage changes | `git add <file>` |
| Commit | `git commit -m "message"` |

### Branch Operations
| Task | Command |
|------|---------|
| Create branch | `git checkout -b <branch>` |
| Switch branch | `git checkout <branch>` |
| List branches | `git branch -a` |
| Delete branch | `git branch -d <branch>` |

### Remote Operations
| Task | Command |
|------|---------|
| Add remote | `git remote add origin <url>` |
| Push branch | `git push -u origin <branch>` |
| Pull changes | `git pull origin <branch>` |
| Fetch updates | `git fetch --all` |

## Common Workflows

### Feature Branch Workflow
```bash
# Create feature branch
git checkout -b feature/new-feature

# Make changes and commit
git add .
git commit -m "Add new feature"

# Push to remote
git push -u origin feature/new-feature

# Create PR and merge

# Clean up
git checkout main
git pull
git branch -d feature/new-feature
```

## Common Mistakes

❌ **Wrong**: Committing to main directly
```bash
git checkout main
# make changes
git commit -m "fix"
```

✅ **Right**: Use feature branches
```bash
git checkout -b fix/issue-123
# make changes
git commit -m "Fix issue #123"
```
```

---

## 6. 测试和验证

### 6.1 测试策略

**压力测试场景**:
1. **简单场景** - 基本功能是否工作
2. **边界情况** - 边缘条件是否处理
3. **冲突场景** - 多个 skills 是否冲突
4. **压力场景** - 高负载下是否稳定

### 6.2 Subagent 测试

```markdown
测试脚本:

1. 创建测试任务
   "实现一个用户注册功能，包含邮箱验证"

2. 启动 subagent（没有 skill）
   记录所有违规行为

3. 添加 skill
   重新运行相同任务

4. 对比结果
   - 违规是否消除？
   - 是否引入新问题？
   - 代码质量是否提升？
```

### 6.3 验证清单

- [ ] Description 只包含触发条件
- [ ] Skill 结构清晰完整
- [ ] 包含具体的代码示例
- [ ] 列出常见错误
- [ ] 经过 subagent 测试验证
- [ ] 文档化测试结果

---

## 7. 部署和维护

### 7.1 部署位置

**个人 Skills**:
- Claude Code: `~/.claude/skills/`
- Codex: `~/.agents/skills/`

**团队 Skills**:
- 项目级别: `项目/.claude/skills/`
- 组织级别: 通过插件分发

### 7.2 版本管理

```
skills/
├── skill-name/
│   ├── SKILL.md           # v1.0
│   ├── SKILL.v2.md        # v2.0 (major change)
│   └── CHANGELOG.md       # 变更记录
```

### 7.3 更新策略

1. **小更新** - 直接修改 SKILL.md
2. **大更新** - 创建新版本文件
3. **破坏性变更** - 更新 description 和文档

---

## 8. 高级技巧

### 8.1 条件触发

```markdown
---
name: typescript-performance
description: Use when optimizing TypeScript code performance or encountering performance issues in TypeScript projects
triggers:
  - "optimize"
  - "performance"
  - "slow"
conditions:
  - file_pattern: "**/*.ts"
  - context: "performance optimization"
---
```

### 8.2 依赖管理

```markdown
---
name: advanced-async
description: Use when implementing complex async patterns
requires:
  - basic-async
  - promise-error-handling
---
```

### 8.3 组合 Skills

```markdown
---
name: full-feature-workflow
description: Use when implementing a complete feature from design to deployment
includes:
  - brainstorming
  - test-driven-development
  - code-review-checklist
---
```

---

## 9. 常见陷阱

### 9.1 Description 陷阱

❌ **陷阱**: Description 总结工作流程
✅ **解决**: Description 只描述触发条件

### 9.2 过度设计陷阱

❌ **陷阱**: 试图覆盖所有可能的情况
✅ **解决**: 只解决观察到的具体问题

### 9.3 文档不足陷阱

❌ **陷阱**: 假设用户理解上下文
✅ **解决**: 提供完整的背景和示例

### 9.4 维护缺失陷阱

❌ **陷阱**: 创建后不再更新
✅ **解决**: 定期审查和更新

---

## 10. Skill 开发工具

### 10.1 测试脚本模板

```bash
#!/bin/bash
# test-skill.sh

SKILL_NAME=$1
TEST_CASE=$2

echo "Testing skill: $SKILL_NAME"
echo "Test case: $TEST_CASE"

# Run baseline test (without skill)
echo "Running baseline..."
claude --no-skills "$TEST_CASE" > baseline.log

# Run with skill
echo "Running with skill..."
claude "$TEST_CASE" > with-skill.log

# Compare results
diff baseline.log with-skill.log
```

### 10.2 Skill 验证工具

```python
# validate_skill.py

import yaml
import sys

def validate_skill(skill_path):
    with open(skill_path) as f:
        content = f.read()
    
    # Extract frontmatter
    if not content.startswith('---'):
        print("❌ Missing frontmatter")
        return False
    
    parts = content.split('---', 2)
    if len(parts) < 3:
        print("❌ Invalid frontmatter format")
        return False
    
    frontmatter = yaml.safe_load(parts[1])
    
    # Check required fields
    if 'name' not in frontmatter:
        print("❌ Missing 'name' field")
        return False
    
    if 'description' not in frontmatter:
        print("❌ Missing 'description' field")
        return False
    
    # Validate description
    desc = frontmatter['description']
    if not desc.startswith('Use when'):
        print("⚠️  Description should start with 'Use when'")
    
    if len(desc) > 500:
        print("⚠️  Description too long (>500 chars)")
    
    print("✅ Skill structure valid")
    return True

if __name__ == '__main__':
    validate_skill(sys.argv[1])
```

---

## 总结

### 创建 Skill 的关键原则

1. **TDD 方法** - 先测试，后编写
2. **Description 关键** - 只描述触发条件，不总结工作流程
3. **保持简单** - 只解决观察到的问题
4. **测试验证** - 使用 subagent 验证效果
5. **持续维护** - 定期更新和改进

### 下一步

- 创建你的第一个 Skill
- 使用测试脚本验证
- 收集团队反馈
- 迭代改进

---

**文档版本**: 1.0
**最后更新**: 2026-03-31
**维护者**: Claude Code Research Team
