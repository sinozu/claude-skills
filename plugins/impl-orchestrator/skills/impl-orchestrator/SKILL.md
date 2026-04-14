---
name: impl-orchestrator
description: 実装計画を読み込み、PR単位で自律的に実装を進めるオーケストレーター。Implementer/Reviewer/Debuggerの3エージェントをfresh contextで分離し、状態機械で制御する。
arguments:
  - name: plan
    type: string
    description: "実装計画書のパス"
    required: true
  - name: pr
    type: string
    description: "特定のPR番号から開始（例: 3）。省略時は最初の未完了PRから"
    required: false
  - name: mode
    type: string
    description: "'auto'（全PR自律実行）または 'step'（PR完了毎に確認）。デフォルト: step"
    required: false
    default: "step"
---

# impl-orchestrator: 実装オーケストレーター

実装計画書を読み込み、PR 単位で **Implementer → Reviewer → (Debugger)** のサブエージェントを状態機械で回して自律的に実装を進める。

**核心**: 各ロールは Agent tool で **fresh context** に spawn する。同一コンテキストでの自己批評を禁止し、認知バイアスの分離を保証する。

## 使い方

```
/impl-orchestrator --plan path/to/implementation-plan.md
/impl-orchestrator --plan path/to/implementation-plan.md --pr 3
/impl-orchestrator --plan path/to/implementation-plan.md --mode auto
```

---

## サブエージェント構成

| ロール | Agent 定義 | 起動条件 |
|---|---|---|
| **Implementer** | `impl-worker` | 各 PR 開始時、Reviewer 却下後の再試行時 |
| **Reviewer** | `impl-reviewer` | Implementer が READY_FOR_REVIEW を返したとき |
| **Debugger** | `impl-debugger` | Implementer BLOCKED、または Reviewer 2 回 REJECTED |

各エージェントは Agent tool で `subagent_type` を指定して spawn する。

---

## 実行手順

### Step 0: コンテキスト読み込み

1. **実装計画書を Read** で読み込む
2. **対象プロジェクト/サービスを特定**: 計画書の内容から判別
3. **チケット番号を抽出**: ファイル名やドキュメント内から特定。見つからない場合はユーザーに確認
4. **プロジェクトコンテキストを読み込む**:
   - 対象ディレクトリの `CLAUDE.md`（存在すれば）
   - プロジェクトのビルド/テスト設定（Makefile, package.json, Cargo.toml 等）
5. **検証コマンドを決定**: ビルド設定から正確なコマンドを特定
   - TEST: テストコマンド
   - LINT: リンターコマンド
   - FMT: フォーマッターコマンド（あれば）
6. **進捗ファイルを確認/作成**: `/tmp/impl-orchestrator/.progress-{計画書ファイル名}.md`
   - `/tmp/impl-orchestrator/` ディレクトリが存在しなければ `mkdir -p` で作成
   - 既存なら Read して再開ポイントを特定
   - なければ新規作成（後述のフォーマット）

### Step 1: タスク選択

1. 実装計画書から **PR 一覧**をパース（`### PR N:` または `PR N:` の見出し/リスト項目を検出）
2. 進捗ファイルで完了(✅)/ブロック(🚫)済みを確認
3. `--pr` 指定があればその PR を選択
4. **依存関係チェック**: 各 PR の依存記述と「実装フロー」セクションを読み、前提 PR が完了していることを検証
5. 次に実行する PR を選択し、ユーザーに表示:
   ```
   ## 次の実装: PR {N} - {タイトル}
   - 依存: {依存PR or なし}
   - チェック項目: {N}個
   - 予想影響ファイル: {ファイルリスト概要}
   ```
6. step モードではここでユーザー確認を待つ。auto モードでは即座に開始

### Step 2: Per-PR イテレーション（状態機械）

以下の変数を PR ごとに管理する:
- `reviewer_rejections`: 0（Reviewer の REJECTED 回数）
- `debug_rounds`: 0（Debugger の RETRY_TASK 回数）
- `reviewer_feedback`: null（最新の REMEDIATION）
- `debugger_fix_plan`: null（最新の FIX_PLAN）

