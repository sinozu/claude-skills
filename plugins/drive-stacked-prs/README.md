# drive-stacked-prs

[gh-stack](https://github.com/github/gh-stack) で作った stacked PR を、下から順に merge まで運転するスキル。

## What it does

`/loop` で N 分ごとに stack の先頭の未 merge PR を見て、次を繰り返す:

- 前の PR が merge されたら trunk を merge で取り込み、上の層へ伝播して ready にする
- AI review bot の判定を待つ。APPROVE なら `gh stack merge` で squash merge して次の層へ
- REJECT は指摘ごとに妥当性を判定し、直すか理由をコメントして再レビューを依頼する
- 人の承認が要る PR は人待ちとして報告し、ループは続ける
- 最上層が merge されたらループを止めて結果を報告する

force push と rebase は使わない。下の層の修正は `git merge` で上の層へ伝播する。

## Usage

```
/drive-stacked-prs 12
/drive-stacked-prs 101 102 103 interval=5m build="go build ./..." reviewer=my-review-bot
```

## Prerequisites

- `gh` CLI と [gh-stack](https://github.com/github/gh-stack) 拡張が認証済み
- PR を AI review bot がレビューし、人の承認が要る PR をラベルで示す運用
- Claude Code の `/loop` (cron) が使えること

## Install

```bash
claude plugin marketplace add sinozu/claude-skills
claude plugin install drive-stacked-prs@sinozu-skills
```

Manual:

```bash
git clone https://github.com/sinozu/claude-skills.git
cp -r claude-skills/plugins/drive-stacked-prs/skills/* ~/.claude/skills/
```
