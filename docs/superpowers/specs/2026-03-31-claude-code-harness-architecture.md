# Claude Code Harness 系统架构深入分析

**日期**: 2026-03-31
**版本**: 1.0
**作者**: Claude Code Research

## 执行摘要

Claude Code Harness 是一个强大的扩展系统，通过三个核心支柱（Skills、Hooks、Settings）实现工作流自动化、安全控制和最佳实践强制执行。本文档深入分析其内部架构、执行机制和设计原则。

---

## 1. 核心架构概览

### 1.1 三大支柱

```
┌─────────────────────────────────────────────────────────┐
│                   Claude Code Harness                    │
├───────────────────┬─────────────────┬───────────────────┤
│   Skills (技能)   │   Hooks (钩子)  │  Settings (设置)  │
│                   │                 │                   │
│ • 工作流程指南     │ • 事件拦截器    │ • 配置管理        │
│ • 最佳实践强制     │ • 安全检查      │ • 权限控制        │
│ • 自动触发机制     │ • 自动化响应    │ • 环境定制        │
└───────────────────┴─────────────────┴───────────────────┘
```

### 1.2 系统层级

```
用户输入
    ↓
[Settings Layer] - 权限检查、环境配置
    ↓
[Skills Layer] - 工作流程检测和加载
    ↓
[Hooks Layer] - Pre-execution 拦截
    ↓
工具执行
    ↓
[Hooks Layer] - Post-execution 处理
    ↓
返回结果
```

---

## 2. Skills 系统架构

### 2.1 Skill 结构

**文件组织**:
```
skills/
├── skill-name/
│   ├── SKILL.md           # 主技能文件
│   ├── scripts/           # 辅助脚本
│   │   ├── helper.js
│   │   └── validator.py
│   └── references/        # 参考文档
│       └── advanced.md
```

**Skill 元数据** (YAML Frontmatter):
```yaml
---
name: skill-name
description: When and how to use this skill
type: rigid | flexible  # 执行严格程度
---
```

### 2.2 Skill 加载机制

**加载流程**:
```
1. 会话开始
   ↓
2. 扫描所有已安装插件的 skills/ 目录
   ↓
3. 解析 SKILL.md 文件的 frontmatter
   ↓
4. 缓存 skill 元数据到内存
   ↓
5. 等待触发条件
```

**触发机制**:
- **自动触发**: AI 在执行任务前检查 skill 适用性
- **手动触发**: 用户通过 `/skill-name` 命令
- **条件触发**: 基于工具使用、文件类型等

### 2.3 Skill 执行流程

```python
# 伪代码表示
def execute_skill(skill_name, context):
    # 1. 加载 skill 内容
    skill_content = load_skill_file(skill_name)
    
    # 2. 解析指令
    instructions = parse_instructions(skill_content)
    
    # 3. 创建任务列表（如果有 checklist）
    if has_checklist(instructions):
        create_tasks_from_checklist(instructions.checklist)
    
    # 4. 执行指令
    for instruction in instructions:
        execute_instruction(instruction, context)
    
    # 5. 验证结果
    verify_execution_results()
```

### 2.4 Superpowers 核心技能

**工作流程技能**:
1. `brainstorming` - 需求分析和设计
2. `writing-plans` - 实现计划编写
3. `executing-plans` - 计划执行
4. `subagent-driven-development` - 并行开发

**质量保证技能**:
1. `test-driven-development` - TDD 工作流
2. `systematic-debugging` - 系统化调试
3. `verification-before-completion` - 完成验证

**协作技能**:
1. `requesting-code-review` - 代码审查请求
2. `receiving-code-review` - 处理审查反馈
3. `using-git-worktrees` - Git 工作树管理

---

## 3. Hooks 系统架构

### 3.1 Hook 生命周期

