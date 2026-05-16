# Case Study: Aigis

Forge Loop の **実証実験 1 号機**である Aigis (LLM セキュリティスキャナ) の運用記録。

実データは [`../examples/aigis/snapshot/`](../examples/aigis/snapshot/) に **生のまま**置いている (ROTATION.md / INDEX.md / 全 pending / サンプル research・changes)。
この記事は、その生データを読み解いた「どう運用されているか」の解説。

---

## プロジェクト概要

- **対象**: Python 製の LLM 入出力スキャナ
- **検出対象**: プロンプトインジェクション、データ漏洩、agent tool abuse、A2A protocol attacks 等
- **運用期間**: 2026-05-07 〜 2026-05-11 (5 日間の集中運用)
- **配布**: PyPI (`pip install aigis`)

---

## 数字で見る実績

| 指標 | 値 |
|---|---|
| 累積サイクル数 | **17** |
| 期間 | 5 日間 |
| リリース回数 | **11** (v1.0.1 → v1.0.12, v1.0.5 はスキップ) |
| Pending 累積 | **22 件** (現在も人レビュー待ち) |
| Release を打たなかった cycle | 6 (累積しきい値 ≥3 未達) |

「6 時間ごと」と言いつつ、INDEX.md を見ると間隔は **30 分 〜 1 日** で揺れている。短期集中的に複数 cycle を回した日もある (テスト的に走らせた回も含む)。**規則的な cron だけでなく `workflow_dispatch` でも回せる**運用になっている。

---

## ドメイン定義 (実物)

10 ドメインで運用 (`NEXT_INDEX mod 10`、`auto-improvement/ROTATION.md`):

| id | name | 主たる調査対象 |
|----|------|--|
| 0 | prompt-injection | プロンプトインジェクション最新手法 (直接 / 間接 / multi-modal) |
| 1 | agent-tool-abuse | tool / MCP 濫用、function 偽装、confused deputy |
| 2 | data-exfiltration | データ漏洩 / PII 流出、出力チャネル悪用、URL exfil |
| 3 | jailbreak-extraction | jailbreak / system prompt 抽出、universal adversarial |
| 4 | memory-context | メモリ汚染 / context 操作 / long-context attack / RAG poisoning |
| 5 | supply-chain-llm | LLM サプライチェーン (model/dataset/dep)、typosquatting、weight tampering |
| 6 | multi-agent | A2A protocol attacks、agent card poisoning、session smuggling |
| 7 | evasion-obfuscation | 検知回避 / 敵対的難読化、encoding、homoglyph、policy bypass |
| 8 | compliance-regulation | NIST AI RMF、EU AI Act、ISO/IEC 等の規制動向 |
| 9 | incident-postmortems | 実インシデント / CVE / ベンダーアドバイザリ / ポストモーテム |

ばらつきがあるのは想定通り。**全ドメインで等しいルール数を出すことを目的にしていない**。

---

## サイクルの中身 (1 例)

### Cycle 6 (2026-05-11T06-12 UTC) — multi-agent

**Research** ([`research/2026-05-11T06-12_6-multi-agent.md`](../examples/aigis/snapshot/research/2026-05-11T06-12_6-multi-agent.md)):

調査ソース 7 件:
- Agent Session Smuggling in A2A Systems (Unit42 / Palo Alto, April 2025)
- Agent Card Poisoning (Keysight / LevelBlue SpiderLabs, March 2026)
- Promptware Kill Chain (arxiv:2601.09625, Nassi et al., Jan 2026)
- MASpi benchmark (OpenReview, April 2026)
- Control-Flow Hijacking (arxiv:2510.17276)
- A2A vulnerability survey (jianshiapp.com, 2026 — 91% / 94% vulnerability rates)
- Google A2A Protocol proposals (arxiv:2505.12490)

**Plan + Implement** ([`changes/2026-05-11T06-12_changes.md`](../examples/aigis/snapshot/changes/2026-05-11T06-12_changes.md)):

実装 2 件:
- `_AGENT_CARD_POISONING_PATTERNS` — 3 regex (19 LOC)
- `_SESSION_FABRICATION_PATTERNS` — 3 regex (21 LOC)

Pending 2 件 (継続):
- Agent spoofing via forged `from_agent` field — metadata-layer validation が必要
- Fan-out rate limiting — topology-layer counters が必要

**Test**: `pytest` 1222 pass + 16 pre-existing failures + 4 skipped

