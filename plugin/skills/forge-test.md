---
name: forge-test
description: Forge Loop の Test 段階。実装した Change ごとに新規テストを追加し、既存テスト含めて全件 pass を確認する。
---

# forge-test

## 入力

- `changes_file`: `auto-improvement/changes/<UTC>.md` のパス

## やること

1. `changes_file` の「実装する」セクション (実装結果 = 完了) を読む。

2. 各 Change について、対応する新規テストを追加:
   - フレームワークはリポジトリの既存テスト方式に合わせる (pytest / jest / etc.)
   - 1 Change につき最低 1 ケース、推奨 3〜5 ケース
   - 正常系 + エッジケース + 失敗系 を意識

3. テスト実行:
   ```bash
   # リポジトリの test command を実行 (例: pytest, npm test)
   ```

4. 結果を changes_file に追記:
   ```markdown
   ### Change 1
   - **テスト追加**: 4 ケース
   - **pytest 結果**: 197 pass, 0 fail
   ```

5. **既存テストが fail した場合**:
   - その Change を pending に格下げ + 該当のコード変更をロールバック
   - changes_file に記録
   - サイクルは止めない (次の Change に進む)

## 禁止事項

- **既存テストを書き換える** (絶対禁止)
- **テストをスキップ (`@pytest.skip` / `.skip()`) する** (絶対禁止)
- **assert を緩める** (検出範囲を広げる、しきい値を緩める等)
- **テストが通らないまま release 段階に進む**

## 失敗が連鎖した場合

- 全ての Change が pending 格下げになった → CHANGELOG への追記は無し → release 段階で「今回はリリースなし」と判定

これは正常動作。**サイクルが空振り**することは設計上想定されている。
