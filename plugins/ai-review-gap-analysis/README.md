# ai-review-gap-analysis

AI コードレビュー（GitHub Actions bot, Cursor Bugbot, Copilot 等）と人間レビューのギャップを分析するスキル。

## What it does

GitHub リポジトリの PR レビューデータを分析し、以下を特定する:

- **AI が承認したが人間が問題を発見した PR** のリスト
- 指摘パターンの分類（設計判断 / 命名 / テスト戦略 / セキュリティ / ドメイン知識 等）
- AI レビューの盲点マップ（得意領域 vs 苦手領域）
- サービス別・レビュアー別の統計

## Usage

```
/ai-review-gap-analysis myorg/myapp
/ai-review-gap-analysis myorg/myapp --since 2025-01-01
/ai-review-gap-analysis myorg/myapp --service customer,partner
/ai-review-gap-analysis myorg/myapp --output ./reports/gap-analysis.md
```

## Prerequisites

- `gh` CLI installed and authenticated (`gh auth status`)
- Read access to the target repository

## Install

```bash
claude plugin configure-marketplace https://raw.githubusercontent.com/sinozu/claude-skills/main/marketplace.json
claude plugin install ai-review-gap-analysis@sinozu-skills
```

Or manually copy to your skills directory:

```bash
cp -r plugins/ai-review-gap-analysis/skills/ai-review-gap-analysis ~/.claude/skills/
```
