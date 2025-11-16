# Claude Code: Best practices for agentic coding

## 概述

本文概述了有效使用Claude Code的实用模式，Claude Code是Anthropic的命令行代理编码工具。指导强调"Claude Code故意设计为低级别和不持有观点"，要求用户开发自己的工作流而不是遵循规定的流程。

## 核心概念

### 设计理念

**低级别和不持有观点：**
- 提供基础能力
- 不强制特定工作流
- 用户自定义流程

**灵活性优先：**
- 适应不同项目需求
- 支持多种开发风格
- 可定制性强

## 环境定制

### CLAUDE.md文件系统

**核心机制：**
`CLAUDE.md`文件自动加载上下文到Claude Code中

#### 文件位置层次

**1. 仓库根目录**
- 检入版本控制
- 团队共享配置
- 项目级别指导

```markdown
# CLAUDE.md (仓库根目录)

## 项目概述
这是一个React应用...

## 代码风格
- 使用TypeScript严格模式
- 遵循Airbnb风格指南
- 使用函数组件和Hooks

## 测试
运行测试：`npm test`
覆盖率：`npm run coverage`

## 构建
生产构建：`npm run build`
```

**2. 父目录**
- 对monorepo有用
- 共享跨项目配置
- 组织级别标准

```markdown
# CLAUDE.md (monorepo根目录)

## Monorepo结构
- /packages/ui - UI组件库
- /packages/api - API服务
- /packages/shared - 共享工具

## 通用命令
- `pnpm install` - 安装依赖
- `pnpm build` - 构建所有包
```

**3. 子目录**
- 按需加载
- 特定模块指导
- 细粒度控制

```markdown
# packages/ui/CLAUDE.md

## UI组件开发
- 使用Storybook开发：`pnpm storybook`
- 组件测试：`pnpm test:components`
- 可访问性检查：`pnpm a11y`
```

**4. 主目录**
- `~/.claude/CLAUDE.md`
- 个人偏好
- 跨项目配置

```markdown
# ~/.claude/CLAUDE.md

## 个人偏好
- 提交消息使用约定式提交
- 代码注释使用中文
- 优先使用函数式编程风格
```

### 文档内容建议

**Bash命令：**
```markdown
## 常用命令
- 启动开发服务器：`npm run dev`
- 运行测试：`npm test -- --watch`
- 代码检查：`npm run lint -- --fix`
```

**代码风格指南：**
```markdown
## 代码风格
- 文件命名：kebab-case
- 组件命名：PascalCase
- 最大行长度：80字符
- 使用单引号
```

**测试流程：**
```markdown
## 测试策略
1. 单元测试：Jest
2. 集成测试：Testing Library
3. E2E测试：Playwright
4. 最小覆盖率：80%
```

**项目特定行为：**
```markdown
## 特殊注意事项
- 数据库迁移前必须备份
- API更改需要版本号递增
- 部署前运行完整测试套件
```

## 权限管理

### 权限请求机制

**默认行为：**
Claude Code对系统修改操作请求批准

**修改方式：**

#### 1. 内联"始终允许"
- 在会话中选择
- 临时设置
- 不持久化

#### 2. /permissions命令
```bash
# 查看当前权限
/permissions

# 添加工具到允许列表
/permissions add bash

# 从允许列表移除
/permissions remove write
```

#### 3. 手动编辑配置文件
```json
// .claude/settings.json
{
  "allowedTools": ["bash", "read", "write"],
  "blockedTools": ["delete"]
}
```

#### 4. CLI标志
```bash
claude --allowedTools=bash,read,write
```

### 安全考虑

**谨慎使用"dangerously-skip-permissions"：**
- 不受限制的命令执行风险
- 仅在受信任环境中使用
- 理解潜在后果

## 工具集成

### Bash环境

**访问能力：**
- 自定义脚本
- 系统工具
- 开发工具链

**示例：**
```markdown
## 自定义脚本
- 数据库重置：`./scripts/reset-db.sh`
- 测试数据生成：`./scripts/seed-data.sh`
```

