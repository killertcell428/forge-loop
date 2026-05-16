# Architecture

Forge Loop の詳細設計ドキュメント。

## 全体図

```
┌──────────────────────────────────────────────────────────────┐
│                     Trigger (cron / scheduled-tasks)         │
└────────────────────────────┬─────────────────────────────────┘
                             │ 6h ごと
                             ▼
┌──────────────────────────────────────────────────────────────┐
│              forge-orchestrator (Claude Code agent)          │
└────┬─────────────┬─────────────┬─────────────┬───────────┬───┘
     │             │             │             │           │
     ▼             ▼             ▼             ▼           ▼
 [research]    [plan]       [implement]    [test]      [release]
     │             │             │             │           │
     ▼             ▼             ▼             ▼           ▼
 research/    changes/      <code>/       <tests>/    CHANGELOG
              pending/                                  + tag
                                                         │
                                                         ▼
                                                  ┌──────────────┐
                                                  │ GitHub Actions│
                                                  │  (release.yml)│
                                                  └──────┬───────┘
                                                         │
                                                         ▼
                                                   PyPI / npm /
                                                   Crates.io / etc.
```

## コンポーネント

### 1. Trigger 層

cron や Claude Code scheduled-tasks が、6h ごとに forge-orchestrator を起動する。

選択肢:
- **GitHub Actions cron** (推奨。設定が簡単で再現性が高い)
- **手元の crontab** (PoC 段階で OK)
- **Claude Code scheduled-tasks** (Claude Code を常駐させる場合)

ポイント: **`concurrency` で重複実行を禁止する**こと。前のサイクルが終わる前に次が始まると、ROTATION.md / INDEX.md が壊れる。

### 2. State 層 (auto-improvement/)

サイクル間の状態を **ファイルで**永続化する。DB を使わないのが意図的:
- git の履歴がそのまま監査ログになる
- 人が直接読める・編集できる
- 復旧が `git checkout` だけで済む

```
auto-improvement/
├── ROTATION.md       # NEXT_INDEX, LAST_RUN_UTC
├── domains.yaml      # ドメイン一覧 (人が編集する)
├── INDEX.md          # 全実行の追記式ログ
├── research/         # 段階1の出力
├── changes/          # 段階2の出力
└── pending/          # 段階3で隔離された提案
```

### 3. Orchestrator 層

`plugin/agents/forge-orchestrator.md` で定義された Claude Code エージェント。
**1 サイクル分の実行を 1 プロセスで完結させる**。途中で止めない。

### 4. Skill 層

各段階を切り出した skill 群:

| Skill | 役割 | 入力 | 出力 |
|---|---|---|---|
| forge-research | 外部情報収集 | domain | research/*.md |
| forge-plan | 実装案策定 + pending 判定 | research/*.md | changes/*.md + pending/*.md |
| forge-implement | コード変更 | changes/*.md | git working tree |
| forge-test | テスト追加 + 検証 | changes/*.md | tests/, pass/fail |
| forge-release | CHANGELOG + tag | changes/*.md | CHANGELOG.md, git tag |

各 skill は独立に呼べる (デバッグ用)。本番では orchestrator が順番に呼ぶ。

### 5. Release 層

git tag push をトリガーに、既存の `.github/workflows/release.yml` 等が PyPI / npm にリリースする。**Forge Loop はここに関与しない**。リポジトリの既存リリースパイプラインに乗る。

これが重要: Forge Loop は「自走 commit + tag」までしかやらない。**実際の配布パイプラインの安全性は元から repo が持っているもの**に乗せる。

## サイクル状態の遷移

```
[Idle]
  │
  │ cron fires
  ▼
[ROTATION読込] ── ROTATION.md → current_domain 決定
  │
  ▼
[Research] ── research/<cycle_id>.md を生成 ──► 成果0件なら → [INDEX記録] へ
  │
  ▼
[Plan] ── changes/<utc>.md を生成 + pending/*.md を生成 ──► 実装ゼロなら → [INDEX記録] へ
  │
  ▼
[Implement] ── code 変更 ──► 失敗 Change は pending 格下げ
  │
  ▼
[Test] ── tests/ 追加 + pytest 実行 ──► 失敗 Change は pending 格下げ + コードロールバック
  │
  ▼
[Release] ── 累積 ≥ threshold なら version bump + tag
  │       ── 未満なら CHANGELOG[Unreleased] に追記のみ
  ▼
[INDEX記録] ── INDEX.md に 1 行追加, ROTATION.md の NEXT_INDEX++
  │
  ▼
[Done]
```

## 失敗モード

| モード | 対処 |
|---|---|
| Research で何も見つからない | 空の research/*.md を作って終了 (INDEX には記録) |
| Plan で全提案が pending 行き | 実装段階をスキップ。CHANGELOG 追記なし |
| Implement 中に型エラー | その Change だけロールバック、pending 格下げ。次の Change へ |
| Test で既存テスト fail | その Change だけロールバック、pending 格下げ |
| 3 サイクル連続で空振り | pending/critical-failure-*.md を生成して人に通知 |
| ROTATION.md が壊れている | サイクル開始時に検知して停止 + GitHub issue 作成 |

## 拡張ポイント

将来的に拡張するなら以下:

- **複数 OSS の同時管理** — 1 つのインスタンスで N 個のリポジトリを管理。ROTATION に repo 軸を追加。
- **Adversarial loop** — Research の代わりに red-team-attacker agent を走らせる (Aigis でやっている)。
- **Human-in-the-loop hook** — pending/ に書かれた瞬間に Slack / Discord 通知。
- **Multi-stage release** — patch 自動 / minor 半自動 / major 手動、のような stage gate。

これらは v0.1 のスコープ外。

## なぜこの設計か (FAQ)

### Q. なぜ DB を使わない?

A. git でいい。サイクルの履歴は commit log に残り、`git log auto-improvement/` で時系列が見える。DB を入れると復旧と監査が面倒になる。

### Q. なぜシングルエージェント?

A. Multi-agent は実験には面白いが、本番では協調コストが高い。Forge Loop は **規約と物質化** で人が見える化することを優先しているので、エージェント間の暗黙のやり取りを増やすのは逆効果。

### Q. なぜ Claude Code?

A. Plugin / skill / agent / cron が一通り揃っているから。同等のことは他のエージェントランタイムでもできるが、Forge Loop の参考実装は Claude Code 前提。
