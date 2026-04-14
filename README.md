# claude-skills

Claude Code 用のスキルプラグイン集。

## Available Plugins

| Plugin | Description |
|--------|-------------|
| [ai-review-gap-analysis](./plugins/ai-review-gap-analysis/) | AI コードレビューと人間レビューのギャップ分析 |
| [impl-orchestrator](./plugins/impl-orchestrator/) | 実装計画から PR 単位で自律実装するオーケストレーター |

## Install

### Via Marketplace

```bash
claude plugin configure-marketplace https://raw.githubusercontent.com/sinozu/claude-skills/main/marketplace.json
claude plugin install ai-review-gap-analysis@sinozu-skills
claude plugin install impl-orchestrator@sinozu-skills
```

### Manual

```bash
git clone https://github.com/sinozu/claude-skills.git

# ai-review-gap-analysis
cp -r claude-skills/plugins/ai-review-gap-analysis/skills/* ~/.claude/skills/

# impl-orchestrator (skill + agents)
cp -r claude-skills/plugins/impl-orchestrator/skills/* ~/.claude/skills/
cp -r claude-skills/plugins/impl-orchestrator/agents/* ~/.claude/agents/
```
