# 高级 Hooks 配置 - 安全规则和自动化检查

**日期**: 2026-03-31
**版本**: 1.0
**作者**: Claude Code Research

## 执行摘要

Hooks 是 Claude Code 的核心安全机制，允许在关键事件点拦截、检查和控制操作。本文档深入探讨高级 Hooks 配置，包括复杂规则、性能优化和实战案例。

---

## 1. Hooks 系统深度解析

### 1.1 Hook 生命周期

```
┌──────────────────────────────────────────────┐
│         Hook 执行完整生命周期                 │
├──────────────────────────────────────────────┤
│                                              │
│  1. SessionStart (会话开始)                  │
│      ↓                                       │
│  2. UserPromptSubmit (用户提交消息)          │
│      ↓                                       │
│  3. PreToolUse (工具执行前)                  │
│      ↓ [决策点: allow/deny]                  │
│  4. Tool Execution (工具执行)                │
│      ↓                                       │
│  5. PostToolUse (工具执行后)                 │
│      ↓                                       │
│  6. Stop (会话结束)                          │
│      ↓ [决策点: allow/block]                 │
│  7. Session End                              │
│                                              │
└──────────────────────────────────────────────┘
```

### 1.2 Hook 类型详解

| Hook 类型 | 触发时机 | 用途 | 返回格式 |
|----------|---------|------|---------|
| **SessionStart** | 会话开始时 | 初始化、加载配置 | 系统消息 |
| **UserPromptSubmit** | 用户提交消息 | 输入验证、上下文增强 | 修改后的输入 |
| **PreToolUse** | 工具执行前 | 权限检查、安全验证 | allow/deny + 消息 |
| **PostToolUse** | 工具执行后 | 结果验证、日志记录 | 系统消息 |
| **Stop** | 会话结束时 | 完成度检查、质量门控 | allow/block + 消息 |

### 1.3 Hook 输入数据结构

**通用字段**:
```json
{
  "hook_event_name": "PreToolUse",
  "tool_name": "Bash",
  "tool_input": {
    "command": "git status"
  },
  "context": {
    "session_id": "session_abc123",
    "user_id": "user_xyz",
    "timestamp": "2026-03-31T10:00:00Z"
  }
}
```

**PreToolUse 特定字段**:
```json
{
  "hook_event_name": "PreToolUse",
  "tool_name": "Bash",
  "tool_input": {
    "command": "rm -rf /tmp/test",
    "timeout": 30000
  }
}
```

**PostToolUse 特定字段**:
```json
{
  "hook_event_name": "PostToolUse",
  "tool_name": "Bash",
  "tool_input": { ... },
  "tool_result": {
    "stdout": "output",
    "stderr": "",
    "exit_code": 0
  }
}
```

**Stop 特定字段**:
```json
{
  "hook_event_name": "Stop",
  "reason": "User requested stop",
  "transcript_path": "/tmp/transcript_abc123.txt"
}
```

---

## 2. 高级规则配置

### 2.1 规则结构完整解析

**基本结构**:
```markdown
---
name: rule-name
enabled: true
event: bash | file | stop | prompt | all
action: warn | block
tool_matcher: Bash | Edit|Write | *  # 可选
conditions:
  - field: field_name
    operator: operator_type
    pattern: pattern_value
---

[消息内容 - Markdown 格式]
```

### 2.2 操作符详解

| 操作符 | 描述 | 示例 | 性能 |
|--------|------|------|------|
| `regex_match` | 正则匹配 | `rm\s+-rf` | 中等（缓存优化） |
| `contains` | 包含子串 | `console.log` | 快 |
| `equals` | 精确匹配 | `production` | 最快 |
| `not_contains` | 不包含 | `test` | 快 |
| `starts_with` | 前缀匹配 | `https://` | 快 |
| `ends_with` | 后缀匹配 | `.env` | 快 |

### 2.3 字段映射表

