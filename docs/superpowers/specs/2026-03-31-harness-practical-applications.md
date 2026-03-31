# Claude Code Harness 实际应用场景

**日期**: 2026-03-31
**版本**: 1.0
**作者**: Claude Code Research

## 执行摘要

本文档展示 Claude Code Harness 在真实开发场景中的应用，通过具体案例说明如何利用 Skills、Hooks 和 Settings 提升开发效率、代码质量和安全性。

---

## 1. 场景一：代码质量控制

### 1.1 阻止调试代码提交

**问题**: 开发者经常忘记移除 `console.log()`、`debugger;` 等调试代码

**解决方案**: 使用 Hook 自动检测和警告

```markdown
---
name: warn-console-log
enabled: true
event: file
pattern: console\.log\(|debugger;|print\(
action: warn
---

🔍 **调试代码检测**

检测到以下调试语句：
- `console.log()` - JavaScript/TypeScript
- `debugger;` - JavaScript 断点
- `print()` - Python

请确保在提交前移除这些调试语句。
```

**效果**:
- ✅ 减少代码审查中的调试代码问题
- ✅ 提高生产代码质量
- ✅ 自动提醒，无需人工检查

### 1.2 强制测试执行

**问题**: 开发者完成任务后忘记运行测试

**解决方案**: 在会话结束时检查是否运行过测试

```markdown
---
name: require-tests-run
enabled: true
event: stop
action: block
conditions:
  - field: transcript
    operator: not_contains
    pattern: npm test|pytest|cargo test|go test
---

⚠️ **未检测到测试执行**

在结束会话前，请运行测试以验证您的更改：
- JavaScript/TypeScript: `npm test`
- Python: `pytest`
- Rust: `cargo test`
- Go: `go test`

**注意**: 此规则会阻止会话结束，直到检测到测试命令。
```

**效果**:
- ✅ 强制 TDD 实践
- ✅ 减少未测试代码提交
- ✅ 提高代码可靠性

---

## 2. 场景二：安全防护

### 2.1 阻止危险命令

**问题**: 意外执行 `rm -rf /` 等危险命令

**解决方案**: 使用 Hook 阻止危险操作

```markdown
---
name: block-dangerous-rm
enabled: true
event: bash
pattern: rm\s+-rf\s+(/|\*|~)
action: block
---

🛑 **危险操作被阻止**

检测到可能导致数据丢失的命令：
- `rm -rf /` - 删除根目录
- `rm -rf *` - 删除当前目录所有文件
- `rm -rf ~` - 删除用户主目录

如需执行删除操作，请：
1. 明确指定目标路径
2. 使用 `rm -rf /specific/path`
3. 确认路径正确后再执行
```

**效果**:
- ✅ 防止数据丢失事故
- ✅ 保护系统关键文件
- ✅ 强制更安全的操作习惯

### 2.2 敏感文件保护

**问题**: 意外修改 `.env`、`credentials.json` 等敏感文件

**解决方案**: 检测敏感文件编辑

```markdown
---
name: warn-sensitive-files
enabled: true
event: file
action: warn
conditions:
  - field: file_path
    operator: regex_match
    pattern: \.env$|credentials|secrets|\.pem$|\.key$
  - field: new_text
    operator: contains
    pattern: KEY|SECRET|PASSWORD|TOKEN
---

🔐 **敏感文件编辑警告**

检测到对敏感文件的编辑：
- 文件类型：环境变量、凭证、密钥
- 包含内容：API 密钥、密码、令牌

请确认：
1. 此文件是否应该被修改
2. 凭证是否应该提交到版本控制
3. 是否应使用环境变量替代硬编码
```

**效果**:
- ✅ 防止凭证泄露
- ✅ 提高安全意识
- ✅ 强制安全最佳实践

### 2.3 网络访问控制

**问题**: 防止意外的网络操作

**解决方案**: 检测网络相关命令

```markdown
---
name: block-network-commands
enabled: true
event: bash
pattern: curl|wget|nc|ncat|telnet
action: warn
---

🌐 **网络命令检测**

检测到网络相关命令：
- `curl` - HTTP 客户端
- `wget` - 文件下载
- `nc/netcat` - 网络工具

请确认：
1. 目标 URL 是否受信任
2. 是否需要代理或 VPN
3. 是否会暴露敏感信息
```

---

## 3. 场景三：开发工作流优化

### 3.1 自动化代码审查

**问题**: 代码审查不一致，遗漏常见问题

**解决方案**: 使用 Skill 标准化审查流程

