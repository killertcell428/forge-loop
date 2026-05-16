# Aigis Example

Forge Loop の **実証実験 1 号機**である Aigis (LLM セキュリティスキャナ) のサンプル。

## 実物の場所

実運用中のリポジトリ:
- https://github.com/.../aigis (TBD)

実運用中の `auto-improvement/` ディレクトリ:
- `aigis/.claude/worktrees/.../auto-improvement/`

## このディレクトリの目的

Aigis の `auto-improvement/` を **sanitize したスナップショット**を、後でここに置く予定。
今は README のみ。

## 詳細

Aigis での運用ログ詳細は [../../docs/case-aigis.md](../../docs/case-aigis.md) を参照。

## やる予定の作業 (TODO)

- [ ] Aigis から ROTATION.md / domains.yaml / INDEX.md (匿名化版) をコピー
- [ ] 直近 3 サイクル分の research/ / changes/ をサンプルとして置く
- [ ] pending/ の代表例 3 件をサンプルとして置く
- [ ] 全てのファイルから個人 / 組織を特定できる情報を除去
