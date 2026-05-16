---
name: forge-research
description: Forge Loop の Research 段階。指定されたドメインについて外部の最新情報 (論文 / CVE / Issue / リリースノート / ブログ等) を収集し、調査メモを残す。
---

# forge-research

## 入力

- `current_domain`: 今回のドメイン定義 (domains.yaml の 1 エントリ)
- `cycle_id`: サイクル ID (UTC + index + domain name)

## やること

1. 指定ドメインに関する **最新情報**を以下のソースから収集:
   - arXiv (該当キーワードでの新着)
   - GitHub Advisories / CVE Database
   - 主要ベンダのリリースノート
   - ドメイン関連 OSS リポジトリの新規 issue / PR
   - 技術ブログ / カンファレンス資料 (該当する場合)

2. 結果を `auto-improvement/research/<cycle_id>.md` に以下フォーマットで保存:

```markdown
---
domain: <domain_name>
cycle_id: <cycle_id>
run_utc: <UTC>
sources_checked: [arxiv, cve, github_advisories, ...]
---

# Research: <domain_name>

## 今回見つけた新規ネタ

### Finding 1: <短いタイトル>
- **Source**: <URL or 論文 ID>
- **Date**: <発見日 / 公開日>
- **概要**: 2-3 文で
- **このリポジトリで対応可能か**: yes / no / partial
- **実装難度見積もり**: low (≤50 LOC) / med (50-100) / high (>100)

### Finding 2: ...

## 今回スキップした既知ネタ

(既に検出対応済み or 範囲外)
- ...

## 出典リスト

- ...
```

## 出力の品質基準

- **新規性**: 過去 cycle で同じ Source URL を引用していたら必ず指摘する (重複なら本サイクルでは無視)
- **具体性**: 「最近の動向」のような抽象表現は禁止。必ず一次ソースに当たる
- **検証性**: 全ての Finding に検証可能な URL or 論文 ID を付ける

## 過去 cycle との重複検出

`auto-improvement/research/` 配下の既存ファイルを `grep` し、同じ URL / CVE ID / 論文 ID を引用していないか確認する。
重複していたら、その Finding は **削除して別のものを探す**。