```markdown
---
name: requesting-code-review
description: Use when completing tasks, implementing major features
---

## 代码审查清单

### 功能正确性
- [ ] 代码实现符合需求规格
- [ ] 边界情况已处理
- [ ] 错误处理完整

### 代码质量
- [ ] 代码可读性良好
- [ ] 命名清晰有意义
- [ ] 无重复代码（DRY）

### 性能
- [ ] 无明显性能问题
- [ ] 资源正确释放
- [ ] 算法复杂度合理

### 安全性
- [ ] 输入已验证
- [ ] 无 SQL 注入风险
- [ ] 无 XSS 漏洞

### 测试
- [ ] 单元测试已编写
- [ ] 集成测试已通过
- [ ] 边界情况已测试
```

**效果**:
- ✅ 标准化审查流程
- ✅ 减少审查遗漏
- ✅ 提高代码质量

### 3.2 Git 工作流自动化

**问题**: Git 操作不规范，分支管理混乱

**解决方案**: 使用 Skill 指导 Git 工作流

```markdown
---
name: using-git-worktrees
description: Use when starting feature work that needs isolation
---

## Git Worktree 工作流

### 创建隔离工作区
1. 创建新分支: `git worktree add ../feature-xxx -b feature/xxx`
2. 切换到工作树: `cd ../feature-xxx`
3. 开始开发工作

### 完成工作
1. 运行测试: `npm test`
2. 提交更改: `git commit -m "message"`
3. 推送分支: `git push origin feature/xxx`
4. 清理工作树: `git worktree remove ../feature-xxx`

### 优势
- ✅ 并行开发多个功能
- ✅ 隔离不同工作上下文
- ✅ 避免分支切换问题
```

---

## 4. 场景四：团队协作

### 4.1 代码风格统一

**问题**: 不同开发者代码风格不一致

**解决方案**: 使用 Hook 检测风格问题

```markdown
---
name: enforce-code-style
enabled: true
event: file
action: warn
conditions:
  - field: file_path
    operator: regex_match
    pattern: \.(js|ts|jsx|tsx)$
  - field: new_text
    operator: regex_match
    pattern: var\s+\w+|function\s+\w+\s*\(
---

🎨 **代码风格建议**

检测到旧式 JavaScript 语法：
- `var` - 建议使用 `const` 或 `let`
- `function` - 考虑使用箭头函数

项目代码风格指南：
- 使用 `const/let` 替代 `var`
- 优先使用箭头函数
- 遵循 ESLint 规则
```

**效果**:
- ✅ 统一代码风格
- ✅ 减少代码审查争议
- ✅ 提高代码可维护性

### 4.2 文档完整性检查

**问题**: 代码缺少必要文档

**解决方案**: 检测公共 API 缺少注释

```markdown
---
name: require-documentation
enabled: true
event: file
action: warn
conditions:
  - field: file_path
    operator: regex_match
    pattern: \.(ts|tsx)$
  - field: new_text
    operator: regex_match
    pattern: ^\s*(export\s+)?(async\s+)?function\s+\w+
---

📝 **文档缺失提醒**

检测到公共函数可能缺少文档：
- 导出的函数应包含 JSDoc 注释
- 说明参数类型和返回值
- 提供使用示例

示例：
```typescript
/**
 * 计算两个数的和
 * @param a - 第一个数
 * @param b - 第二个数
 * @returns 两数之和
 */
export function add(a: number, b: number): number {
  return a + b;
}
```
```

---

## 5. 场景五：DevOps 和部署

### 5.1 部署前检查

**问题**: 部署遗漏关键步骤

**解决方案**: 使用 Skill 标准化部署流程

```markdown
---
name: deployment-checklist
description: Use before deploying to production
---

## 生产部署清单

### 预部署检查
- [ ] 所有测试通过
- [ ] 代码审查完成
- [ ] 环境变量已配置
- [ ] 数据库迁移脚本已准备

### 部署步骤
- [ ] 备份当前版本
- [ ] 部署新版本
- [ ] 运行健康检查
- [ ] 验证关键功能

### 部署后验证
- [ ] 监控指标正常
- [ ] 日志无错误
- [ ] 用户反馈正常
- [ ] 回滚计划就绪
```

### 5.2 Docker 操作安全

**问题**: Docker 操作可能导致资源问题

**解决方案**: 检测危险 Docker 命令

```markdown
---
name: docker-safety
enabled: true
event: bash
action: warn
pattern: docker\s+(run|exec).*--privileged
---

🐳 **Docker 安全警告**

检测到特权模式容器：
- `--privileged` 赋予容器完整系统权限
- 可能导致安全风险

建议：
- 仅在必要时使用特权模式
- 考虑使用 `--cap-add` 限制权限
- 审查容器安全策略
```