```
┌─────────────────────────────────────────────┐
│           Hook 执行生命周期                  │
├─────────────────────────────────────────────┤
│                                             │
│  SessionStart                               │
│      ↓                                      │
│  UserPromptSubmit ← ─ ─ ─ ─ ─ ─ ─ ─ ─ ─   │
│      ↓                                      │
│  PreToolUse ← ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─   │
│      ↓                                      │
│  [Tool Execution]                           │
│      ↓                                      │
│  PostToolUse ← ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─   │
│      ↓                                      │
│  Stop (completion check)                    │
│                                             │
└─────────────────────────────────────────────┘
```

### 3.2 Hook 类型详解

#### PreToolUse Hook
**触发时机**: 工具执行前
**输入数据**:
```json
{
  "hook_event_name": "PreToolUse",
  "tool_name": "Bash",
  "tool_input": {
    "command": "rm -rf /tmp/test"
  }
}
```

**返回格式**:
```json
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "deny"  // or "allow"
  },
  "systemMessage": "⚠️ Dangerous command blocked!"
}
```

#### PostToolUse Hook
**触发时机**: 工具执行后
**用途**: 结果验证、日志记录、状态更新

#### UserPromptSubmit Hook
**触发时机**: 用户提交消息时
**用途**: 输入验证、上下文增强

#### Stop Hook
**触发时机**: AI 准备结束会话时
**用途**: 完成度检查、质量门控

### 3.3 Hookify 插件实现

**核心组件**:

1. **config_loader.py** - 配置加载器
   - 扫描 `.claude/hookify.*.local.md` 文件
   - 解析 YAML frontmatter
   - 构建 Rule 对象列表

2. **rule_engine.py** - 规则引擎
   - 评估规则匹配
   - 支持 6 种操作符：
     - `regex_match` - 正则匹配
     - `contains` - 包含检查
     - `equals` - 精确匹配
     - `not_contains` - 不包含
     - `starts_with` - 前缀匹配
     - `ends_with` - 后缀匹配

3. **Hook Scripts** - 钩子脚本
   - `pretooluse.py` - 工具执行前检查
   - `posttooluse.py` - 工具执行后处理
   - `userpromptsubmit.py` - 用户输入处理

**规则评估流程**:
```python
def evaluate_rules(rules, input_data):
    blocking_rules = []
    warning_rules = []
    
    for rule in rules:
        if rule_matches(rule, input_data):
            if rule.action == 'block':
                blocking_rules.append(rule)
            else:
                warning_rules.append(rule)
    
    # 优先处理阻止规则
    if blocking_rules:
        return block_operation(blocking_rules)
    
    # 警告规则允许操作继续
    if warning_rules:
        return warn_user(warning_rules)
    
    return allow_operation()
```

### 3.4 性能优化

**Regex 缓存**:
```python
@lru_cache(maxsize=128)
def compile_regex(pattern: str) -> re.Pattern:
    return re.compile(pattern, re.IGNORECASE)
```

**事件过滤**:
- 只加载匹配当前事件类型的规则
- 避免不必要的规则评估

**错误处理**:
- Hook 失败不应阻止操作（fail-open）
- 错误日志记录到 stderr

---

## 4. Settings 系统架构

### 4.1 配置层级

```
优先级（从高到低）:
1. 命令行参数
2. 环境变量
3. 项目设置 (.claude/settings.json)
4. 本地设置 (~/.claude/settings.local.json)
5. 全局设置 (~/.claude/settings.json)
6. 默认值
```

### 4.2 配置结构

**settings.json**:
```json
{
  "model": "claude-sonnet-4-6",
  "env": {
    "ANTHROPIC_API_KEY": "sk-..."
  },
  "enabledPlugins": {
    "superpowers@claude-plugins-official": true
  }
}
```

**settings.local.json**:
```json
{
  "permissions": {
    "allow": [
      "Bash(safe-command)",
      "Read(/project/**)"
    ],
    "deny": [
      "Bash(rm -rf /**)"
    ]
  }
}
```

### 4.3 权限系统

