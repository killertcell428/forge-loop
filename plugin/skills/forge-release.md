---
name: forge-release
description: Forge Loop の Release 段階。実装完了した Change を CHANGELOG.md に追記し、累積しきい値を超えていればバージョン bump + git tag する。
---

# forge-release

## 入力

- `changes_file`: `auto-improvement/changes/<UTC>.md` のパス

## やること

### Step 1: CHANGELOG.md への追記

`CHANGELOG.md` の `## [Unreleased]` セクションに、今回完了した各 Change のエントリを追加:

```markdown
## [Unreleased]

- **`<change_id>`** — <1 文要約>。<実装内容の補足>
  - 由来: <Research Finding の Source>
  - LOC: <数値> / Tests: <件数>
```

`<change_id>` は規約に従う (例: Aigis では `sc_langchain_deserialization` のような snake_case)。

### Step 2: 累積数のカウント

`## [Unreleased]` セクションのエントリ数を数える。

- `release_threshold.min_accumulated_units` (既定 3) **以上**なら → リリースへ
- 未満なら → 今回はリリースなし、INDEX.md にその旨を記録して終了

### Step 3: バージョン bump

patch version を 1 上げる:
- `pyproject.toml` / `package.json` / `Cargo.toml` 等の version フィールドを編集
- 該当言語の `__version__` / `VERSION` 定数があれば同期

### Step 4: CHANGELOG.md の Unreleased → リリース版に昇格

```markdown
## [Unreleased]

(empty)

## [vX.Y.Z] - YYYY-MM-DD

- **`<change_id>`** — ...
```

リリースノートには **必ず**:
- 各 Change の rule_id / change_id (検索可能性のため)
- 元になった Research Finding の Source
- 1 つ以上の **具体例** (検出される攻撃の例 / 改善されるユーザ動作の例 等)

### Step 5: git tag

```bash
git add -A
git commit -m "release: vX.Y.Z — forge-loop cycle <cycle_id>"
git tag vX.Y.Z
git push origin <branch> --tags
```

タグ push をトリガーに GitHub Actions が PyPI / npm / Crates.io 等にリリースする (CI 側で設定済みの前提)。

## やってはいけないこと

- **release_threshold を勝手に緩める** (1 件で release してしまう等)
- **CHANGELOG エントリを書かずにバージョンだけ上げる**
- **major / minor の bump を勝手にする** (Forge Loop は patch bump のみ自動化)
- **既存の released version の CHANGELOG を書き換える**

## 失敗時

- CHANGELOG への書き込みに失敗 → 今サイクルはリリース無しで終了 + INDEX.md に記録
- git tag に失敗 → tag が無いまま CHANGELOG だけ更新されている状態。次サイクルの最初に検知して復旧