---

## 6. 场景六：API 开发

### 6.1 API 设计规范

**问题**: API 设计不一致

**解决方案**: 使用 Skill 指导 API 设计

```markdown
---
name: api-design-guidelines
description: Use when designing REST APIs
---

## RESTful API 设计指南

### URL 设计
- 使用名词复数: `/users`, `/products`
- 层级关系: `/users/{id}/orders`
- 避免动词: ❌ `/getUsers`

### HTTP 方法
- `GET` - 获取资源
- `POST` - 创建资源
- `PUT` - 完整更新
- `PATCH` - 部分更新
- `DELETE` - 删除资源

### 响应格式
```json
{
  "success": true,
  "data": {},
  "error": null,
  "timestamp": "2026-03-31T10:00:00Z"
}
```

### 状态码
- `200` - 成功
- `201` - 创建成功
- `400` - 请求错误
- `404` - 资源不存在
- `500` - 服务器错误
```

### 6.2 API 密钥保护

**问题**: API 密钥意外泄露

**解决方案**: 检测硬编码的 API 密钥

```markdown
---
name: detect-api-keys
enabled: true
event: file
action: block
conditions:
  - field: new_text
    operator: regex_match
    pattern: (API_KEY|SECRET|TOKEN)\s*[:=]\s*["'][^"']{20,}["']
---

🔑 **API 密钥泄露风险**

检测到硬编码的 API 密钥！

请立即：
1. **不要提交此代码**
2. 使用环境变量存储密钥
3. 轮换已泄露的密钥

正确做法：
```javascript
const apiKey = process.env.API_KEY;
```
```

---

## 7. 场景七：数据库操作

### 7.1 SQL 注入防护

**问题**: SQL 查询存在注入风险

**解决方案**: 检测不安全的 SQL 拼接

```markdown
---
name: prevent-sql-injection
enabled: true
event: file
action: warn
conditions:
  - field: new_text
    operator: regex_match
    pattern: (query|execute)\(.*\+\s*(req|params|body)
---

⚠️ **SQL 注入风险**

检测到可能的 SQL 注入漏洞：
- 字符串拼接构建 SQL 查询
- 直接使用用户输入

请使用参数化查询：
```javascript
// ❌ 危险
const query = `SELECT * FROM users WHERE id = ${userId}`;

// ✅ 安全
const query = 'SELECT * FROM users WHERE id = ?';
db.query(query, [userId]);
```
```

### 7.2 数据库迁移检查

**问题**: 数据库迁移缺少回滚计划

**解决方案**: 使用 Skill 指导迁移流程

```markdown
---
name: database-migration
description: Use when creating database migrations
---

## 数据库迁移最佳实践

### 迁移文件结构
```
migrations/
├── 001_create_users.up.sql
├── 001_create_users.down.sql
├── 002_add_email_index.up.sql
└── 002_add_email_index.down.sql
```

### 检查清单
- [ ] 提供 up 和 down 迁移
- [ ] 测试回滚流程
- [ ] 备份生产数据
- [ ] 在测试环境验证
- [ ] 评估迁移时间

### 危险操作
- ❌ 删除列（数据丢失）
- ❌ 修改列类型（可能失败）
- ❌ 添加 NOT NULL 约束（已有数据）
```

---

## 8. 场景八：前端开发

### 8.1 性能优化提醒

**问题**: 前端性能问题

**解决方案**: 检测常见性能反模式

```markdown
---
name: frontend-performance
enabled: true
event: file
action: warn
conditions:
  - field: file_path
    operator: regex_match
    pattern: \.(jsx|tsx)$
  - field: new_text
    operator: regex_match
    pattern: useEffect\(\s*\(\)\s*=>|map\(.*=>.*map\(
---

⚡ **前端性能提示**

检测到可能的性能问题：

### useEffect 依赖
- 确保依赖数组完整
- 避免不必要的重新渲染
- 使用 `useMemo` 和 `useCallback`

### 嵌套 map
- 避免深层嵌套循环
- 考虑数据扁平化
- 使用虚拟滚动（大列表）

优化建议：
```typescript
// ✅ 使用 useMemo
const items = useMemo(() => 
  data.map(item => <Item key={item.id} {...item} />),
  [data]
);
```
```

### 8.2 可访问性检查

**问题**: 缺少可访问性支持

**解决方案**: 检测 a11y 问题