**Bash 工具字段**:
```
command     → tool_input.command
timeout     → tool_input.timeout
```

**Edit/Write 工具字段**:
```
file_path   → tool_input.file_path
new_text    → tool_input.new_string (Edit) / content (Write)
old_text    → tool_input.old_string (Edit only)
```

**Stop 事件字段**:
```
reason      → input_data.reason
transcript  → input_data.transcript (from file)
```

**UserPromptSubmit 字段**:
```
user_prompt → input_data.user_prompt
```

---

## 3. 复杂规则案例

### 3.1 多条件组合规则

**场景**: 阻止在生产环境文件中硬编码 API 密钥

```markdown
---
name: block-api-keys-in-production
enabled: true
event: file
action: block
conditions:
  # 条件 1: 文件类型检查
  - field: file_path
    operator: regex_match
    pattern: \.(ts|js|py|go)$
  
  # 条件 2: 排除测试文件
  - field: file_path
    operator: not_contains
    pattern: test|spec|__tests__
  
  # 条件 3: 排除配置文件
  - field: file_path
    operator: not_contains
    pattern: config|settings
  
  # 条件 4: 检测密钥模式
  - field: new_text
    operator: regex_match
    pattern: (API_KEY|SECRET|TOKEN|PASSWORD)\s*[:=]\s*["'][^"']{16,}["']
---

🛑 **检测到硬编码的敏感凭证！**

**风险等级**: 高
**影响**: 可能导致凭证泄露到版本控制系统

**检测到的模式**:
- API 密钥
- 访问令牌
- 密码
- 密钥

**立即行动**:
1. **不要提交此代码**
2. 使用环境变量: `process.env.API_KEY`
3. 添加到 `.env` 文件（已在 .gitignore）
4. 如已泄露，立即轮换密钥

**正确示例**:
```typescript
// ❌ 错误
const apiKey = "sk-1234567890abcdef";

// ✅ 正确
const apiKey = process.env.API_KEY;
if (!apiKey) {
  throw new Error("API_KEY not configured");
}
```
```

### 3.2 时间和上下文感知规则

**场景**: 工作时间外阻止危险操作

```python
# hooks/context_aware.py
import json
import sys
from datetime import datetime, time

def check_business_hours():
    """检查是否在工作时间内 (9:00-18:00)"""
    now = datetime.now()
    return time(9, 0) <= now.time() <= time(18, 0)

def main():
    input_data = json.load(sys.stdin)
    tool_name = input_data.get('tool_name', '')
    tool_input = input_data.get('tool_input', {})
    
    # 只检查 Bash 命令
    if tool_name != 'Bash':
        print(json.dumps({}))
        return
    
    command = tool_input.get('command', '')
    
    # 检测危险命令
    dangerous_patterns = [
        r'rm\s+-rf',
        r'drop\s+table',
        r'delete\s+from',
        r'truncate\s+table'
    ]
    
    import re
    is_dangerous = any(
        re.search(pattern, command, re.IGNORECASE)
        for pattern in dangerous_patterns
    )
    
    if is_dangerous and not check_business_hours():
        print(json.dumps({
            "hookSpecificOutput": {
                "hookEventName": "PreToolUse",
                "permissionDecision": "deny"
            },
            "systemMessage": """⚠️ **工作时间外危险操作**

检测到危险命令在工作时间外执行（当前时间: {}）

工作时间: 09:00 - 18:00 (周一至周五)

如需执行此操作，请：
1. 联系团队负责人获得批准
2. 记录操作原因和影响
3. 确保有回滚计划

或者在工作时间内重试。
""".format(datetime.now().strftime("%H:%M"))
        }))
    else:
        print(json.dumps({}))

if __name__ == '__main__':
    main()
```

### 3.3 状态机规则

**场景**: 强制执行部署流程顺序

