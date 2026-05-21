# CLAUDE.md — Project AI Workflow Configuration

> This file is read automatically at the start of every Claude Code session.
> It routes planning through OpenSpec and coding discipline through Superpowers.
> Do NOT modify without team review — changes affect all AI-assisted work in this repo.

---

## 工具分工（Tool Ownership）

| 阶段        | 工具                   | 说明                            |
| --------- | -------------------- | ----------------------------- |
| 需求探索 / 规划 | OpenSpec (`/opsx:*`) | 生成 proposal、spec、design、tasks |
| 编码规范执行    | Superpowers skills   | TDD、调试、审查、验证                  |
| 实际执行      | Claude Code          | 写代码、跑测试、管理 Git                |

---

## 规划阶段（Planning Phase）

**任何新功能、新接口、新模块，统一从 `/opsx:propose` 开始。**

```
/opsx:propose <功能描述>
```

- 跳过 `brainstorming` 和 `writing-plans`（OpenSpec 的 propose 已覆盖其职责，避免产生两份互相冲突的设计文档）
- 如需先发散想法，使用 `/opsx:explore`，再进入 propose
- 生成的四个文档均视为唯一事实来源（source of truth）：
  - `proposal.md` — Why + What + Out of Scope
  - `specs/` — GIVEN/WHEN/THEN 行为规格
  - `design.md` — 技术决策及理由
  - `tasks.md` — 2-5 分钟粒度的执行清单
- 修改规格时，直接编辑对应文件，不要新建文档

---

## 编码阶段（Coding Phase）

### TDD — 强制测试先行

使用 `/opsx:apply` 时，**激活 test-driven-development， 始终遵循 TDD 流程：**

1. **RED** — 先写一个失败的测试，此时不得存在任何实现代码
2. **GREEN** — 写刚好能让测试通过的最少实现
3. **REFACTOR** — 整理代码，确保测试持续通过
4. **REVIEW** — 触发 `code-reviewer` 进行自我审查

> ⚠️ 如果在测试之前写了实现代码，删除该实现代码，重新从 RED 开始。

### Git 分支隔离

每次执行 `/opsx:apply` 前，使用 `using-git-worktrees` 在新分支上创建隔离工作区：

```bash
# Claude Code 应自动执行，等价于：
git worktree add ../worktrees/<feature-name> -b feat/<feature-name>
```

- 实验失败时丢弃 worktree，主分支不受影响
- 每个 OpenSpec change 对应一个独立的 worktree

### 调试规范

遇到 bug、测试失败、异常行为时，激活 `systematic-debugging`：

1. **重现** — 先写一个能稳定重现问题的测试
2. **定位** — 找到根本原因，不做假设
3. **修复** — 只修改与根本原因直接相关的代码
4. **验证** — 确认修复不引入新问题（回归测试）

禁止在未找到根本原因前就开始修改代码。

### 代码审查

完成每个主要实现步骤后，触发 `code-reviewer`，检查：

- 实现是否与 `specs/` 中的行为规格一致
- 是否存在安全隐患（注入、未鉴权的私有接口、明文敏感信息）
- 错误处理是否完整
- 是否存在重复代码（DRY 原则）

### 完成验证

在声称任何任务"完成"之前，通过 `verification-before-completion`：

- 所有相关测试通过（`npm test` / `pytest` / 项目对应命令）
- 实现与 `tasks.md` 中的验收标准逐条对应
- 没有遗留的 `TODO` / `FIXME` / 调试日志

---

## 收尾阶段（Wrap-up Phase）

每次功能完成后，**最后一个操作必须是**：

```
/opsx:archive
```

- 将 Delta Spec 合并回主规格库
- 将本次变更归档至 `openspec/changes/<id>/archive/`
- 跳过此步将导致下次会话读取过期规格，重复实现已有功能

---

## 场景决策（When to Skip Full Pipeline）

| 场景                 | 策略                                                   |
| ------------------ | ---------------------------------------------------- |
| 快速原型 / 探索性代码（< 2h） | 仅 Claude Code，无需 OpenSpec 流程                         |
| Bug 修复（已有规格）       | 直接 `systematic-debugging` → TDD 修复 → `code-reviewer` |
| 文档 / 注释更新          | 直接编辑，无需 propose/apply                                |
| 依赖升级               | 直接执行，跑测试验证，无需规格                                      |
| 新功能 / 新接口 / 重构     | 完整流程：propose → apply → archive                       |

---

## 禁止行为（Hard Rules）

- ❌ 不在 `main` / `master` 分支上直接写功能代码
- ❌ 不在测试之前写实现代码（TDD 强制）
- ❌ 不跳过 `/opsx:archive`（功能完成后必须执行）
- ❌ 不同时维护 `docs/superpowers/specs/` 和 `openspec/changes/` 两套设计文档（统一走 OpenSpec）
- ❌ 不在未找到根本原因前修改代码（调试规范）
- ❌ 不声称完成但跳过验证步骤

---

## 快速参考（Cheat Sheet）

```
新功能完整流程：
  /opsx:propose <描述>   # 生成四文档
  审查 tasks.md          # 5 分钟，确认任务顺序和验收标准
  /opsx:apply            # TDD 执行（需 CLAUDE.md 配置生效）
  /opsx:archive          # 必须，归档变更

Bug 修复流程：
  写重现测试 (RED)
  systematic-debugging 定位根因
  最小修复 (GREEN)
  code-reviewer 审查
  verification-before-completion

探索流程：
  /opsx:explore          # 头脑风暴
  /opsx:propose          # 固化为规格
```
