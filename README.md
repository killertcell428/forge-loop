# Forge Loop

> **Claude Code に、OSS の継続改善を「数時間おきに 調査 → 実装 → GitHub リリース」まで全任せできるループ。** 鍵は AI と人の境界線設計。

[![Status](https://img.shields.io/badge/status-experimental-orange)]()
[![License](https://img.shields.io/badge/license-MIT-blue)]()

---

## これは何か

AI に「OSS を良くしておいて」と丸投げすると、暴走する。
逆に厳格な脚本を与えると、何もできなくなる。

**Forge Loop は「外枠を人が決め、中身を AI に任せる」設計パターン**。

実例: [Aigis](https://github.com/...) (LLM セキュリティスキャナ) で **5 日 / 17 サイクル / 11 リリース (v1.0.1 → v1.0.12) を半自動運用**。実データは [`examples/aigis/snapshot/`](./examples/aigis/snapshot/) に丸ごと公開。

---

## 5 秒で分かる仕組み

```
[6h ごとに cron が起動]
       ↓
  ROTATION.md を読む → 今回のドメインを決定 (mod N)
       ↓
  research/<domain>.md ← AI が論文/CVE/issue を調査
       ↓
  changes/<日時>.md ← 実装案を記録
       ↓
  通常のコード変更 + テスト追加
       ↓
  巨大 / 破壊的なものは pending/ へ隔離
       ↓
  累積しきい値で CHANGELOG 追記 + git tag
       ↓
  GitHub Actions が release
```

---

## 大事な点 3 つ

### ① Rotate (巡回)

ドメイン一覧を人が書き、`NEXT_INDEX` mod N で順番に消化。**「次に何やるか」を AI に選ばせない。** これで一点特化と暴走を防ぐ。

### ② Materialize (物質化)

各段階の出力が必ずファイルに残る規約。**「考えただけ」「実装したけど記録なし」を構造的に不可能にする。** 後から人が監査できる。

### ③ Defer (保留)

破壊的 / 大規模 / 不確実な提案は実装せず `pending/` に隔離。**「やらない判断」を AI に許す。** これがないと AI は「とにかく実装する」に倒れて事故る。

詳細は [CONCEPT.md](./CONCEPT.md) を参照。

---

## クイックスタート

### 1. プラグインを試す

ローカルで:

```bash
git clone https://github.com/killertcell428/forge-loop
claude --plugin-dir ./forge-loop/plugin
```

公式仕様の `.claude-plugin/plugin.json` + `skills/<name>/SKILL.md` + `agents/<name>/AGENT.md` 構造になっているので、そのまま読み込めます (marketplace 登録はまだ)。

### 2. テンプレートをコピー

```bash
npx degit killertcell428/forge-loop/template my-project/
cd my-project
```

### 3. ドメインを定義

`auto-improvement/domains.yaml` を編集して、自分の OSS の改善対象を書く:

```yaml
domains:
  - id: 0
    name: api-docs
    description: API ドキュメントの不足箇所
  - id: 1
    name: error-messages
    description: ユーザ向けエラーメッセージの改善
  # ...
```

### 4. cron を仕掛ける

GitHub Actions / 手元の cron / Claude Code の scheduled-tasks のどれでも OK。

```yaml
# .github/workflows/forge-cycle.yml
on:
  schedule:
    - cron: '0 */6 * * *'  # 6h ごと
```

詳細は [docs/architecture.md](./docs/architecture.md)。

---

## リポジトリ構成

```
forge-loop/
├── CONCEPT.md              # 北極星ドキュメント
├── README.md               # この文書
├── docs/
│   ├── architecture.md     # 詳細設計
│   ├── boundary-design.md  # AI と人の境界線設計の指針
│   ├── adapters.md         # 自分の OSS への移植ガイド
│   └── case-aigis.md       # Aigis 事例集
├── plugin/                 # Claude Code Plugin (公式仕様準拠)
│   ├── .claude-plugin/
│   │   └── plugin.json    # メタ情報のみ (name/version/author/...)
│   ├── skills/
│   │   ├── forge-research/SKILL.md
│   │   ├── forge-plan/SKILL.md
│   │   ├── forge-implement/SKILL.md
│   │   ├── forge-test/SKILL.md
│   │   └── forge-release/SKILL.md
│   └── agents/
│       └── forge-orchestrator/AGENT.md
├── template/               # 新規プロジェクト用スターター
│   ├── auto-improvement/
│   │   ├── ROTATION.md
│   │   ├── INDEX.md
│   │   ├── domains.yaml
│   │   ├── research/
│   │   ├── changes/
│   │   └── pending/
│   └── .github/workflows/
│       └── forge-cycle.yml
├── examples/
│   └── aigis/              # Aigis 実運用スナップショット
└── articles/
    ├── zenn-main.md        # Zenn 本編記事
    └── qiita-variant.md    # Qiita / note 向け派生記事
```

---

## ステータス

実験段階。Aigis での実運用は継続中だが、Plugin としての配布は未完成。
コンセプトと参考実装は揃っているので、自分の OSS にコピーして手で動かすことは可能。

---

## ライセンス

MIT
