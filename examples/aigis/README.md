# Aigis Example

Forge Loop の **実証実験 1 号機**である Aigis (LLM セキュリティスキャナ) の運用スナップショット。

## このディレクトリの中身

```
examples/aigis/
├── README.md           ← この文書
└── snapshot/           ← Aigis の auto-improvement/ をそのままコピー
    ├── AIGIS-README.md ← Aigis 側の運用ルール定義
    ├── ROTATION.md     ← 10 ドメイン定義 + 現在のカウンタ
    ├── INDEX.md        ← 全 17 サイクルの時系列ログ
    ├── research/       ← サンプル 3 サイクル分のリサーチレポート
    ├── changes/        ← サンプル 3 サイクル分の改修記録
    └── pending/        ← 全 17 件の保留提案 (人レビュー待ち)
```

## 何が見えるか

### ROTATION.md

10 ドメインの定義と現在のカウンタ。`NEXT_INDEX: 7` で次は `evasion-obfuscation` を処理する。

### INDEX.md

5 日間 (2026-05-07 〜 2026-05-11) で 17 サイクル、11 リリース (v1.0.1 → v1.0.12) を時系列で全部記録。
release 列が `—` の cycle は累積しきい値未達で release 見送り、`v1.0.X` の cycle はその場で patch bump。

### research/

3 サイクル分の調査メモ:
- `2026-05-07T14-24_0-prompt-injection.md` — 最初の cycle、prompt-injection ドメイン
- `2026-05-10T18-15_4-memory-context.md` — release につながった cycle (v1.0.11)
- `2026-05-11T06-12_6-multi-agent.md` — 最新 cycle、A2A protocol attacks の取り込み

論文 ID / CVE / 一次ソース URL がすべて記載されている。

### changes/

3 サイクル分の実装記録:
- どのファイルに何 LOC 追加したか
- テストを何ケース追加したか
- pytest 結果 (pass / pre-existing failures / skipped)
- リリース判定の根拠
- 失敗・調整した点 (例: regex の修正過程)

### pending/

**22 件すべて**を収録。Forge Loop の特徴である「保留レーン」の実例。
- `2026-05-08_crescendo-multiturn-detection.md` — 200 LOC 超 & API breaking で保留
- `2026-05-08_agent-spoofing-metadata-validation.md` — メタデータ層の検証が必要で content scanner では実装不可
- `2026-05-09_nist-ai-critical-infra-template.md` — 大規模 compliance template (人が判断)
- ...

## 学び

このスナップショットを見れば、Forge Loop の以下が **実物として確認できる**:

1. **Rotate** — INDEX.md の Domain 列が毎回変わり、10 domain を巡回している
2. **Materialize** — research/ → changes/ → pending/ の各段階に必ずファイルが残っている
3. **Defer** — pending/ に 22 件が "理由付き" で隔離されている (実装されていない)
4. **Accumulating release** — INDEX.md の Release 列で、累積 ≥3 件 cycle だけが release を打っている

## ライセンス・出典

これらのファイルは Aigis プロジェクト (将来公開予定) の `auto-improvement/` から取得したもの。
研究引用先 (論文 / CVE / ベンダー advisory) は各 research ファイル内に記載。
