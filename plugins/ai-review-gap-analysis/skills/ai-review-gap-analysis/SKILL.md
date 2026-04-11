---
name: ai-review-gap-analysis
description: Analyze gaps between AI code review approvals and human reviewer feedback on GitHub PRs. Use when asked to audit AI review quality, find blind spots in automated reviews, or compare AI vs human review effectiveness. Requires gh CLI authenticated.
allowed-tools: Bash, Read, Write, Edit, Grep, Glob, Agent
arguments:
  - name: repo
    type: string
    description: "GitHub repository in owner/repo format (e.g. myorg/myapp)"
    required: true
  - name: since
    type: string
    description: "Start date in YYYY-MM-DD format (default: 90 days ago)"
    required: false
  - name: service
    type: string
    description: "Filter by service/directory name (e.g. customer, api). Comma-separated for multiple."
    required: false
  - name: output
    type: string
    description: "Output file path for the report (default: ./ai-review-gap-analysis.md)"
    required: false
---

# AI Review Gap Analysis

AI コードレビューの盲点を特定する。GitHub PR のレビューデータを分析し、「AI が承認したが人間が問題を発見した」ケースを体系的に抽出・分類する。

## Prerequisites

- `gh` CLI が認証済みであること（`gh auth status` で確認）
- 対象リポジトリへの read 権限があること

## Parameters

- `repo`: GitHub リポジトリ（owner/repo 形式）**必須**
- `since`: 分析開始日（YYYY-MM-DD）。省略時は90日前
- `service`: サービス/ディレクトリ名でフィルタ（カンマ区切りで複数指定可）
- `output`: レポート出力先パス。省略時は `./ai-review-gap-analysis.md`

## Execution Process

### Phase 1: Bot/AI レビュアーの特定

対象リポジトリの最近のPRからレビュアー一覧を取得し、Bot/AI を分類する。

```bash
# レビュアー一覧を取得（直近50件のマージ済みPR）
gh pr list -R $REPO --state merged --limit 50 --json number \
  --jq '.[].number' | while read pr; do
  gh api repos/$REPO/pulls/$pr/reviews \
    --jq '.[] | .user.login + " " + .user.type' 2>/dev/null
done | sort | uniq -c | sort -rn
```

以下のパターンで Bot/AI を自動判別する:

| パターン | 判定 |
|---|---|
| `user.type == "Bot"` | Bot |
| login に `[bot]` を含む | Bot |
| login に `ci`, `actions`, `copilot` を含む | Bot (要確認) |
| 常に APPROVED のみ出す（COMMENTED/CHANGES_REQUESTED なし） | 自動承認 Bot の可能性 |

ユーザーに Bot/AI 一覧を提示し、確認を得る。

### Phase 2: PR データ収集

サービスフィルタがある場合はタイトル・ブランチ名・ラベルで絞り込む。

```bash
# サービスフィルタありの場合
gh pr list -R $REPO --search "$SERVICE in:title created:>$SINCE" \
  --state all --limit 200 \
  --json number,title,author,state,createdAt,headRefName,mergedAt

# ラベルベースの検索も併用
gh pr list -R $REPO --label "service/$SERVICE" \
  --search "created:>$SINCE" --state all --limit 200 \
  --json number,title,author,state,createdAt,headRefName,mergedAt
```

サービスフィルタがない場合は全PRを対象にする。

### Phase 3: レビューデータ取得と分類

各PRについて以下を取得:

```bash
# レビュー（APPROVED / CHANGES_REQUESTED / COMMENTED / DISMISSED）
gh api repos/$REPO/pulls/$PR_NUMBER/reviews \
  --jq '.[] | "\(.user.login)\t\(.state)\t\(.user.type)"'

# レビューコメント（インラインコメント）
gh api repos/$REPO/pulls/$PR_NUMBER/comments \
  --jq '.[] | "\(.user.login)\t\(.path):\(.line)\t\(.body | gsub("\n"; " ") | .[0:200])"'
```

各PRを以下のカテゴリに分類:

| カテゴリ | 条件 | 意味 |
|---|---|---|
| **AI承認+人間指摘** | Bot が APPROVED かつ 人間が CHANGES_REQUESTED または実質的 COMMENT 2件以上 | **ギャップ（分析対象）** |
| **Bot Only承認** | Bot が APPROVED、人間レビューなし | 人間レビュー未実施 |
| **人間承認のみ** | 人間が APPROVED、Bot 関与なし | 通常のレビュー |
| **AI+人間承認** | 両方が APPROVED | 合意 |