### MCP（模型上下文协议）

**功能：**
- 扩展服务器连接
- 自动处理认证
- 标准化集成

**常见集成：**
- GitHub
- Slack
- Asana
- 数据库

### 自定义Slash命令

**位置：**
`.claude/commands/`

**创建示例：**
```bash
# .claude/commands/test-all.md
运行完整测试套件并生成覆盖率报告
```

**使用：**
```bash
/test-all
```

## 工作流模式

### 探索-规划-编码-提交工作流

这是一个结构化的开发方法，包含四个顺序阶段：

#### 1. 探索阶段（Exploration）

**目标：**
理解代码库而不进行修改

**活动：**
- 阅读相关文件
- 理解架构
- 识别模式
- 找到关键组件

**示例对话：**
```
用户：我想添加用户认证功能
Claude：让我先探索现有的认证相关代码...
[读取 auth/, middleware/, routes/auth.js]
发现你已经有基础的认证中间件在 middleware/auth.js...
```

#### 2. 规划阶段（Planning）

**目标：**
创建详细的实施计划

**使用扩展思考模式：**
- "think"：基础思考
- "think hard"：深度思考
- 其他变体

**计划内容：**
- 需要修改的文件
- 实施步骤
- 潜在风险
- 测试策略

**示例：**
```
用户：请规划认证功能的实施
Claude：[使用 "think hard" 模式]

计划：
1. 修改 User 模型添加密码字段
2. 创建 AuthController 处理登录/注册
3. 实现 JWT token 生成和验证
4. 添加认证中间件到受保护路由
5. 编写单元测试和集成测试
6. 更新API文档
```

#### 3. 编码阶段（Coding）

**目标：**
执行计划并实施解决方案

**最佳实践：**
- 遵循既定计划
- 进行验证步骤
- 测试每个更改
- 处理错误

**迭代执行：**
```
Claude：开始实施步骤1...
[修改 models/User.js]
运行测试验证...
测试通过，继续步骤2...
```

#### 4. 提交阶段（Committing）

**目标：**
将经过验证的更改提交到版本控制

**活动：**
- 创建有意义的提交消息
- 分组相关更改
- 遵循提交约定
- 记录更改文档

**示例：**
```
feat(auth): implement user authentication

- Add password field to User model
- Create AuthController with login/register
- Implement JWT token generation
- Add auth middleware
- Add comprehensive tests

Closes #123
```

### 测试驱动开发（TDD）模式

**特别有效于：**
可验证的更改

#### TDD流程

**1. 编写测试（先不写实现）**
```javascript
describe('UserAuthentication', () => {
  it('should hash password on user creation', () => {
    // 测试代码
  });

  it('should validate correct password', () => {
    // 测试代码
  });

  it('should reject incorrect password', () => {
    // 测试代码
  });
});
```

**2. 验证测试失败**
```bash
npm test
# 所有测试应该失败（红色）
```

**3. 实施代码使测试通过**
```javascript
class User {
  async setPassword(password) {
    this.passwordHash = await bcrypt.hash(password, 10);
  }

  async validatePassword(password) {
    return await bcrypt.compare(password, this.passwordHash);
  }
}
```

**4. 验证测试通过**
```bash
npm test
# 所有测试应该通过（绿色）
```

**5. 提交经过验证的结果**
```bash
git add .
git commit -m "feat: add password hashing to User model"
```

### 视觉迭代循环

**适用于：**
UI/设计工作

#### 工作流程

**1. 截图获取**
- Puppeteer（web）
- iOS模拟器MCP（移动）
- 其他截图工具

**2. 对比设计稿**
```
用户：这是设计稿 [上传图片]
Claude：让我截图当前实现并对比...
[获取当前UI截图]
发现以下差异：
- 按钮颜色不匹配
- 间距需要调整
- 字体大小偏小
```

**3. 迭代改进**
```
Claude：调整样式...
[修改 CSS]
让我再次截图验证...
[新截图]
现在更接近设计稿了
```

