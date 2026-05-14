# Deep Interview Spec: Claude Code 配置切换工具

## Metadata
- Interview ID: 550e8400-e29b-41d4-a716-446655440000
- Rounds: 11
- Final Ambiguity Score: 19.0%
- Type: greenfield
- Generated: 2026-05-12
- Threshold: 0.2
- Status: PASSED

## Clarity Breakdown
| Dimension | Score | Weight | Weighted |
|-----------|-------|--------|----------|
| Goal Clarity | 0.90 | 0.40 | 0.36 |
| Constraint Clarity | 0.80 | 0.30 | 0.24 |
| Success Criteria | 0.70 | 0.30 | 0.21 |
| **Total Clarity** | | | **0.81** |
| **Ambiguity** | | | **0.19** |

## Goal
构建一款用 Rust 编写的跨平台 CLI/TUI 工具，让用户可以轻松管理多个 Claude Code 配置预设（Profile），一键切换不同的 API 提供商（base_url + api_key）、模型或完整配置集，无需手动编辑 `~/.claude/settings.json`。

## Constraints
- **技术栈**: Rust，目标跨平台（macOS/Linux/Windows）
- **Profile 存储**: 独立 TOML 配置文件，遵循 XDG 规范存放
- **核心场景**: 切换 API 提供商（不同平台的 base_url + api_key）是最高优先级
- **工具形态**: CLI + TUI + Claude Code Hook 集成（多形态结合）
- **即时生效**: 理想情况下修改后即时生效，但接受渐进方案
- **务实优先**: 优先实现最可行的方案，其他作为增强

## Non-Goals
- 不修改 Claude Code 本身的源代码
- 不创建通用的 dotfiles 管理工具（聚焦 Claude Code 配置）
- 不替代 Claude Code 内置的配置管理
- 不强制要求即时生效（接受重启方案作为 fallback）

## Acceptance Criteria
- [ ] 用户可以定义多个 Profile，每个 Profile 是一个独立的 TOML 文件
- [ ] 工具可以扫描并列出所有可用的 Profile
- [ ] 用户可以通过 CLI 命令一键切换当前激活的 Profile
- [ ] 切换前自动备份当前 `settings.json`
- [ ] 切换后支持一键回滚到上一个配置
- [ ] 切换前验证目标 Profile 的配置合法性（如 API key 格式、base_url 有效性）
- [ ] 提供 TUI 交互式菜单选择 Profile
- [ ] 提供 Claude Code Hook 集成，支持在对话中触发切换
- [ ] 跨平台支持（macOS/Linux/Windows）
- [ ] Profile 文件遵循 XDG 规范存放

## Assumptions Exposed & Resolved
| Assumption | Challenge | Resolution |
|-----------|-----------|------------|
| 配置修改后可以即时生效 | Round 4 Contrarian 模式挑战 | 接受渐进方案：优先修改文件，即时生效作为增强 |
| 需要一次性实现全部功能 | Round 5 提问 | 用户表态"全部功能一次性实现" |
| 核心场景是"切换接入" | Round 6 Simplifier 模式简化 | 核心场景收敛为"切换 API 提供商" |
| 工具需要从零写一个完整程序 | Round 8 Ontologist 模式质疑 | 确认工具需提供配置验证、备份回滚、配置发现等 `cp` 做不到的价值 |
| Profile 应嵌入 settings.json | Round 7 提问 | 确定为独立配置文件 |
| 工具名称可用 | Round 10 用户反馈 | "cc-switch"已被占用，需要新名称 |

## Technical Context
- **Greenfield 项目**: 当前工作目录无相关现有代码
- **Claude Code 配置机制**: 配置存储在 `~/.claude/settings.json` 中，包含 env 变量（如 `ANTHROPIC_BASE_URL`、`ANTHROPIC_AUTH_TOKEN`）、model 设置、hooks 等
- **Rust 生态选择**:
  - CLI: `clap` (派生宏或 builder 模式)
  - TUI: `ratatui`
  - TOML 解析: `toml` + `serde`
  - 文件操作: 标准库 + `dirs` (XDG 规范)
  - HTTP 验证: `reqwest` (验证 base_url 可访问性)

## Ontology (Key Entities)