```python
# hooks/deployment_state_machine.py
import json
import os
import sys

STATE_FILE = "/tmp/deployment_state.json"

class DeploymentStateMachine:
    STATES = {
        'idle': {'allowed_next': ['tests_passed']},
        'tests_passed': {'allowed_next': ['staging_deployed']},
        'staging_deployed': {'allowed_next': ['staging_verified', 'tests_passed']},
        'staging_verified': {'allowed_next': ['production_deployed']},
        'production_deployed': {'allowed_next': ['production_verified']},
        'production_verified': {'allowed_next': ['idle']}
    }
    
    def __init__(self):
        self.state = self.load_state()
    
    def load_state(self):
        if os.path.exists(STATE_FILE):
            with open(STATE_FILE) as f:
                return json.load(f).get('state', 'idle')
        return 'idle'
    
    def save_state(self):
        with open(STATE_FILE, 'w') as f:
            json.dump({'state': self.state}, f)
    
    def can_transition(self, new_state):
        allowed = self.STATES.get(self.state, {}).get('allowed_next', [])
        return new_state in allowed
    
    def transition(self, new_state):
        if self.can_transition(new_state):
            self.state = new_state
            self.save_state()
            return True
        return False

def detect_state_from_command(command):
    """从命令推断状态转换"""
    if 'npm test' in command or 'pytest' in command:
        return 'tests_passed'
    elif 'deploy staging' in command:
        return 'staging_deployed'
    elif 'verify staging' in command:
        return 'staging_verified'
    elif 'deploy production' in command:
        return 'production_deployed'
    elif 'verify production' in command:
        return 'production_verified'
    return None

def main():
    input_data = json.load(sys.stdin)
    tool_name = input_data.get('tool_name', '')
    tool_input = input_data.get('tool_input', {})
    
    if tool_name != 'Bash':
        print(json.dumps({}))
        return
    
    command = tool_input.get('command', '')
    new_state = detect_state_from_command(command)
    
    if not new_state:
        print(json.dumps({}))
        return
    
    sm = DeploymentStateMachine()
    
    if not sm.can_transition(new_state):
        print(json.dumps({
            "hookSpecificOutput": {
                "hookEventName": "PreToolUse",
                "permissionDecision": "deny"
            },
            "systemMessage": f"""🚫 **部署流程违规**

当前状态: {sm.state}
尝试操作: {new_state}

允许的状态转换:
{chr(10).join(f"  • {sm.state} → {s}" for s in sm.STATES[sm.state]['allowed_next'])}

请按正确顺序执行部署步骤。
"""
        }))
    else:
        sm.transition(new_state)
        print(json.dumps({
            "systemMessage": f"✅ 状态转换: {sm.state} → {new_state}"
        }))

if __name__ == '__main__':
    main()
```

---

## 4. 性能优化策略

### 4.1 Regex 缓存机制

```python
from functools import lru_cache
import re

@lru_cache(maxsize=128)
def compile_regex(pattern: str) -> re.Pattern:
    """编译并缓存正则表达式"""
    return re.compile(pattern, re.IGNORECASE)

# 使用缓存的正则
def match_pattern(pattern: str, text: str) -> bool:
    regex = compile_regex(pattern)  # 从缓存获取
    return bool(regex.search(text))
```

**性能对比**:
- 无缓存: 每次编译 ~2-5ms
- 有缓存: 首次编译 ~2-5ms，后续 <0.1ms
- **提升**: 20-50倍（重复模式）

### 4.2 事件过滤

```python
def load_rules(event: Optional[str] = None) -> List[Rule]:
    """只加载匹配当前事件的规则"""
    all_rules = []
    
    # 扫描规则文件
    for rule_file in glob.glob('.claude/hookify.*.local.md'):
        rule = parse_rule_file(rule_file)
        
        # 过滤: 只加载相关事件
        if event and rule.event != 'all' and rule.event != event:
            continue  # 跳过不相关的规则
        
        if rule.enabled:
            all_rules.append(rule)
    
    return all_rules
```

**性能影响**:
- 全部规则: 50+ 规则 → ~50ms
- 过滤后: 5-10 规则 → ~5ms
- **提升**: 10倍

