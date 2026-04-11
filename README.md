# claude-skills

Claude Code 用のスキルプラグイン集。

## Available Plugins

| Plugin | Description |
|--------|-------------|
| [ai-review-gap-analysis](./plugins/ai-review-gap-analysis/) | AI コードレビューと人間レビューのギャップ分析 |

## Install

### Via Marketplace

```bash
claude plugin configure-marketplace https://raw.githubusercontent.com/sinozu/claude-skills/main/marketplace.json
claude plugin install ai-review-gap-analysis@sinozu-skills
```

### Manual

```bash
git clone https://github.com/sinozu/claude-skills.git
cp -r claude-skills/plugins/ai-review-gap-analysis/skills/* ~/.claude/skills/
```
