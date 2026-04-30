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
   - TEST: 単体テストコマンド（例: `go test ./...` / `npm test` / `pytest`）
   - E2E_TEST: E2E テストコマンドパターン（あれば）
     - **スコープ指定の placeholder を含む**形式で渡すこと
     - 例: `go test -run {TEST_NAME} ./e2etests/...` / `jest -t '{TEST_NAME}'` / `pytest -k '{TEST_NAME}'`
     - エージェントは変更箇所から TEST_NAME を特定して置換する
     - E2E テスト基盤がない or 該当しないプロジェクトでは N/A
   - LINT: リンターコマンド（例: `make lint` / `npm run lint` / `cargo clippy`）
   - FMT: フォーマッターコマンド（例: `make fmt` / `npm run format` / `cargo fmt`）
6. **進捗ファイルを確認/作成**: `/tmp/impl-orchestrator/.progress-{計画書ファイル名}.md`
   - `/tmp/impl-orchestrator/` ディレクトリが存在しなければ `mkdir -p` で作成
   - 既存なら Read して再開ポイントを特定
   - なければ新規作成（後述のフォーマット）

### Step 0.5: 環境 pre-check

Implementer/Reviewer が環境問題の切り分けに時間を消費するのを防ぐため、orchestrator が事前に環境状態を確認する。
プロジェクトに該当しない項目は skip して構わない（proto-only PR や、ビルド/テスト環境を必要としない PR では本ステップ全体を skip）。

**キャッシュ機構**:
- `/tmp/impl-orchestrator/.env-cache-{計画書ファイル名}.txt` を確認
- 存在し、最終更新から **30 分以内** ならキャッシュを採用して本ステップ実行を skip
- それ以外は以下を実行し、結果をキャッシュに書き出す（`ENV_STATUS=...\nENV_DETAILS=...\nENV_BASELINE_VERIFIED=...\nTIMESTAMP=...` 形式）

**チェックの原則**: 起動有無の判定は **「事実観測」を優先** し、ハードコードされたポート番号にいきなりフォールバックしない。
具体的には、各サービスの `host:port` を以下の順で解決する:

1. **環境変数を最優先**:
   - `env | grep -E '_(EMULATOR_)?HOST=' || true` で `*_EMULATOR_HOST` 系
     （`SPANNER_EMULATOR_HOST`, `PUBSUB_EMULATOR_HOST`, `DATASTORE_EMULATOR_HOST`,
     `FIRESTORE_EMULATOR_HOST`, `BIGTABLE_EMULATOR_HOST` 等）と、`<SERVICE>_HOST` 系
     （`POSTGRES_HOST`, `REDIS_HOST` 等）を抽出
   - 値の形式: `host:port` または `host`（後者の場合 port は別変数または compose で解決）
2. **`compose.yaml` / `docker-compose.yml` の host 側 ports**:
   - `services.<service>.ports` の `"<host>:<container>"` から host 側ポートを抽出
   - host が省略形（例: `"10004:9010"`）の場合は `localhost`
3. **`docker ps` の listening ポート（システム全体観測）**:
   - `docker ps --format 'table {{.Names}}\t{{.Ports}}\t{{.Status}}'` で全コンテナを観測
   - **`docker compose ps` は使わない**（cwd/worktree 依存で別所から起動された stack を見逃すため）
   - 該当サービス（`spanner`, `postgres`, `redis` 等）を grep し、`Ports` 列から
     `0.0.0.0:<host>-><container>/tcp` の host 側を抽出
4. **デフォルトポート（最終フォールバック）**: 1〜3 で解決できなかった場合のみ採用:
   - Spanner: 9010 (gRPC) / 9020 (REST)
   - Pub/Sub: 8085 / Datastore: 8081 / Firestore: 8080 / Bigtable: 9000
   - PostgreSQL: 5432 / Redis: 6379 / MySQL: 3306

**チェック項目**（プロジェクトに該当するものだけ実行）:

1. **Docker コンテナ起動状況**:
   - `docker ps --format 'table {{.Names}}\t{{.Ports}}\t{{.Status}}'` で全コンテナ観測
   - 該当サービスのコンテナが見えなければ「起動していない可能性あり」として 2 へ
2. **Emulator 系サービス**（プロジェクトが該当 emulator を使う場合のみ）:
   - 上記の解決手順で `host:port` を確定
   - `lsof -nP -iTCP:<port> -sTCP:LISTEN` でホストプロセスの LISTEN を確認（プロセス名まで分かる）
   - 失敗時のフォールバックとして `nc -z <host> <port>`
3. **wrench 動作状態**（wrench をテストで使う場合のみ）:
   - `wrench instance list` を実行（環境変数 `SPANNER_EMULATOR_HOST` で接続先を解決するため、解決原則と整合）
