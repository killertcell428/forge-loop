---
name: forge-plan
description: Forge Loop の Plan 段階。research/ の Finding を実装案に変換し、pending 行きかどうかを判定する。
---

# forge-plan

## 入力

- `research_file`: `auto-improvement/research/<cycle_id>.md` のパス

## やること

1. `research_file` を読み込み、各 Finding ごとに以下を判定:
   - **実装可能**か？ (Finding の「対応可能か」フィールド)
   - **pending 行き**条件に当たるか？

2. pending 行きの判定基準 (plugin.json の defer_thresholds と一致):
   - LOC 見込み > `max_loc_per_change` (既定 100)
   - API breaking change を含む
   - 既存テストを書き換える必要がある
   - 確信度 "low" 以下

3. 結果を `auto-improvement/changes/<UTC>.md` に保存:

```markdown
---
cycle_id: <cycle_id>
run_utc: <UTC>
---

# Changes for cycle <cycle_id>

## 実装する (3 件)

### Change 1
- **由来**: Finding 1 (research/<cycle_id>.md)
- **概要**: <1 文>
- **対象ファイル**: <path>
- **LOC 見込み**: <数値>
- **テスト追加**: <件数>

### Change 2
...

## 保留 (pending 行き)

### Pending 1
- **由来**: Finding 4
- **概要**: <1 文>
- **理由**: <pending 行きの根拠 (LOC / breaking / etc.)>
- **隔離先**: pending/<UTC>_<short-name>.md
```

4. pending 行きの提案ごとに `auto-improvement/pending/<UTC>_<short-name>.md` を作成:

```markdown
---
status: pending
created_utc: <UTC>
cycle_id: <cycle_id>
reason: large_change | api_breaking | low_confidence | test_rewrite
---

# Pending: <タイトル>

## 提案の中身
...

## 実装しない理由
...

## 人が判断すべきこと
- [ ] このまま実装すべきか / 諦めるか
- [ ] 実装するなら、分割案は何か
- [ ] 期限を切るか / 永久保留か
```

## 禁止事項

- pending 行きの基準を勝手に緩めること
- 「やれそう」「いけそう」という主観で pending を回避すること
- 既存テストの fail を「無視できる」と判断して実装すること