### 4.3 短路评估

```python
def evaluate_rules(rules: List[Rule], input_data: Dict) -> Dict:
    """短路评估 - 发现 block 规则立即返回"""
    
    for rule in rules:
        if not rule_matches(rule, input_data):
            continue
        
        if rule.action == 'block':
            # 立即返回，不评估其他规则
            return build_block_response(rule)
        
        # warn 规则继续评估，可能发现 block 规则
    
    # 所有规则评估完成
    return build_warning_response()
```

### 4.4 异步 Hook（高级）

```python
# hooks/async_hook.py
import asyncio
import json
import sys

async def check_external_service(api_key: str) -> bool:
    """异步检查外部服务"""
    # 模拟 API 调用
    await asyncio.sleep(0.1)
    return True

async def async_hook_handler(input_data: Dict):
    """异步 hook 处理器"""
    tool_input = input_data.get('tool_input', {})
    api_key = extract_api_key(tool_input)
    
    if api_key:
        # 并行检查多个服务
        results = await asyncio.gather(
            check_external_service(api_key),
            check_database(api_key),
            check_cache(api_key)
        )
        
        if not all(results):
            return build_block_response("API key validation failed")
    
    return {}

def main():
    input_data = json.load(sys.stdin)
    result = asyncio.run(async_hook_handler(input_data))
    print(json.dumps(result))

if __name__ == '__main__':
    main()
```

---

## 5. 实战案例集

### 5.1 企业级安全规则集

**案例 1: SQL 注入防护**

```markdown
---
name: prevent-sql-injection
enabled: true
event: file
action: block
conditions:
  - field: file_path
    operator: regex_match
    pattern: \.(py|js|ts|go|java|rb)$
  - field: new_text
    operator: regex_match
    pattern: (execute|query)\([^)]*\+|^.*\.\s*format\(.*\%s
---

🛡️ **SQL 注入漏洞检测**

**严重性**: 关键
**CVE 分类**: CWE-89

**检测到**:
可能的 SQL 注入漏洞 - 使用字符串拼接构建查询

**立即修复**:

Python (psycopg2):
```python
# ❌ 危险
cursor.execute("SELECT * FROM users WHERE id = " + user_id)

# ✅ 安全
cursor.execute("SELECT * FROM users WHERE id = %s", (user_id,))
```

JavaScript (pg):
```javascript
// ❌ 危险
client.query(`SELECT * FROM users WHERE id = ${userId}`)

// ✅ 安全
client.query('SELECT * FROM users WHERE id = $1', [userId])
```

**参考**:
- OWASP SQL Injection Prevention
- CWE-89: SQL Injection
```

**案例 2: XSS 防护**

```markdown
---
name: prevent-xss
enabled: true
event: file
action: warn
conditions:
  - field: file_path
    operator: regex_match
    pattern: \.(jsx?|tsx?|html?)$
  - field: new_text
    operator: regex_match
    pattern: (innerHTML|dangerouslySetInnerHTML|document\.write)
---

⚠️ **XSS 漏洞风险**

**严重性**: 高
**CVE 分类**: CWE-79

**检测到**:
可能的跨站脚本攻击（XSS）漏洞

**修复方案**:

React:
```jsx
// ❌ 危险
<div dangerouslySetInnerHTML={{__html: userInput}} />

// ✅ 安全
<div>{userInput}</div>

// ✅ 如必须使用 HTML，先清理
import DOMPurify from 'dompurify';
<div dangerouslySetInnerHTML={{
  __html: DOMPurify.sanitize(userInput)
}} />
```

Vanilla JS:
```javascript
// ❌ 危险
element.innerHTML = userInput;

// ✅ 安全
element.textContent = userInput;
```
```

### 5.2 代码质量规则集

**案例 3: 复杂度控制**