4. **その他のサービス**: PostgreSQL / Redis / MySQL 等も同じ解決手順 + 疎通確認を適用

**結果の構造化**:
- 全項目正常 → `ENV_STATUS: OK`
- 1 つでも失敗 → `ENV_STATUS: ISSUE` + `ENV_DETAILS`（**解決元を必ず明記**）
  - 例: `Spanner Emulator (resolved from $SPANNER_EMULATOR_HOST=localhost:10004) unreachable on lsof check`
  - 例: `Pub/Sub Emulator (resolved from compose.yaml services.pubsub.ports[0]=8085) unreachable`
  - 例: `Spanner Emulator (resolved from default 9010 — no env/compose hit) unreachable`
- 全項目 skip → `ENV_STATUS: NOT_APPLICABLE`

`ENV_DETAILS` に解決元を明記することで、ハードコードのデフォルトポート由来の誤判定（例: 実際は `localhost:10004` で起動しているのに `9020` を確認して NG 判定）を後追いで識別できる。

**baseline 再現判定**（`ENV_BASELINE_VERIFIED` の決定）:
- `ENV_STATUS: OK` または `NOT_APPLICABLE` → `ENV_BASELINE_VERIFIED: true`（環境問題なし、もしくは判定不要）
- `ENV_STATUS: ISSUE` のとき、`ENV_DETAILS` の症状が **実装に依存しない種類**（emulator/サービスの起動失敗、外部依存のタイムアウト等）であれば `ENV_BASELINE_VERIFIED: true`
- 症状が実装の影響を受けうる種類（個別テストの失敗、ビルドエラー等）なら `ENV_BASELINE_VERIFIED: false`
  - この場合 Reviewer は従来どおり `git stash` 経由で baseline 検証を行う

**結果の伝搬先**:
- Step 2 ② Implementer prompt（後述。`ENV_STATUS` / `ENV_DETAILS`）
- Step 2 ③ Reviewer prompt（後述。`ENV_STATUS` / `ENV_DETAILS` / `ENV_BASELINE_VERIFIED`）
- 進捗ファイルの「Implementation Notes」末尾に `- env pre-check ({YYYY-MM-DD HH:MM}): {ENV_STATUS} / {ENV_DETAILS}` を 1 行追記

**バナー出力**:
```
[進捗ヘッダー ── 0.5 環境 pre-check]
>> 環境 pre-check: {ENV_STATUS} (baseline_verified: {true|false})
   {ENV_DETAILS or "全項目 OK" or "対象外 (skip)" or "キャッシュ採用 (age {分}m)"}
```

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

### 進捗バナー出力ルール

状態遷移のたびに **進捗ヘッダー + 遷移バナー** をユーザーに出力する。
auto / step 両モードで同一の内容を出力する（auto では一時停止しないだけ）。

**進捗ヘッダー**（各遷移で 1 行目に出力）:
```
───── PR {N}/{total} [{タイトル30文字まで}] ── {ステップ名} ── {R回数}/2R {D回数}/2D ─────
```
- ステップ名: 0.5 環境 pre-check / ① ブランチ作成 / ② Implementer / ③ Reviewer / ④ 完了処理 / ⑤ Debugger / ⑥ 上限チェック
- Step 0.5 は PR 共通の前処理のため、ヘッダーは PR 番号を省いた `───── 0.5 環境 pre-check ─────` の短縮形にしてよい
- R = reviewer_rejections, D = debug_rounds

**遷移バナー**（ヘッダーの次行に `>>` で出力）: 各状態で何が起きたか/起きるかを 1〜3 行で表示する。詳細は以下の状態機械内に記載。