**人間コメントの実質判定**:
- 除外: 「LGTM」「👍」のみ、Bot のコメント、PR 作者自身のセルフレビュー応答
- 含む: CHANGES_REQUESTED、code suggestion、2件以上の技術的コメント

### Phase 4: ギャップPRの詳細分析

「AI承認+人間指摘」に分類されたPRについて、人間のコメント全文を取得:

```bash
gh api repos/$REPO/pulls/$PR_NUMBER/comments --paginate \
  --jq '.[] | select(
    .user.login != "BOT_LOGIN_1" and
    .user.login != "BOT_LOGIN_2"
  ) | "[\(.user.login)] \(.path // "general"):\(.line // "N/A") | \(.body | gsub("\n"; " ") | .[0:300])"'
```

各指摘を以下のパターンに分類:

| パターン | 説明 | 例 |
|---|---|---|
| **設計判断** | アーキテクチャ、API設計、集約境界、責務配置 | 「この API は統合すべき」「集約が間違っている」 |
| **命名・意図** | 関数名の意味的適切さ、変数名、規約 | 「関数名が汎用的すぎる」「命名規約と違う」 |
| **テスト戦略** | E2E の要否、テスト構造、不要な前提条件 | 「E2E書いて」「このロックは不要では？」 |
| **セキュリティ** | 暗号処理、認証バイパス、入力検証 | 「skip 経路が本番に残っている」 |
| **ドメイン知識** | 法的要件、ビジネスロジック、外部 API 仕様 | 「電帳法の考慮が必要」「off-by-one」 |
| **コード構造** | 責務分離、ドメイン層の純粋性、リファクタリング提案 | 「副作用を usecase 層に分離すべき」 |
| **サービス間整合性** | データ整合性、API 互換性、トランザクション境界 | 「サービス間の不整合が起きる」 |
| **インフラ・運用** | DB設計、Terraform、GCP権限、デプロイ | 「INDEX が必要」「IAM 権限が足りない」 |

### Phase 5: レポート生成

以下の構成でレポートを生成する:

```markdown
# AI Review Gap Analysis Report

**Repository**: {repo}
**Period**: {since} ~ {today}
**Services**: {service or "all"}

## Summary

| Service | Total PRs | Bot Only | Human Review | AI Approved + Human Caught |
|---------|-----------|----------|--------------|---------------------------|
| ...     | ...       | ...      | ...          | ...                       |

## AI/Bot Reviewers

| Reviewer | Type | Action Pattern |
|----------|------|----------------|
| ...      | ...  | ...            |

## Gap PRs: AI Approved but Humans Caught Issues

### High Severity (Design / Security)

#### PR #{number} — {title}
- **Author**: {author} | **Date**: {date}
- **AI Action**: {bot} APPROVED / "no issues"
- **Human Action**: {reviewer} — {summary of comments}
- **Pattern**: {classification}

### Medium Severity (Naming / Testing / Code Structure)
...

### Low Severity (Style / Readability)
...

## Pattern Analysis

| Pattern | Count | Services | Examples |
|---------|-------|----------|----------|
| ...     | ...   | ...      | ...      |

## AI Blind Spot Map

What AI catches well vs what it misses.

## Human Reviewer Activity

| Reviewer | Reviews | Patterns | Primary Services |
|----------|---------|----------|-----------------|
| ...      | ...     | ...      | ...             |

## Recommendations

Actionable items based on the gap patterns found.
```

### Phase 6: 並列実行の最適化

サービスが複数指定された場合、Agent ツールを使って**並列にデータ収集**する。
各サービスの PR 検索・レビュー取得を別エージェントに委譲し、結果を統合する。

```
Agent("Search {service1} PRs", ...)  ─┐
Agent("Search {service2} PRs", ...)  ─┤─→ 結果を統合 → レポート生成
Agent("Search {service3} PRs", ...)  ─┘
```

1エージェントあたり最大100件のPRを処理。100件を超える場合はページネーションで対応。

## Output

- 指定された output パスにマークダウンレポートを出力
- レポートの要約をユーザーに表示

## Notes

- GitHub API のレート制限に注意。大量のPR（500件以上）がある場合は期間を短くするか、サービスフィルタを使用すること
- private リポジトリの場合、`gh auth status` で適切なスコープが設定されていることを確認
- Bot の判定はリポジトリによって異なる。Phase 1 で必ずユーザーに確認を取ること
