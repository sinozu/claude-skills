# impl-orchestrator

実装計画を読み込み、PR 単位で自律的に実装を進めるオーケストレーター。

## Overview

[cc-sdd](https://github.com/gotalab/cc-sdd) の `/kiro-impl` パターンをベースに、**Implementer / Reviewer / Debugger** の 3 エージェントを fresh context で分離し、有限状態機械で制御する。

### Why sub-agent separation?

多くの自律実装ツールは同一コンテキストで自己批評させるが、このオーケストレーターは各ロールを**プロセス境界で分離**している:

- **Reviewer** が Implementer の「こういう意図で省略した」という弁明に引きずられない
- **Debugger** が「前の試行はこうだった」というバイアスから自由になり根本原因分析ができる
- 各ロールのプロンプトを独立に最適化できる

## Architecture

```
実装計画読み込み → PR選択 → ①Implementer → ②Reviewer
                                ↑                |
                                |        APPROVED → コミット → 次PR
                                |        REJECTED(1-2回) → feedback付きで①へ
                                |        REJECTED(3回目) ↓
                                └── ③Debugger ← BLOCKED
                                     ├─ RETRY → ①へ（FIX_PLAN付き）
                                     ├─ BLOCK → 記録 → 次PR
                                     └─ STOP  → ユーザーに報告
```

### Sub-agents

| Role | Agent | Responsibility |
|------|-------|---------------|
| **Implementer** | `impl-worker` | PR checklist implementation with tests |
| **Reviewer** | `impl-reviewer` | Mechanical + judgment checks on git diff |
| **Debugger** | `impl-debugger` | Root cause analysis on failures |

### Retry Limits

| Type | Limit | On Exceed |
|------|-------|-----------|
| Reviewer REJECTED → re-implement | 2 | Debugger spawn |
| Debugger RETRY → re-implement | 2 rounds/PR | Block and skip |
| NEEDS_CONTEXT → re-spawn | 1 | Treat as BLOCKED |

## Usage

```
/impl-orchestrator --plan path/to/implementation-plan.md
/impl-orchestrator --plan path/to/implementation-plan.md --pr 3
/impl-orchestrator --plan path/to/implementation-plan.md --mode auto
```

### Modes

- **`step`** (default): Pause for user confirmation after each PR
- **`auto`**: Execute all PRs autonomously, stop only on BLOCK or STOP_FOR_HUMAN

## Install

### Manual

```bash
git clone https://github.com/sinozu/claude-skills.git
cp -r claude-skills/plugins/impl-orchestrator/skills/* ~/.claude/skills/
cp -r claude-skills/plugins/impl-orchestrator/agents/* ~/.claude/agents/
```

## Files

```
plugins/impl-orchestrator/
├── .claude-plugin/
│   └── plugin.json
├── README.md
├── skills/
│   └── impl-orchestrator/
│       └── SKILL.md          # Main orchestrator (state machine)
└── agents/
    ├── impl-worker.md        # Implementer agent
    ├── impl-reviewer.md      # Reviewer agent
    └── impl-debugger.md      # Debugger agent
```

## Credits

Inspired by [cc-sdd](https://github.com/gotalab/cc-sdd) v3's `/kiro-impl` autonomous implementation pattern.
