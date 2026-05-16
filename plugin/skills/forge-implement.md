---
name: forge-implement
description: Forge Loop の Implement 段階。changes/ の「実装する」リストのみを対象にコード変更を加える。pending は絶対に触らない。
---

# forge-implement

## 入力

- `changes_file`: `auto-improvement/changes/<UTC>.md` のパス

## やること

1. `changes_file` の **「実装する」セクションのみ**を読み込む (「保留」セクションは絶対に開かない)。

2. 各 Change について:
   - 対象ファイルを Read で確認
   - 必要な変更を Edit で実施 (新規ファイル作成は最小限に)
   - 変更後の LOC を計測し、changes_file に追記:
     ```markdown
     ### Change 1
     - **実装結果**: 完了
     - **追加 LOC**: 47
     - **変更ファイル**: aigis/patterns.py
     ```

3. **LOC が見積もりを超過した場合**:
   - 実装途中で `max_loc_per_change` を超えそうなら **そこで停止**し、その Change を pending 行きに格下げする
   - changes_file に降格を記録

## やってはいけないこと

- **pending 行きの提案を実装する** (絶対禁止)
- **「実装する」リストにない変更を追加する** (リスト外の改修を勝手にやらない)
- **既存テストを書き換える** (新規テスト追加のみ)
- **依存関係 (package.json / pyproject.toml) を勝手に変更する** (変更が必要なら pending 行き)
- **環境変数 / .env を変更する**

## 失敗時

- 実装途中で型エラーやインポートエラーが解消できなければ、その Change を **pending に格下げ**してロールバック
- ロールバックが必要なら `git checkout <files>` で該当ファイルを戻す
- changes_file に「実装失敗 → pending 格下げ」を記録