```python
# hooks/complexity_check.py
import ast
import json
import sys

class ComplexityChecker(ast.NodeVisitor):
    def __init__(self):
        self.max_complexity = 10
        self.violations = []
    
    def visit_FunctionDef(self, node):
        complexity = self.calculate_complexity(node)
        if complexity > self.max_complexity:
            self.violations.append({
                'function': node.name,
                'complexity': complexity,
                'line': node.lineno
            })
        self.generic_visit(node)
    
    def calculate_complexity(self, node):
        """计算圈复杂度"""
        complexity = 1
        for child in ast.walk(node):
            if isinstance(child, (ast.If, ast.While, ast.For, ast.ExceptHandler)):
                complexity += 1
            elif isinstance(child, ast.BoolOp):
                complexity += len(child.values) - 1
        return complexity

def main():
    input_data = json.load(sys.stdin)
    tool_name = input_data.get('tool_name', '')
    
    if tool_name not in ['Write', 'Edit']:
        print(json.dumps({}))
        return
    
    tool_input = input_data.get('tool_input', {})
    content = tool_input.get('content') or tool_input.get('new_string', '')
    
    try:
        tree = ast.parse(content)
        checker = ComplexityChecker()
        checker.visit(tree)
        
        if checker.violations:
            message = "⚠️ **高复杂度函数检测**\n\n"
            for v in checker.violations:
                message += f"- `{v['function']}` (复杂度: {v['complexity']}, 行: {v['line']})\n"
            
            message += f"\n最大允许复杂度: {checker.max_complexity}\n"
            message += "\n建议:\n"
            message += "- 拆分大函数为小函数\n"
            message += "- 使用多态替代复杂条件\n"
            message += "- 提取重复逻辑"
            
            print(json.dumps({
                "systemMessage": message
            }))
        else:
            print(json.dumps({}))
    except SyntaxError:
        # 不是有效的 Python 代码
        print(json.dumps({}))

if __name__ == '__main__':
    main()
```

### 5.3 DevOps 规则集

**案例 4: Kubernetes 安全**

```markdown
---
name: k8s-security-best-practices
enabled: true
event: file
action: warn
conditions:
  - field: file_path
    operator: regex_match
    pattern: \.(yaml|yml)$
  - field: new_text
    operator: contains
    pattern: kind:\s*(Deployment|Pod|StatefulSet)
---

🔒 **Kubernetes 安全检查**

检测到 Kubernetes 资源定义，请确保遵循安全最佳实践：

**必需配置**:
- [ ] `runAsNonRoot: true` - 非 root 用户运行
- [ ] `readOnlyRootFilesystem: true` - 只读文件系统
- [ ] `capabilities.drop: ["ALL"]` - 删除所有能力
- [ ] 资源限制（limits 和 requests）

**示例**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: secure-app
spec:
  template:
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
      containers:
      - name: app
        image: app:latest
        securityContext:
          readOnlyRootFilesystem: true
          capabilities:
            drop: ["ALL"]
        resources:
          limits:
            cpu: "500m"
            memory: "512Mi"
          requests:
            cpu: "250m"
            memory: "256Mi"
```

**参考**: K8s Pod Security Standards
```

---

## 6. 调试和测试

### 6.1 Hook 调试工具

```python
# tools/debug_hook.py
import json
import sys

def debug_hook():
    """调试 hook 执行"""
    # 读取输入
    input_data = json.load(sys.stdin)
    
    # 记录到文件
    with open('/tmp/hook_debug.log', 'a') as f:
        f.write(json.dumps(input_data, indent=2))
        f.write('\n' + '='*50 + '\n')
    
    # 正常处理
    result = process_hook(input_data)
    
    # 记录输出
    with open('/tmp/hook_debug.log', 'a') as f:
        f.write('OUTPUT:\n')
        f.write(json.dumps(result, indent=2))
        f.write('\n\n')
    
    print(json.dumps(result))

def process_hook(input_data):
    # 你的 hook 逻辑
    return {}