**权限模型**:
```
Permission = ToolName + Parameters + Constraint

示例:
- Bash(git status)          # 允许 git status
- Read(/project/**/*.ts)    # 允许读取项目 TypeScript 文件
- Write(/tmp/**)            # 允许写入临时目录
```

**权限检查流程**:
```
工具调用请求
    ↓
检查 settings.local.json 中的 permissions.allow
    ↓
检查 permissions.deny
    ↓
匹配？ → 允许/拒绝
    ↓
不匹配？ → 询问用户
```

---

## 5. 插件系统架构

### 5.1 插件结构

```
plugin-name/
├── .claude-plugin/
│   ├── plugin.json        # 插件元数据
│   └── marketplace.json   # 市场信息
├── skills/                # 技能目录
│   └── skill-name/
│       └── SKILL.md
├── hooks/                 # 钩子目录
│   ├── hooks.json         # 钩子配置
│   └── scripts/           # 钩子脚本
│       ├── pretooluse.py
│       └── posttooluse.py
├── CLAUDE.md              # Claude Code 指令
└── GEMINI.md              # Gemini CLI 指令
```

### 5.2 插件加载流程

```
1. 扫描插件目录
   ├─ ~/.claude/plugins/cache/
   └─ 项目 .claude/plugins/
    ↓
2. 读取 plugin.json
    ↓
3. 加载 hooks.json
    ↓
4. 扫描 skills/ 目录
    ↓
5. 加载 CLAUDE.md/GEMINI.md
    ↓
6. 注册插件到运行时
```

### 5.3 插件隔离

**环境变量注入**:
```bash
CLAUDE_PLUGIN_ROOT=/path/to/plugin
```

**路径引用**:
```json
{
  "command": "python3 ${CLAUDE_PLUGIN_ROOT}/hooks/script.py"
}
```

---

## 6. 事件系统与消息流

### 6.1 事件总线

```
┌─────────────────────────────────────────┐
│           Event Bus                     │
├─────────────────────────────────────────┤
│                                         │
│  User Action ────→ Event ────→ Hooks   │
│                      ↓                  │
│                   Skills                │
│                      ↓                  │
│                   Tools                 │
│                      ↓                  │
│                   Response              │
│                                         │
└─────────────────────────────────────────┘
```

### 6.2 消息格式

**标准 Hook 输入**:
```json
{
  "hook_event_name": "PreToolUse",
  "tool_name": "ToolName",
  "tool_input": {
    "param1": "value1",
    "param2": "value2"
  },
  "context": {
    "session_id": "xxx",
    "user_id": "yyy"
  }
}
```

**标准 Hook 输出**:
```json
{
  "systemMessage": "Optional message to AI",
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "allow|deny"
  },
  "decision": "block|allow",
  "reason": "Explanation for block"
}
```

---

## 7. 性能与可靠性

### 7.1 性能优化策略

1. **缓存机制**
   - Regex 编译缓存（LRU, max 128）
   - Skill 元数据缓存
   - 插件清单缓存

2. **延迟加载**
   - Skill 内容按需加载
   - Hook 规则动态加载

3. **并行处理**
   - 独立 hook 并行执行
   - Skill 检查并行化

### 7.2 错误处理

**Fail-Open 原则**:
```python
try:
    result = evaluate_hooks(input_data)
except Exception as e:
    # Hook 失败不应阻止操作
    log_error(e)
    return allow_operation()
```

**错误恢复**:
- Hook 超时机制（默认 10 秒）
- 降级策略（跳过失败的 hook）
- 错误日志和监控

---

## 8. 安全模型

### 8.1 多层防御

```
Layer 1: Settings 权限控制
    ↓
Layer 2: Hooks 安全检查
    ↓
Layer 3: Skill 最佳实践
    ↓
Layer 4: 运行时验证
```

### 8.2 信任边界

**受信任**:
- 官方插件市场
- 用户显式安装的插件
- 项目级别的配置