```
① ブランチ作成
   前の PR ブランチが存在する（依存関係あり）→ そこから分岐
   それ以外 → main から分岐
   ブランチ名: {ticket}-pr{N}-{short-desc}（kebab-case）

   【後処理】vendor 整合チェック (Go プロジェクトのみ):
     `go.mod` が存在する場合に `go mod vendor` を実行。
     差分が出ても **コミット対象外** として扱う（vendor は通常 commit しない運用が多いため）。
     差分が出た場合は進捗ファイル Implementation Notes に
       `- vendor 自動更新 ({件数} ファイル) at PR{N}`
     を 1 行追記。これにより Implementer/Reviewer が `vendor/modules.txt` 不整合に時間を取られない。
   【スキップ条件】proto-only PR、Go 以外のプロジェクト、`go.mod` がないリポジトリ

   【バナー出力】ブランチ作成 + vendor 整合チェック後:
   [進捗ヘッダー ── ① ブランチ作成]
   >> ブランチ作成: {branch-name} (from {base-branch})
   >> vendor 整合: {OK | 自動更新 N ファイル | skip}

② Implementer spawn

   【前処理 1】caller pre-grep:
     計画書の該当 PR セクションから「変更対象 symbol」（関数名/型名/RPC 名/メソッド名）を抽出する。
     抽出パターンの例:
       - 「`HandleSubscriptionWebhook` のシグネチャ拡張」→ symbol: `HandleSubscriptionWebhook`
       - 「`UserUseCase.UpdateXxx` を変更」→ symbol: `UpdateXxx`
     抽出できた symbol について `git grep -ln "<symbol>" -- <scope>` を実行:
       - Go: `*.go`
       - TS/JS: `*.ts` `*.tsx` `*.js` `*.jsx`
       - Python: `*.py`
     0 件 → `CALLERS: なし`
     1 件 → `CALLERS: 単一（自身の定義のみと推定）`
     2 件以上 → caller リストを `CALLERS:` としてプロンプトに注入
     【スキップ条件】計画書から symbol が抽出できない / 新規追加のみで既存 symbol 変更なし → `CALLERS: skip`

   【前処理 2】test scope hint:
     計画書の「変更対象ファイル」または CHANGED_FILES（再試行時）から所属パッケージを推定し
     `TEST_SCOPE_HINT` を構築する。
     例 (Go): `internal/usecase/customer/*.go` → `./internal/usecase/customer/...`
     例 (TS): `src/handlers/webhook/*.ts` → `src/handlers/webhook`
     【スキップ条件】変更対象ファイルが推定できない場合 → `TEST_SCOPE_HINT: skip`

   【バナー出力】spawn 前:
   [進捗ヘッダー ── ② Implementer]
   >> Implementer 起動中... (チェック項目 {N}個 / CALLERS {件数 or skip} / TEST_SCOPE {hint or skip})
   # feedback / fix_plan のいずれかがある場合のみ追加行を出す:
   #    feedback: {REMEDIATION 1 行抜粋}
   #    fix_plan: {ROOT_CAUSE 1 行}

   Agent tool で impl-worker を spawn。prompt に以下を含める:
   ─────────────────────────────────────
   - PR番号とタイトル
   - 実装計画書の該当PRセクション全文
   - プロジェクトコンテキスト（CLAUDE.md 等）
   - 検証コマンド (TEST, E2E_TEST, LINT, FMT)
   - E2E_TEST パターンの説明（スコープ指定必須、例: `go test -run {TEST_NAME}`）
   - **ENV_STATUS / ENV_DETAILS**（Step 0.5 の結果。ISSUE のときは「該当症状の単体テスト/E2E は SKIPPED_ENV_ISSUE で報告してよい」と明示）
   - **CALLERS**（前処理 1 の結果。リスト or 「なし」or 「単一」or 「skip」）
   - **TEST_SCOPE_HINT**（前処理 2 の結果。Implementer はこれで TEST 実行をスコープする）
   - Implementation Notes（進捗ファイルから）
   - reviewer_feedback があれば「Reviewer フィードバック」として追加
   - debugger_fix_plan があれば「Debug レポート」として追加
   ─────────────────────────────────────

   戻り値の STATUS をパースし、【バナー出力】:

   READY_FOR_REVIEW の場合:
   >> Implementer 完了: READY_FOR_REVIEW
      変更: {CHANGED_FILES の件数}ファイル / TEST_SCOPE: {Implementer 報告の TEST_SCOPE}
      テスト: {TEST_RESULT} / E2E: {E2E_RESULT} / リント: {LINT_RESULT} / fmt: {FMT_RESULT}
      E2E スコープ: {E2E_SCOPE}
   → ③ へ

   BLOCKED の場合:
   >> Implementer 完了: BLOCKED
      理由: {BLOCKED_REASON}
   → ⑤ へ

   NEEDS_CONTEXT の場合:
   >> Implementer 完了: NEEDS_CONTEXT
      必要情報: {NEEDS}
      >> コンテキスト調査後に再spawn...
   → 1 回だけ NEEDS の内容を調べて再 spawn。再度 NEEDS_CONTEXT なら ⑤ へ

③ Reviewer spawn
   git diff を取得: `git diff {base-branch}...HEAD`

   【バナー出力】spawn 前:
   [進捗ヘッダー ── ③ Reviewer]
   >> Reviewer 起動中... (diff: {git diff --stat のファイル数と行数概要})

   Agent tool で impl-reviewer を spawn。prompt に以下を含める:
   ─────────────────────────────────────
   - PR番号とタイトル
   - 実装計画書の該当PRセクション全文
   - git diff の出力
   - 検証コマンド (TEST, E2E_TEST, LINT, FMT)
   - E2E_TEST パターンの説明（スコープ指定必須）
   - Implementer が報告した E2E_SCOPE（妥当性検証のため）
   - **Implementer が報告した TEST_SCOPE**（Reviewer は同範囲で 1a 実行、TEST 全体で 1b 実行）
   - CHANGED_FILES のリスト
   - プロジェクト固有のアーキテクチャルール（あれば）
   - **ENV_STATUS / ENV_DETAILS**（Step 0.5 の結果）
   - **ENV_BASELINE_VERIFIED**: true | false（Step 0.5 で決定。true なら Reviewer は `git stash` 経由の baseline 検証を省略可）
   ─────────────────────────────────────

   戻り値の VERDICT をパースし、【バナー出力】:

   APPROVED の場合:
   >> Reviewer 完了: APPROVED
      機械チェック: テスト 1a={Tests1a}/1b={Tests1b} / E2E {E2E} / リント {Lint} / fmt {Fmt} / TODO {TBD/TODO grep} / 構造 {Architecture}
      Baseline: {BaselineCheck} / E2E スコープ: {E2E_SCOPE}
      総評: {SUMMARY}
   → ④ へ

   REJECTED (reviewer_rejections < 2) の場合:
   >> Reviewer 完了: REJECTED ({reviewer_rejections+1}/2)
      機械チェック: テスト 1a={Tests1a}/1b={Tests1b} / E2E {E2E} / リント {Lint} / fmt {Fmt} / TODO {TBD/TODO grep} / 構造 {Architecture}
      Baseline: {BaselineCheck}
      指摘 ({FINDINGS の件数}件): {FINDINGS の 1件目}
      >> Implementer に差し戻し (REMEDIATION 付き)
   reviewer_rejections += 1, reviewer_feedback = REMEDIATION → ② へ

   REJECTED (reviewer_rejections >= 2) の場合:
   >> Reviewer 完了: REJECTED (上限到達 {reviewer_rejections+1}/2)
      >> Debugger にエスカレーション
   → ⑤ へ

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

   【バナー出力】コミット後:
   >> PR {N} コミット完了: {commit hash 先頭8文字} on {branch-name}

   4c. 進捗更新:
       - 進捗ファイルの該当 PR を ✅ に更新
       - Implementer の NOTES が「なし」でなければ Implementation Notes に追記

   4d. 次の PR へ（Step 1 に戻る）

⑤ Debugger spawn
   【バナー出力】spawn 前:
   [進捗ヘッダー ── ⑤ Debugger]
   >> Debugger 起動中...
      投入理由: {BLOCKED or REJECTED} / Implementer {試行回数}回試行済み

   Agent tool で impl-debugger を spawn。prompt に以下を含める:
   ─────────────────────────────────────
   - PR番号とタイトル
   - エラー/失敗の内容（BLOCKED_REASON、テスト出力、FINDINGS）
   - git diff の出力
   - Implementer 試行回数、Reviewer 却下回数
   - reviewer_feedback（あれば）
   ─────────────────────────────────────

   戻り値の NEXT_ACTION をパースし、【バナー出力】:

   RETRY_TASK (debug_rounds < 2) の場合:
   >> Debugger 完了: RETRY_TASK (debug {debug_rounds+1}/2)
      根本原因: {ROOT_CAUSE}
      カテゴリ: {CATEGORY} / 確信度: {CONFIDENCE}
      修正計画: {FIX_PLAN の 1行目}
      >> Implementer に FIX_PLAN 付きで再spawn...
   debug_rounds += 1, debugger_fix_plan = FIX_PLAN, reviewer_feedback = null → ② へ

   RETRY_TASK (debug_rounds >= 2) の場合:
   >> Debugger 完了: RETRY_TASK だが上限超過 (debug {debug_rounds+1}/2)
      根本原因: {ROOT_CAUSE}
      >> PR {N} を強制ブロック、次の PR へ
   → 🚫 ブロック記録、次 PR へ

   BLOCK_TASK の場合:
   >> Debugger 完了: BLOCK_TASK
      根本原因: {ROOT_CAUSE}
      カテゴリ: {CATEGORY}
      >> PR {N} をブロック記録、次の PR へ
   → 🚫 ブロック記録（ROOT_CAUSE を備考に）、次 PR へ

   STOP_FOR_HUMAN の場合:
   >> Debugger 完了: STOP_FOR_HUMAN
      根本原因: {ROOT_CAUSE}
      カテゴリ: {CATEGORY} / 確信度: {CONFIDENCE}
      修正計画: {FIX_PLAN 全文}
      NOTES: {NOTES}
      >> 人間の判断が必要です。実行を停止します。
   → 🚫 ブロック記録、ユーザーに詳細報告して停止

⑥ 上限チェック（② ③ ⑤ の各遷移前に確認）
   reviewer_rejections > 2 or debug_rounds > 2

   【バナー出力】:
   [進捗ヘッダー ── ⑥ 上限チェック]
   >> 上限超過: reviewer_rejections={N}, debug_rounds={N}
      >> PR {N} を強制ブロック、次の PR へ
   → 進捗ファイルに 🚫 記録、次 PR へ
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