**4. 多轮精细化**
重复直到达到设计要求

### Headless模式自动化

**使用-p标志：**
```bash
claude -p "运行linter并修复所有问题"
```

**适用场景：**

#### CI/CD管道
```yaml
# .github/workflows/ci.yml
- name: Fix linting issues
  run: claude -p "运行 eslint --fix 并提交更改"
```

#### Issue分类
```bash
claude -p "分析issue #123并添加适当标签"
```

#### 自动化linting
```bash
# 定期任务
claude -p "修复所有TypeScript类型错误"
```

#### 提交消息生成
```bash
# Git hook
claude -p "为暂存的更改生成约定式提交消息"
```

## 主要优势

### 1. 减少入职时间

**通过回答代码库问题：**
```
新成员：认证是如何工作的？
Claude：[读取相关文件]
认证系统使用JWT token，实现在...
主要流程是...
```

### 2. 自动化常规任务

**示例：**
- lint修复
- 提交消息生成
- 测试更新
- 文档同步

### 3. 并行工作流

**使用git worktree：**
```bash
# 主工作区：功能开发
cd main-worktree
claude

# 次工作区：bug修复
cd bugfix-worktree
claude
```

### 4. 迭代改进

**通过清晰反馈机制：**
- 测试结果
- 截图对比
- linter输出
- 构建日志

### 5. 上下文保存

**通过.claude/commands/组织：**
- 可重用提示模板
- 项目特定命令
- 团队共享工作流

## 挑战讨论

### 1. 学习曲线

**对于新手：**
- 需要理解代理编码概念
- 学习最佳实践
- 建立有效工作流

**缓解：**
- 从简单任务开始
- 逐步引入复杂功能
- 参考最佳实践

### 2. Token消耗

**自动上下文收集：**
- 可能消耗大量token
- 需要注意成本
- 优化CLAUDE.md内容

**策略：**
- 精简上下文文件
- 使用/clear重置
- 定期清理会话

### 3. 不受限命令执行风险

**"dangerously-skip-permissions"：**
- 潜在安全风险
- 需要完全信任环境
- 理解后果

**建议：**
- 仅在必要时使用
- 在沙箱环境测试
- 审查生成的命令

### 4. CLAUDE.md文件迭代需求

**持续优化：**
- 根据实际使用调整
- 添加遗漏的指导
- 删除过时内容

**最佳实践：**
- 定期审查
- 收集团队反馈
- 版本控制跟踪

### 5. 扩展会话中上下文填充

**问题：**
长时间会话可能填满上下文窗口

**解决方案：**
- 使用/clear在任务之间重置
- 创建新会话处理不同任务
- 优化CLAUDE.md减少初始加载

## 重要建议

### 1. 在指令中具体化

**效果：**
详细提示显著提高首次尝试成功率

**对比：**
```
❌ 模糊：添加认证
✓ 具体：实现基于JWT的认证，包括：
  - User模型的密码哈希（bcrypt）
  - 登录/注册端点
  - 认证中间件
  - 刷新token机制
  - 单元测试和集成测试
```

### 2. 使用视觉辅助

**类型：**
- 截图
- 图表
- 设计稿

**增强：**
迭代质量显著提升

**示例：**
```
用户：按照这个设计稿实现登录页面
[附加设计稿截图]

Claude：我会实现这个设计，注意到：
- 使用渐变背景
- 圆角输入框
- 品牌色按钮
- 居中布局
```

### 3. 早期纠正方向

**利用：**
- 规划步骤
- Escape键中途指导

**好处：**
- 避免错误方向
- 节省时间和token
- 更好的结果

**示例：**
```
Claude：我计划修改数据库架构...
用户：[按Escape] 等等，我们应该先创建迁移脚本
Claude：好的，我会先创建迁移脚本...
```

### 4. 使用/clear

**时机：**
在任务之间重置上下文

**目的：**
- 保持性能
- 清除不相关上下文
- 开始干净的会话