```markdown
---
name: accessibility-check
enabled: true
event: file
action: warn
conditions:
  - field: new_text
    operator: regex_match
    pattern: <img[^>]*>|<button[^>]*>[^<]*</button>
---

♿ **可访问性提醒**

检测到需要可访问性支持的元素：

### 图片
- 必须包含 `alt` 属性
- 装饰性图片使用 `alt=""`

### 按钮
- 确保有可点击文本或 `aria-label`
- 键盘可访问

示例：
```tsx
// ✅ 正确
<img src="photo.jpg" alt="用户头像" />
<button aria-label="关闭">×</button>
```
```

---

## 9. 场景九：微服务架构

### 9.1 服务间通信

**问题**: 服务调用不规范

**解决方案**: 使用 Skill 指导微服务设计

```markdown
---
name: microservices-guidelines
description: Use when designing microservices
---

## 微服务设计原则

### 服务拆分
- 单一职责
- 独立部署
- 故障隔离

### 通信模式
- 同步: REST, gRPC
- 异步: 消息队列
- 事件驱动: Event Sourcing

### 容错机制
- 熔断器（Circuit Breaker）
- 重试策略
- 超时控制
- 降级方案

### 监控
- 分布式追踪
- 日志聚合
- 指标收集
```

### 9.2 环境变量管理

**问题**: 环境配置混乱

**解决方案**: 检测硬编码配置

```markdown
---
name: detect-hardcoded-config
enabled: true
event: file
action: warn
conditions:
  - field: file_path
    operator: regex_match
    pattern: \.(ts|js|py|go)$
  - field: new_text
    operator: regex_match
    pattern: (localhost|127\.0\.0\.1|192\.168\.|10\.)[:\d]
---

⚙️ **配置硬编码检测**

检测到硬编码的地址：
- 使用环境变量配置
- 支持12-factor app
- 区分环境配置

正确做法：
```javascript
const dbHost = process.env.DB_HOST || 'localhost';
const apiUrl = config.get('api.url');
```
```

---

## 10. 场景十：测试自动化

### 10.1 测试覆盖要求

**问题**: 测试覆盖不足

**解决方案**: 使用 Skill 制定测试策略

```markdown
---
name: testing-strategy
description: Use when implementing features
---

## 测试金字塔

### 单元测试（70%）
- 快速执行
- 隔离测试
- 覆盖边界情况

### 集成测试（20%）
- 组件交互
- 数据库操作
- API 调用

### 端到端测试（10%）
- 用户流程
- 关键路径
- 真实环境

### 测试清单
- [ ] 正常路径测试
- [ ] 边界值测试
- [ ] 异常处理测试
- [ ] 性能测试
- [ ] 安全测试
```

### 10.2 测试数据管理

**问题**: 测试数据污染

**解决方案**: 检测测试中的硬编码数据

```markdown
---
name: test-data-best-practices
enabled: true
event: file
action: warn
conditions:
  - field: file_path
    operator: regex_match
    pattern: \.(test|spec)\.(ts|js|py)$
  - field: new_text
    operator: contains
    pattern: password|secret|token
---

🧪 **测试数据安全**

检测到测试中的敏感数据：
- 不要使用真实密码
- 不要使用生产令牌
- 使用测试专用凭证

建议：
```javascript
// ✅ 使用测试数据
const testUser = {
  email: 'test@example.com',
  password: 'test-password-123'
};

// ❌ 避免真实数据
const realUser = {
  email: 'user@company.com',
  password: 'RealP@ss123'
};
```
```

---

## 总结：最佳实践

### 选择合适的工具

| 场景 | 推荐工具 | 原因 |
|------|---------|------|
| 强制执行规则 | Hook (block) | 必须遵守，无法绕过 |
| 最佳实践提醒 | Hook (warn) | 提醒但不强制 |
| 复杂工作流 | Skill | 多步骤指导 |
| 环境配置 | Settings | 灵活可配置 |
| 权限控制 | Settings.permissions | 安全管理 |

### 实施建议

1. **渐进式引入**
   - 从警告（warn）开始
   - 观察效果
   - 逐步升级为阻止（block）

2. **团队共识**
   - 讨论并同意规则
   - 文档化原因
   - 定期审查

3. **持续优化**
   - 收集反馈
   - 调整规则
   - 添加新场景

### 度量指标

- 代码审查效率提升
- Bug 数量减少
- 开发时间缩短
- 团队满意度

---

**文档版本**: 1.0
**最后更新**: 2026-03-31
**维护者**: Claude Code Research Team