```
① ブランチ作成
   前の PR ブランチが存在する（依存関係あり）→ そこから分岐
   それ以外 → main から分岐
   ブランチ名: {ticket}-pr{N}-{short-desc}（kebab-case）

② Implementer spawn
   Agent tool で impl-worker を spawn。prompt に以下を含める:
   ─────────────────────────────────────
   - PR番号とタイトル
   - 実装計画書の該当PRセクション全文
   - プロジェクトコンテキスト（CLAUDE.md 等）
   - 検証コマンド (TEST, LINT, FMT)
   - Implementation Notes（進捗ファイルから）
   - reviewer_feedback があれば「Reviewer フィードバック」として追加
   - debugger_fix_plan があれば「Debug レポート」として追加
   ─────────────────────────────────────

   戻り値の STATUS をパース:
   ├─ READY_FOR_REVIEW → ③ へ
   ├─ BLOCKED          → ⑤ へ
   └─ NEEDS_CONTEXT    → 1 回だけ NEEDS の内容を調べて再 spawn。再度 NEEDS_CONTEXT なら ⑤ へ

③ Reviewer spawn
   git diff を取得: `git diff {base-branch}...HEAD`
   Agent tool で impl-reviewer を spawn。prompt に以下を含める:
   ─────────────────────────────────────
   - PR番号とタイトル
   - 実装計画書の該当PRセクション全文
   - git diff の出力
   - 検証コマンド (TEST, LINT)
   - CHANGED_FILES のリスト
   - プロジェクト固有のアーキテクチャルール（あれば）
   ─────────────────────────────────────

   戻り値の VERDICT をパース:
   ├─ APPROVED                    → ④ へ
   ├─ REJECTED (reviewer_rejections < 2)
   │    reviewer_rejections += 1
   │    reviewer_feedback = REMEDIATION
   │    → ② へ（再 Implementer）
   └─ REJECTED (reviewer_rejections >= 2)
        → ⑤ へ（Debugger）

④ 完了処理
   4a. step モードの場合、ユーザーに提示:
       - 変更ファイル一覧
       - git diff --stat
       - Reviewer の SUMMARY
       - 「コミットしてよいですか？ (y/n/skip/stop)」
       y → 続行、n → diff 表示して再判断、skip → スキップ、stop → 停止

   4b. コミット:
       - CHANGED_FILES を個別に git add（git add -A 禁止）
       - git commit: feat({scope}): PR{N} {description}
       - フォーマット: Co-Authored-By を含める

   4c. 進捗更新:
       - 進捗ファイルの該当 PR を ✅ に更新
       - Implementer の NOTES が「なし」でなければ Implementation Notes に追記

   4d. 次の PR へ（Step 1 に戻る）

⑤ Debugger spawn
   Agent tool で impl-debugger を spawn。prompt に以下を含める:
   ─────────────────────────────────────
   - PR番号とタイトル
   - エラー/失敗の内容（BLOCKED_REASON、テスト出力、FINDINGS）
   - git diff の出力
   - Implementer 試行回数、Reviewer 却下回数
   - reviewer_feedback（あれば）
   ─────────────────────────────────────

   戻り値の NEXT_ACTION をパース:
   ├─ RETRY_TASK (debug_rounds < 2)
   │    debug_rounds += 1
   │    debugger_fix_plan = FIX_PLAN
   │    reviewer_feedback = null（リセット）
   │    → ② へ（Implementer に FIX_PLAN 込みで再 spawn）
   ├─ RETRY_TASK (debug_rounds >= 2)
   │    → 上限超過: 🚫 ブロック記録、次 PR へ
   ├─ BLOCK_TASK
   │    → 🚫 ブロック記録（ROOT_CAUSE を備考に）、次 PR へ
   └─ STOP_FOR_HUMAN
        → 🚫 ブロック記録、ユーザーに詳細報告して停止

⑥ 上限チェック（② ③ ⑤ の各遷移前に確認）
   reviewer_rejections > 2 or debug_rounds > 2
   → 強制ブロック: 進捗ファイルに 🚫 記録、次 PR へ
```

### Step 3: 完了レポート

全 PR の処理後（または停止時）に以下を出力:

```markdown
## 実装レポート

### 結果サマリ
- 完了: {N}/{total} PR
- ブロック: {N} PR
- 未着手: {N} PR

### 完了 PR
| PR | ブランチ | 試行回数 |
|---|---|---|

### ブロック PR
| PR | 原因 | カテゴリ | 推奨 |
|---|---|---|---|

### Implementation Notes（横断的な学び）
{蓄積された全 Notes}

### 次のステップ
- 各ブランチから PR を作成
- ブロック PR の手動解消
```