**用法：**
```bash
# 完成功能A
/clear
# 开始功能B
```

### 5. 使用子代理

**部署场景：**
- 并行Claude实例
- 独立验证
- 调查研究

**示例：**
```
主会话：实现功能
子会话1：运行测试验证
子会话2：研究最佳实践
子会话3：审查代码质量
```

### 6. 增量记录

**使用#键：**
在开发周期中更新CLAUDE.md

**流程：**
```
# 发现有用的命令
按 # 键

Claude：我应该在CLAUDE.md中记录什么？
用户：添加数据库重置命令
Claude：[更新CLAUDE.md]

## 数据库管理
- 重置数据库：`npm run db:reset`
```

## 模式是起点

**重要理解：**
文章强调这些模式代表起点而非通用解决方案

**鼓励：**
实验发现团队特定的最佳工作流

**迭代：**
- 尝试不同方法
- 衡量效果
- 调整优化
- 分享学习

## 实用建议

### 项目设置

**初始化检查清单：**
- [ ] 创建仓库根CLAUDE.md
- [ ] 记录常用命令
- [ ] 添加代码风格指南
- [ ] 包含测试说明
- [ ] 配置权限设置
- [ ] 设置自定义命令

### 团队协作

**共享配置：**
- 将CLAUDE.md检入版本控制
- 文档化团队工作流
- 共享自定义命令
- 定期同步最佳实践

### 个人优化

**~/.claude/CLAUDE.md：**
```markdown
## 我的偏好
- 提交消息：约定式提交
- 代码风格：函数式优先
- 注释语言：中文
- 测试框架：Jest
```

### 性能优化

**减少token使用：**
1. 精简CLAUDE.md内容
2. 使用/clear频繁重置
3. 避免不必要的上下文
4. 优化文件组织

### 安全实践

**权限管理：**
1. 最小权限原则
2. 定期审查允许的工具
3. 谨慎使用危险标志
4. 在沙箱测试新命令

## 关键要点

1. **灵活性是核心**：无固定工作流，自定义适合你的
2. **CLAUDE.md至关重要**：有效的上下文加载机制
3. **具体化很重要**：详细指令提高成功率
4. **视觉辅助有效**：截图和设计稿增强迭代
5. **早期纠正方向**：利用规划和Escape键
6. **定期清理**：使用/clear保持性能
7. **子代理有用**：并行实例提供独立视角
8. **增量文档**：使用#键持续更新

## 工作流示例

### 完整功能开发流程

```
1. 探索
   "显示当前的认证系统实现"
   [Claude读取相关文件]

2. 规划
   "使用think hard模式规划OAuth集成"
   [Claude生成详细计划]

3. 确认
   用户审查计划，提供反馈

4. 编码
   "按计划实施，每步后运行测试"
   [Claude迭代实施]

5. 视觉验证（如果适用）
   "截图登录流程"
   [Claude对比设计稿]

6. 测试
   "运行完整测试套件"
   [验证所有测试通过]

7. 提交
   "创建提交，使用约定式提交消息"
   [Claude生成并执行提交]

8. 清理
   /clear
```

### Bug修复流程

```
1. 重现
   "重现issue #456中描述的bug"

2. 诊断
   "分析根本原因"

3. 规划修复
   "规划最小化修复方案"

4. 实施
   "实施修复并添加回归测试"

5. 验证
   "运行测试确保bug已修复且无退化"

6. 提交
   "创建提交：fix(auth): resolve token expiration issue"
```

## 结论

Claude Code的最佳实践强调灵活性、定制化和迭代改进。通过有效使用CLAUDE.md文件、理解不同工作流模式、利用视觉反馈和自动化能力，开发者可以显著提高生产力。

关键是将这些模式视为起点，根据自己的项目需求、团队风格和个人偏好进行实验和调整。随着经验积累，你将发现最适合自己的工作流，并能够充分利用Claude Code的代理编码能力。

---

*原文链接：https://www.anthropic.com/engineering/claude-code-best-practices*
