---
name: drive-stacked-prs
description: gh-stack で作った stacked PR を下から順に merge まで運転するスキル。N 分ごとに先頭の未 merge PR を見て、前の PR が merge されたら trunk を取り込んで ready にし、AI review bot の判定を待ち、approve されたら gh stack merge で squash merge して次へ進む。REJECT は指摘の妥当性を判定して直すか反論し、人の承認が要る PR は人待ちとして報告する。「stack を順に merge して」「3 分ごとに PR の状態を確認して進めて」「drive-stacked-prs」などで起動。
user-invocable: true
argument-hint: <stack 番号 or 下から順の PR 番号列> [interval=3m] [build=<ビルドコマンド>] [reviewer=<AI review bot のログイン名>]
---

# drive-stacked-prs: stacked PR を下から順に merge まで運転する

前提は 2 つ。

- [gh-stack](https://github.com/github/gh-stack) で作った stack (`gh stack view --json` で状態が取れる)
- PR を AI review bot がレビューし、APPROVE なら auto-merge が入る運用。人の承認が要る PR は bot がラベル (以下 `hold`) で示す

ラベル名・bot 名・ビルドコマンドはリポジトリごとに違う。起動時に「入力」の表で確定してから始める。

## 入力

| 項目 | 決め方 |
|---|---|
| 対象 PR の順序 | stack 番号か PR 番号列を受け取り、`gh stack view --json` の `branches[]` (下から上) で順序とブランチ名を確定する。merge 済み (`isMerged`) は飛ばす |
| interval | 既定 3 分。`/loop <interval> <このスキルの tick プロンプト>` で cron を張る |
| build | 変更したパッケージのビルドやテスト (例: `go build -o /dev/null ./services/foo/...`) |
| reviewer | AI review bot の GitHub ログイン名。分からなければ直近の merge 済み PR の reviews から探す |
| ラベル | 人の承認待ちを示すラベル (`hold` 等) と、AI review を起動するラベル (`ai-review` 等)。リポジトリの `.github/workflows` を読んで確定する |

## 起動時にやること

1. `gh stack view --json` で trunk・順序・各 PR 番号・state・`needsRebase` を取る
2. AI review が走らない PR (設定ファイルだけの PR など、workflow の paths 条件から外れるもの) を見分ける。人の approve が要る
3. `/loop <interval> <tick プロンプト>` で cron を作る。tick プロンプトは順序・ブランチ名・stack 番号・入力の値・下の手順を全部含めて自己完結にする (cron はこの会話の文脈を持たない前提で書く)
4. 最初の tick をその場で実行する

## 各 tick の手順

先頭の未 merge PR を N とする。

1. **N が MERGED** → 同じ tick で次の PR に進む
2. **N が draft で、1 つ前の PR が MERGED (または N が最下層)**:
   - `git fetch origin <trunk>` → N のブランチを checkout → `git merge --no-edit origin/<trunk>`
   - 上の層へ順に checkout → `git merge --no-edit <親ブランチ>` (rebase はしない)
   - build を通す → `gh stack push` → `gh stack submit --auto` → `gh pr ready N`
   - AI review が走らない PR なら「人の approve 待ち」と報告する
3. **N が open**: `gh pr view N --json labels,reviews,autoMergeRequest,mergeStateStatus,state` と
   `gh api repos/<owner>/<repo>/pulls/N/reviews` (reviewer の最新 review と対象 commit) を見る
   - `hold` + reviewer の `CHANGES_REQUESTED` (REJECT) → review 本文を読み、指摘ごとに妥当性を判定する。
     事実誤認・合意済みの設計への異議・API の契約を変える提案は反映せず、理由をコメントする。
     妥当なら直してテストし、上の層へ merge で伝播 → push → `hold` と review 起動ラベルを外して付け直す → 対応コメントを残す。
     ユーザーには指摘と対応を報告する
   - `hold` + reviewer の `COMMENTED` (人のレビューが必要という判定) → コード修正は不要。人の承認待ちとして報告し、ループは続ける
   - APPROVED (bot か人) があり `autoMergeRequest` が有効 → 待つ
   - APPROVED があり auto-merge が無効で `mergeStateStatus` が CLEAN → `gh stack merge N --yes --squash`
     (`gh pr merge` は stack の PR では「asynchronous merge REST API を使え」と拒否される)
   - それ以外 (CI 実行中 = BLOCKED / UNKNOWN) → 待つ
4. 最上層が MERGED → `CronDelete` でループを止め、全 PR の結果をまとめて報告する

## 報告

- 各 tick は 1〜3 行。「何を待っているか」(どの check・どの review) を書く
- REJECT と人待ちは、指摘の要点と自分の対応 (直した / 反論した / 人待ち) を書く

## 落とし穴

- **force push は使わない**。下の層の修正は上の層へ `git merge` で伝播し、`gh stack push` は fast-forward で通す。`gh stack rebase` / `gh stack sync` は使わない (sync はローカルの trunk を更新するので、trunk が別の worktree に checkout されていると失敗する)
- AI review が `labeled` / `ready_for_review` でしか起動しない運用では、push しただけでは再レビューされない。修正後は `hold` と起動ラベルを外して付け直す
- AI review が実装計画と diff を突き合わせる運用では、意図した仕様が計画と違うと正しい実装が REJECT され続ける。先に計画を直す PR を出す
- 下の PR が squash merge されると上の PR の base は自動で trunk になるが、diff には下の層の commit が残って見える。trunk を merge で取り込むと消える
- API スキーマ (proto 等) を変えた層の上は、生成クライアントの更新が trunk に入るまでビルドできないことがある。入ったら trunk を取り込み、vendor 等の依存を同期する
- Stop hook の lint がビルドできない層で止まるときは、作業ツリーを依存の無い下の層に checkout して戻す
- `gh pr merge --squash` ではなく `gh stack merge <PR> --yes --squash`
- CI の E2E だけが timeout するときは、PR で足した E2E の構造を疑う。親テストがロックを持ったまま並列のサブテストを起動すると、並列枠とロックの待ちが循環して deadlock する
- ブランチ保護で「push 時に古い approve を取り消す」が無効なら、approve 後に push しても approve は残る。逆に古い CHANGES_REQUESTED は dismiss しないと merge を塞ぐ