---

## 進捗ファイルフォーマット

**ファイル名**: `.progress-{計画書ファイル名}.md`
**場所**: `/tmp/impl-orchestrator/`
**役割**: 中断からの再開点。このファイルがあれば Step 0 で状態を復元できる。

```markdown
# 実装進捗: {機能名}

## メタ情報
- 計画書: {パス}
- プロジェクト: {対象ディレクトリ}
- チケット: {ticket}
- 開始日: {YYYY-MM-DD}
- 最終更新: {YYYY-MM-DD HH:MM}
- モード: {auto | step}

## PR 進捗

| PR | ステータス | ブランチ | 試行 | 備考 |
|---|---|---|---|---|
| PR 1: {タイトル} | ✅ 完了 | {branch} | 1 | |
| PR 2: {タイトル} | 🔄 実装中 | {branch} | 2 | Reviewer 1回却下 |
| PR 3: {タイトル} | 🚫 ブロック | - | 3 | {理由} |
| PR 4: {タイトル} | ⏳ 待機 | - | 0 | PR 2 依存 |

## Implementation Notes
- PR 1: {学び}
- PR 2: {学び}
```

---

## リトライ上限

| 種別 | 上限 | 超過時 |
|---|---|---|
| Reviewer REJECTED → Implementer 再試行 | 2 回 | 3 回目 REJECTED → Debugger |
| Debugger RETRY → Implementer 再試行 | 2 ラウンド/PR | 🚫 ブロック、次 PR へ |
| NEEDS_CONTEXT → Implementer 再 spawn | 1 回 | BLOCKED 扱い → Debugger |
| 構造化出力パース失敗 → 再 spawn | 1 回 | BLOCKED 扱い |

---

## 構造化出力のパースルール

サブエージェントの出力から**構造化フィールドだけ**をパースする。散文部分は状態遷移に使わない。

| エージェント | パースするフィールド | 有効な値 |
|---|---|---|
| Implementer | `STATUS:` | `READY_FOR_REVIEW`, `BLOCKED`, `NEEDS_CONTEXT` |
| Reviewer | `VERDICT:` | `APPROVED`, `REJECTED` |
| Debugger | `NEXT_ACTION:` | `RETRY_TASK`, `BLOCK_TASK`, `STOP_FOR_HUMAN` |

フィールドが見つからない場合 → 1 回だけ再 spawn。再度見つからない → BLOCKED 扱い。

---

## 禁止事項（安全装置）

- `git reset --hard`, `git clean -f` 等の破壊的操作
- `git add -A` / `git add .`（変更ファイルを個別に `git add` すること）
- 構造化フィールド以外の散文解釈による状態遷移
- 実装計画と矛盾する実装の黙認（必ず BLOCKED に記録）
- テスト/リント未実行でのレビュー APPROVED
- 自動生成ファイルの手動編集
- 進捗ファイルの削除

---

## 学習の前方伝播

```
PR N の Implementer NOTES
    ↓ (進捗ファイルの Implementation Notes に追記)
PR N+1 の Implementer prompt に注入
```

記録すべき学び:
- テスト実行の前提条件（Docker、環境変数等）
- プロジェクト固有のコード配置パターン
- 依存ライブラリの挙動に関する発見
- 既存コードの非自明な慣習

記録すべきでないもの:
- 個別 PR に閉じた情報（特定の変数名等）
- 一時的なワークアラウンド

---

## ユーザー確認（step モード）

各 PR の完了処理で、コミット前に提示:

```
## PR {N} 完了: {タイトル}

変更ファイル:
  {一覧}

変更規模: +{additions} -{deletions} ({files} files)

Reviewer: {SUMMARY}

> コミットしますか？
> y: コミットして次へ / n: diff を表示 / skip: スキップ / stop: 停止
```

---

## 中断と再開

- **中断時**: `/tmp/impl-orchestrator/` の進捗ファイルが残る。作業中のブランチも残る
- **再開時**: 同じ `--plan` で再実行 → `/tmp/impl-orchestrator/` から進捗ファイルを検出し、✅ 以外の最初の PR から再開
- **リセット**: 進捗ファイルを手動削除してから再実行（`rm /tmp/impl-orchestrator/.progress-*.md`）
- **注意**: OS 再起動で `/tmp/` はクリアされるため、長期間の中断には向かない
