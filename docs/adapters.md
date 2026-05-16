# Adapters — 自分の OSS への移植ガイド

Forge Loop を自分の OSS に導入する手順と、ドメインタイプ別の設計指針。

---

## クイックスタート

### 1. テンプレートをコピー

```bash
# 既存リポジトリの中で:
mkdir -p auto-improvement/{research,changes,pending}
cp /path/to/forge-loop/template/auto-improvement/* auto-improvement/
cp /path/to/forge-loop/template/.github/workflows/forge-cycle.yml .github/workflows/
```

### 2. Plugin をインストール

```bash
claude plugin install forge-loop
```

### 3. ドメインを定義

`auto-improvement/domains.yaml` を編集。**ここが移植作業の本体**。詳細は後述。

### 4. 既存の release.yml を確認

`.github/workflows/release.yml` (または同等) があり、**git tag push でリリースが走る**ことを確認。
無ければ作る (forge-loop は tag を push するだけで、その後の配布は責任を持たない)。

### 5. 試運転

```bash
# 手動で 1 サイクル実行 (workflow_dispatch)
gh workflow run forge-cycle.yml
```

研究フェーズの出力 (research/*.md) を見て、ドメイン定義が機能しているか確認。

---

## ドメイン設計の指針

### 良いドメイン分割の条件

- **N = 5 〜 15** の範囲 (5 未満は反復しすぎ、15 超は触れる頻度が低すぎる)
- **MECE** (Mutually Exclusive, Collectively Exhaustive)
- **粒度が揃っている** (どれも 1 サイクル分の調査・実装に収まる)
- **外部ソースが取得可能** (research フェーズで枯れない)
- **「misc」「その他」を作らない** (ダンピング場と化す)

### 良くないドメイン分割

❌ N = 3 (毎週同じドメインに 2 回戻る)
❌ N = 30 (1 ドメインあたり 20 日に 1 回しか触らない)
❌ "improvements" "fixes" "features" (粒度が抽象的すぎ)
❌ 1 ドメインだけ巨大 (例: "core" が他の 10 倍重い)

---

## ドメインタイプ別の例

### A. セキュリティスキャナ (Aigis 型)

```yaml
domains:
  - id: 0
    name: prompt-injection
    sources: [arxiv, owasp-llm, github-advisories]
  - id: 1
    name: agent-tool-abuse
  - id: 2
    name: data-exfiltration
  - id: 3
    name: jailbreak-techniques
  - id: 4
    name: supply-chain-attacks
  - id: 5
    name: model-stealing
  - id: 6
    name: pii-leakage
  - id: 7
    name: adversarial-suffixes
  - id: 8
    name: rag-poisoning
  - id: 9
    name: incident-postmortems
```

### B. Linter / Formatter

```yaml
domains:
  - id: 0
    name: python-rules
    sources: [pep-updates, ruff-issues, flake8-issues]
  - id: 1
    name: typescript-rules
  - id: 2
    name: go-rules
  - id: 3
    name: rust-rules
  - id: 4
    name: security-rules-cross-language
  - id: 5
    name: performance-rules-cross-language
  - id: 6
    name: style-rules
  - id: 7
    name: documentation-rules
```

### C. CLI ツール

```yaml
domains:
  - id: 0
    name: subcommand-init
  - id: 1
    name: subcommand-build
  - id: 2
    name: subcommand-deploy
  - id: 3
    name: subcommand-test
  - id: 4
    name: error-messages
  - id: 5
    name: help-text
  - id: 6
    name: shell-completion
  - id: 7
    name: config-validation
```

### D. ドキュメント生成

```yaml
domains:
  - id: 0
    name: getting-started
  - id: 1
    name: api-reference
  - id: 2
    name: tutorials
  - id: 3
    name: troubleshooting
  - id: 4
    name: faq
  - id: 5
    name: migration-guides
  - id: 6
    name: example-projects
```

### E. テストカバレッジ拡張

```yaml
domains:
  - id: 0
    name: core-module-tests
  - id: 1
    name: api-handler-tests
  - id: 2
    name: util-tests
  - id: 3
    name: integration-tests
  - id: 4
    name: edge-case-tests
  - id: 5
    name: regression-tests-from-past-issues
  - id: 6
    name: performance-tests
```

---

## pending 判定基準のカスタマイズ

`plugin.json` の `defer_thresholds` を自分のプロジェクトに合わせて調整:

```json
{
  "config": {
    "defer_thresholds": {
      "max_loc_per_change": 100,           // 既定: 100
      "block_api_breaking": true,          // 既定: true
      "block_dependency_changes": true,    // package.json / pyproject.toml への変更を pending 行きに
      "block_workflow_changes": true,      // .github/workflows/ の変更を pending 行きに
      "max_new_files_per_change": 5        // 新規ファイル数の上限
    }
  }
}
```

**緩めるべきでない**もの:
- `block_api_breaking` — これを false にすると公開 API が勝手に壊れる
- `max_loc_per_change` を 1000 などに — 暴走の温床

**緩めても OK** なもの:
- `max_new_files_per_change` — リポジトリの性質によっては多くの新規ファイルが普通
- `max_loc_per_change` を 200 程度に — 規模感が大きいリポジトリでは

---

## リリース判定のカスタマイズ

```json
{
  "config": {
    "release_threshold": {
      "min_accumulated_units": 3,          // 既定: 3
      "max_cycles_without_release": 10,    // 10 サイクル溜まったら強制リリース
      "version_bump_strategy": "patch"     // patch / minor / major / "auto"
    }
  }
}
```

**Aigis の選択**: `min_accumulated_units = 3`, `version_bump_strategy = patch` のみ自動。
minor / major は自動化しない (リリースサイクルが大きい変化は人が判断)。

---

## 自分のプロジェクトには合わない可能性があるケース

Forge Loop が合わないシナリオ:

| シナリオ | 理由 |
|---|---|
| 1 サイクル分の改修に何日もかかる | 6h 起動と相性悪い。stage gate を入れる必要あり |
| 改修対象が外部 API の変動に依存 | research フェーズで外部状態を確実に取れないと spec が揺れる |
| ライセンス上 AI が生成したコードを取り込めない | そもそも導入不可 |
| 100% 自動リリースが必須 | Forge Loop は半自動が設計思想 (人が止められる) |
| ドメインが 1 つしかない | 巡回が機能しない |

---

## 段階的導入のすすめ

いきなり cron + 自動 release はリスク高い。段階的に上げる:

| Phase | 何を自動化 | 人がやる |
|---|---|---|
| 0 | (何も) | 通常開発 |
| 1 | Research のみ | 残り全部 |
| 2 | Research + Plan | Implement 以降 |
| 3 | Implement まで (テスト含む) | CHANGELOG + tag |
| 4 | tag まで全自動 | 配布後の監視のみ |

各 phase を 1〜2 週間運用して、想定通りに動くと分かったら次へ。
**Phase 4 にいきなり飛ぶのは推奨しない**。