**Release**: 累積 5 件 (前回 supply-chain 3 + 今回 multi-agent 2) で **v1.0.12 patch bump**

**所要時間**: 推定 30 分以内

---

## うまくいったこと

### ① 「触らないでいい」ドメインに自然と戻る

人が手で改修していたら、興味のあるドメインに偏る。Forge Loop は機械的に巡回するので、**普段触らないドメインにも順番に必ず手が入る**。
INDEX.md を見ると、5 日間で 10 ドメイン全て少なくとも 1 度は研究 cycle が回っている (`evasion-obfuscation` だけ 2 回、それ以外は概ね均等)。

### ② pending が「未来の自分への手紙」になる

pending/ には現在 22 件溜まっている。代表例:
- **`crescendo-multiturn-detection`** — 多ターン jailbreak (Crescendo) の検出。200 LOC 超 & 公開 API 変更必要で保留
- **`agent-spoofing-metadata-validation`** — content regex では実装不可能、暗号署名が必要
- **`nist-ai-critical-infra-template`** — 大規模 compliance template、人による方針判断が必要

これらは月末にまとめてレビューする想定。普通の issue 管理だと埋もれるが、`pending/` だと必ず目を通す。

### ③ INDEX.md が会話のネタになる

「先月どんなルールが追加された?」「あの v1.0.11 で何が変わった?」が **INDEX.md の 1 行で分かる**。
研究者と話すときの取っ掛かりとして便利。

### ④ pending → 実装の昇格パスがある

`crescendo-multiturn-detection` のように **明確な実装案 + 保留理由**まで書かれていると、後で人が実装するときの工数が下がる。「思いつきメモ」ではなく「保留中の設計書」になっている。

---

## 失敗したこと

### 失敗 1: pending の判定をプロンプトに任せた初期実装

最初は `defer_thresholds.max_loc_per_change = 100` を **プロンプトに書いていただけ**だった。
ある cycle で AI が「110 LOC ですが本質的には 1 つの変更です」と判断して実装し、テストが通ったので tag が打たれた。

→ **修正**: skill 実行終了時に diff を計測し、超過していれば自動 revert。プロンプトではなくスクリプトで強制。

### 失敗 2: research/ なしで implement に直行

ある cycle で外部 API が落ちていて research が空ファイルになった。AI は「知識ベースから」と称して既存知識で 3 件実装した。

→ **修正**: research/<cycle_id>.md に最低 1 つの一次ソース URL が含まれていない場合、implement に進めない検証を入れた。

### 失敗 3: regex の初版が現実と合っていない

cycle 6 では、agent card poisoning の 3 つ目の regex が `route\s+to` を要求していたが、実際の攻撃文は "route requests to" (動詞と目的語の間に "requests" が入る)。
最初のテストで気づき、その場で `(route|send|forward).{0,30}(me|this\s+agent)` に修正 (cycle 内で完結)。

→ **学び**: cycle 内でのテスト失敗→修正のサイクルが回せている。これは Materialize の効能 (changes/*.md に修正過程が記録されているので、後から学び直せる)。

---

## 学び

### ① プロンプトでの強制は弱い

「pending 行きの基準」「最低 LOC」「ソース必須」みたいなルールは、プロンプトに書くだけだと簡単に破られる。**コードで強制**するべき。

### ② Append-only が効く

INDEX.md / research/ / pending/ をすべて **追記式** (削除・上書き禁止) にしたことで、長期間の挙動が見える。失敗 cycle も削除せず残すと、後で「あの時何が悪かったか」が再構成できる。

### ③ 「空振り」を受容する

17 サイクル中 6 回は release を打たなかった (累積 < 3)。これは正常。**毎回必ず release させようとすると、薄い変更が大量生産される**。

### ④ Pending が増え続けても問題ない

5 日で 22 件溜まったが、運用には支障なし。**pending は失敗ではなく "意図的な先送り"**。月単位でまとめて棚卸しすれば十分。

---

## このまま続けるとどうなるか

開きっぱなしの疑問:

- 数ヶ月継続したらルール数が 500 超。検出ロジックの整理が追いつくか?
- 同じドメインに 10 回戻ったら、AI が「もうネタがない」状態になる? (今のところ毎回新しい一次ソースを引いてきている)
- ルール数が増えると false positive が累積する。テストでは捉えきれないユーザ報告の品質低下をどう拾うか?

これらはまだ答えがない。**Forge Loop の長期運用課題として記事の続編で書く予定**。