| Entity | Type | Fields | Relationships |
|--------|------|--------|---------------|
| ConfigSwitcher | core domain | profiles_dir, active_profile, backup_dir | manages Profile, switches between APIProviders |
| Profile | core domain | name, description, env, model, hooks, effortLevel | contains APIProvider config, stored as TOML file |
| APIProvider | supporting | base_url, api_key, name | used by Profile, validated before switch |
| CLI | supporting | commands (list, switch, add, remove, backup, rollback), flags | interface to ConfigSwitcher |
| TUI | supporting | menu_items, selection, preview | interactive interface to ConfigSwitcher |

## Ontology Convergence

| Round | Entity Count | New | Changed | Stable | Stability Ratio |
|-------|-------------|-----|---------|--------|----------------|
| 1 | 3 | 3 | - | - | N/A |
| 2 | 5 | 2 | 0 | 3 | 100% |
| 3 | 5 | 0 | 0 | 5 | 100% |
| 4 | 5 | 0 | 0 | 5 | 100% |
| 5 | 5 | 0 | 0 | 5 | 100% |
| 6 | 5 | 0 | 0 | 5 | 100% |
| 7 | 5 | 0 | 0 | 5 | 100% |
| 8 | 5 | 0 | 0 | 5 | 100% |
| 9 | 5 | 0 | 0 | 5 | 100% |
| 10 | 5 | 0 | 0 | 5 | 100% |
| 11 | 5 | 0 | 0 | 5 | 100% |

**结论**: 实体从 Round 2 起完全收敛，所有 5 个实体在 10 轮中保持稳定，领域模型已完全确定。

## Interview Transcript

<details>
<summary>Full Q&A (11 rounds)</summary>

### Round 1
**Targeting:** Goal Clarity | **Ambiguity:** 100%

**Q:** 当你说「切换接入 claude code」，具体是指切换什么？
**A:** 以上多种都需要支持（API 提供商、模型、完整 Profile）

### Round 2
**Targeting:** Constraint Clarity | **Ambiguity:** 70.5%

**Q:** 你希望这个配置切换工具以什么形式存在？你通常在什么场景下使用它？
**A:** 以上多种结合（CLI + Hook + TUI）

### Round 3
**Targeting:** Success Criteria | **Ambiguity:** 61.5%

**Q:** 当你切换配置后，你期望怎么确认切换已经成功生效？
**A:** 修改后应该即时生效（不需要重启），工具需要触发 Claude Code 配置重载

### Round 4
**Targeting:** Constraint Clarity | Contrarian Mode | **Ambiguity:** 47.0%

**Q:** 如果 Claude Code 不支持运行时配置热重载，你能接受以下哪种方案？
**A:** 以上都可以接受，优先实现最可行的，其他作为增强

### Round 5
**Targeting:** Success Criteria | **Ambiguity:** 48.0%

**Q:** 你觉得第一个可用的最小版本（MVP）应该做到什么程度？
**A:** 全部功能一次性实现（CLI + Hook + TUI + 所有切换类型）

### Round 6
**Targeting:** Constraint Clarity | Simplifier Mode | **Ambiguity:** 41.5%

**Q:** 如果简化到只做一件事，你最希望先解决哪个痛点？
**A:** 切换 API 提供商（不同平台的 base_url + api_key）

### Round 7
**Targeting:** Constraint Clarity | **Ambiguity:** 35.0%

**Q:** 你希望 Profile（配置预设）以什么方式存储和管理？
**A:** 独立配置文件

### Round 8
**Targeting:** Constraint Clarity | Ontologist Mode | **Ambiguity:** 30.0%

**Q:** 这个工具和用 `cp` 切换的本质区别是什么？
**A:** 以上都是核心需求（配置验证、备份回滚、配置发现）

### Round 9
**Targeting:** Constraint Clarity | **Ambiguity:** 24.3%

**Q:** 你希望用什么技术栈来实现这个工具？目标平台是什么？
**A:** Rust

### Round 10
**Targeting:** Success Criteria | **Ambiguity:** 24.3%

**Q:** 目标平台是什么？Profile 配置文件用什么格式？（用户未直接回答，转而评论名称）
**A:** 不叫 cc-switch，这个已经有人叫了

### Round 11
**Targeting:** Constraints | **Ambiguity:** 24.3%

**Q:** 你有为这个工具想好名称吗？
**A:** 还没想好，需要建议

### Round 11 (续)
**Targeting:** Constraints | **Ambiguity:** 19.0%

**Q:** 回到关键的技术决策：目标平台是什么？Profile 配置文件用什么格式？
**A:** 跨平台 + TOML

</details>