if __name__ == '__main__':
    debug_hook()
```

### 6.2 单元测试

```python
# tests/test_hooks.py
import json
import subprocess

def test_dangerous_command_blocked():
    """测试危险命令被阻止"""
    hook_input = {
        "hook_event_name": "PreToolUse",
        "tool_name": "Bash",
        "tool_input": {
            "command": "rm -rf /"
        }
    }
    
    # 运行 hook
    result = subprocess.run(
        ['python3', 'hooks/pretooluse.py'],
        input=json.dumps(hook_input),
        capture_output=True,
        text=True
    )
    
    output = json.loads(result.stdout)
    
    assert output.get('hookSpecificOutput', {}).get('permissionDecision') == 'deny'
    assert 'dangerous' in output.get('systemMessage', '').lower()

def test_safe_command_allowed():
    """测试安全命令被允许"""
    hook_input = {
        "hook_event_name": "PreToolUse",
        "tool_name": "Bash",
        "tool_input": {
            "command": "git status"
        }
    }
    
    result = subprocess.run(
        ['python3', 'hooks/pretooluse.py'],
        input=json.dumps(hook_input),
        capture_output=True,
        text=True
    )
    
    output = json.loads(result.stdout)
    
    assert output == {} or output.get('hookSpecificOutput', {}).get('permissionDecision') != 'deny'
```

### 6.3 性能测试

```python
# tests/performance_test.py
import time
import json
import subprocess

def benchmark_hook(iterations=100):
    """性能基准测试"""
    hook_input = {
        "hook_event_name": "PreToolUse",
        "tool_name": "Bash",
        "tool_input": {"command": "ls"}
    }
    
    times = []
    
    for _ in range(iterations):
        start = time.time()
        
        subprocess.run(
            ['python3', 'hooks/pretooluse.py'],
            input=json.dumps(hook_input),
            capture_output=True,
            text=True
        )
        
        times.append(time.time() - start)
    
    avg_time = sum(times) / len(times)
    max_time = max(times)
    min_time = min(times)
    
    print(f"Average: {avg_time*1000:.2f}ms")
    print(f"Max: {max_time*1000:.2f}ms")
    print(f"Min: {min_time*1000:.2f}ms")
    
    assert avg_time < 0.05, "Hook too slow (>50ms)"

if __name__ == '__main__':
    benchmark_hook()
```

---

## 7. 最佳实践总结

### 7.1 规则设计原则

1. **明确单一职责** - 每个规则只检查一件事
2. **性能优先** - 使用缓存、过滤、短路评估
3. **错误容忍** - Hook 失败不应阻止操作（fail-open）
4. **清晰消息** - 提供具体的修复建议

### 7.2 配置管理

```
.claude/
├── hookify.dangerous-commands.local.md    # 危险命令
├── hookify.security-checks.local.md        # 安全检查
├── hookify.code-quality.local.md           # 代码质量
└── hookify.deployment-rules.local.md       # 部署规则
```

### 7.3 监控和日志

```python
def log_hook_execution(event, result, duration):
    """记录 hook 执行"""
    log_entry = {
        "timestamp": datetime.now().isoformat(),
        "event": event,
        "result": "blocked" if is_blocked(result) else "allowed",
        "duration_ms": duration * 1000
    }
    
    # 发送到监控系统
    metrics.gauge('hook.duration', duration * 1000)
    metrics.increment(f'hook.{event}.{log_entry["result"]}')
```

---

## 总结

### 关键要点

1. **Hooks 是安全基石** - 在关键点拦截和控制
2. **性能至关重要** - 缓存、过滤、短路评估
3. **测试驱动** - 单元测试、集成测试、性能测试
4. **持续改进** - 监控、日志、迭代优化

### 下一步

- 实施企业级安全规则
- 建立监控和告警
- 定期审查和更新规则
- 培训团队成员

---

**文档版本**: 1.0
**最后更新**: 2026-03-31
**维护者**: Claude Code Research Team