**不受信任**:
- 用户输入
- 外部文件内容
- 网络数据

### 8.3 沙箱机制

**执行隔离**:
- Hook 脚本独立进程
- 限制文件系统访问
- 网络访问控制

---

## 9. 扩展性与定制

### 9.1 扩展点

1. **自定义 Skills**
   - 创建 SKILL.md 文件
   - 定义触发条件
   - 编写工作流程

2. **自定义 Hooks**
   - 实现 hook 脚本
   - 配置 hooks.json
   - 定义拦截规则

3. **自定义插件**
   - 打包 skills + hooks
   - 创建 plugin.json
   - 发布到市场

### 9.2 集成模式

**MCP (Model Context Protocol)**:
```
Claude Code ← → MCP Server ← → External Tools
```

**Webhook 集成**:
```
Hook Event → HTTP Request → External Service
```

---

## 10. 未来演进方向

### 10.1 计划增强

1. **技能市场**
   - 去中心化技能共享
   - 版本管理和依赖解析

2. **高级 Hooks**
   - 条件链和状态机
   - ML 驱动的规则生成

3. **性能优化**
   - 增量加载
   - 预测性缓存

### 10.2 社区生态

- 技能标准化
- 最佳实践指南
- 培训和认证

---

## 附录 A: 关键源码分析

### Hookify 规则引擎核心代码

```python
class RuleEngine:
    def evaluate_rules(self, rules, input_data):
        blocking_rules = []
        warning_rules = []
        
        for rule in rules:
            if self._rule_matches(rule, input_data):
                if rule.action == 'block':
                    blocking_rules.append(rule)
                else:
                    warning_rules.append(rule)
        
        # Blocking rules take priority
        if blocking_rules:
            return self._build_block_response(blocking_rules)
        
        # Warnings allow operation to continue
        if warning_rules:
            return self._build_warn_response(warning_rules)
        
        return {}
```

### Skill 加载伪代码

```javascript
async function loadSkills(pluginDir) {
  const skillsDir = path.join(pluginDir, 'skills');
  const skills = await fs.readdir(skillsDir);
  
  const skillRegistry = {};
  
  for (const skillName of skills) {
    const skillPath = path.join(skillsDir, skillName, 'SKILL.md');
    const content = await fs.readFile(skillPath, 'utf-8');
    const metadata = parseFrontmatter(content);
    
    skillRegistry[skillName] = {
      name: metadata.name,
      description: metadata.description,
      path: skillPath,
      content: content
    };
  }
  
  return skillRegistry;
}
```

---

## 附录 B: 配置示例

### 完整的 Hook 规则示例

```markdown
---
name: block-dangerous-commands
enabled: true
event: bash
action: block
conditions:
  - field: command
    operator: regex_match
    pattern: rm\s+-rf\s+(/|\*|~)
---

🛑 **危险操作被阻止**

检测到可能导致数据丢失的命令：
- `rm -rf /` - 删除根目录
- `rm -rf *` - 删除当前目录所有文件
- `rm -rf ~` - 删除用户主目录

请确认操作意图并使用更安全的替代方案。
```

### 完整的 Skill 元数据示例

```yaml
---
name: test-driven-development
description: Use when implementing any feature or bugfix, before writing implementation code
type: rigid
triggers:
  - "implement"
  - "add feature"
  - "fix bug"
  - "create function"
priority: high
---
```

---

## 附录 C: 故障排查指南

### 常见问题

1. **Skill 未触发**
   - 检查 SKILL.md 文件格式
   - 验证 description 中的触发词
   - 确认插件已启用

2. **Hook 未执行**
   - 检查 hooks.json 语法
   - 验证脚本路径和权限
   - 查看 stderr 日志

3. **权限被拒绝**
   - 检查 settings.local.json
   - 验证权限模式匹配
   - 确认没有冲突规则

---

**文档版本**: 1.0
**最后更新**: 2026-03-31
**维护者**: Claude Code Research Team
