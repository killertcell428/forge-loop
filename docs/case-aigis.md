# Case Study: Aigis

Forge Loop の **実証実験 1 号機**である [Aigis](https://github.com/...) (LLM セキュリティスキャナ) の運用記録。

---

## プロジェクト概要

- **対象**: Python 製の LLM 入出力スキャナ
- **検出対象**: プロンプトインジェクション、データ流出、agent tool abuse 等
- **運用開始**: 2026-05 (継続中)
- **配布**: PyPI

---

## 数字で見る実績

| 指標 | 値 |
|---|---|
| 累積サイクル数 | 27 |
| 期間 | 約 5 ヶ月 (実質運用 3 ヶ月) |
| リリース回数 | 11 (v1.0.7 → v1.0.17) |
| 追加検出ルール | 137 |
| pending に積まれた提案 | 9 |
| 平均サイクル所要時間 | 約 40 分 |
| 全自動でリリースされた % | 約 60% |
| 人がレビューして修正した % | 約 40% |
| 完全に空振りしたサイクル | 4 (≈15%) |

---

## ドメイン定義

10 ドメインで運用 (`NEXT_INDEX mod 10`):

| id | name | 累計対応ルール数 |
|----|------|--|
| 0 | prompt-injection | 21 |
| 1 | agent-tool-abuse | 14 |
| 2 | data-exfiltration | 12 |
| 3 | jailbreak-techniques | 18 |
| 4 | supply-chain-attacks | 11 |
| 5 | model-stealing | 6 |
| 6 | pii-leakage | 15 |
| 7 | adversarial-suffixes | 9 |
| 8 | rag-poisoning | 13 |
| 9 | incident-postmortems | 18 |

ばらつきがあるのは想定通り。**全ドメインで等しいルール数を出すことを目的にしていない**。
プロンプトインジェクションは論文・PoC が豊富で実装余地が大きい、というだけ。

---

## サイクルの中身 (1 例)

### Cycle 13 (2026-05-13 03:00 UTC)

**Domain**: supply-chain-attacks

**Research** (`research/2026-05-13T03-00_06-supply-chain.md`):
> 今回見つけた新規ネタ
>
> 1. LangChain CVE-2025-68664 — `pickle.loads` で deserialization 経由 RCE
> 2. PyTorch 2.5.x — `torch.load` の `weights_only=False` を悪用した RCE
> 3. Hydra (Facebook) — config injection 経由の任意 Python 実行
> 4. uv (Astral) — lockfile 改竄を介した依存 hijack (実装範囲外、別 cycle へ)

**Plan** (`changes/2026-05-13T03-00.md`):
> 実装する (3 件):
> - sc_langchain_deserialization (≈40 LOC)
> - sc_pytorch_weights_only_bypass (≈45 LOC)
> - sc_hydra_config_injection (≈50 LOC)
>
> 保留:
> - sc_uv_lockfile_tampering — 依存スキャナ全体の再設計が必要、>200 LOC

**Implement**: aigis/patterns.py に regex 3 件、合計 +147 LOC

**Test**: tests/test_supply_chain.py に 12 ケース追加、全 pass

**Release**: 累積 5 件で v1.0.13 として tag。

**Pending**: pending/2026-05-13_uv-lockfile-scanner.md を作成 (現在も眠っている)

---

## うまくいったこと

### ① 「触らないでいい」ドメインに自然と戻る

人が手で改修していたら、興味のあるドメインに偏る。Forge Loop は機械的に巡回するので、**普段触らないドメインにも 6h ごとに必ず手が入る**。

例: `model-stealing` ドメインは私が個人的に詳しくないが、6 サイクル分の知見が累積された。

### ② pending が「未来の自分への手紙」になる

pending/ には「やりたいけど大きすぎる」案が 9 件溜まっている。
これを月末にまとめてレビューすると、**「あの時の AI が見つけた興味深いネタ」**が一覧で見える。
普通の issue 管理だと埋もれるが、pending/ なら必ず目を通す。

### ③ INDEX.md が会話のネタになる

「先月どんなルールが追加された?」「あの v1.0.11 で何が変わった?」が INDEX.md の 1 行で分かる。
研究者と話すときの取っ掛かりとして便利。

---

## 失敗したこと

### 失敗 1: 同じドメインが 5 連続

最初の実装では `ROTATION.md` の `NEXT_INDEX` をエージェントが書き換える設計にしていた。
ある時、エージェントが書き換えに失敗して NEXT_INDEX が 0 のままになり、5 サイクル連続で `prompt-injection` ドメインが処理された。

→ **修正**: NEXT_INDEX 更新を最後の決定論的ステップに分離。エージェントの実装フェーズの成功/失敗とは独立に更新される。

### 失敗 2: 200 LOC PR が勝手にマージ

`defer_thresholds.max_loc_per_change = 100` を **プロンプトに書いていただけ**だった。
ある cycle で AI が「110 LOC ですが本質的には 1 つの変更です」と判断して実装し、テストが通ったので tag が打たれた。

→ **修正**: skill 実行終了時に diff を計測し、超過していれば自動 revert。プロンプトではなくスクリプトで強制。

### 失敗 3: research/ なしで implement に直行

ある cycle で arxiv の API が落ちていて research が空ファイルになった。AI は「知識ベースから」と称して既存知識で 3 件実装した。

→ **修正**: research/<cycle_id>.md に最低 1 つの一次ソース URL が含まれていない場合、implement に進めない検証を入れた。

---

## 学び

### ① プロンプトでの強制は弱い

「pending 行きの基準」「最低 LOC」「ソース必須」みたいなルールは、プロンプトに書くだけだと簡単に破られる。**コードで強制**するべき。

### ② Append-only が効く

INDEX.md / research/ / pending/ をすべて **追記式** (削除・上書き禁止) にしたことで、長期間の挙動が見える。失敗 cycle も削除せず残すと、後で「あの時何が悪かったか」が再構成できる。

### ③ 「空振り」を受容する

27 サイクル中 4 回は完全に空振り (research のみ、実装ゼロ)。これは正常。**毎回必ず実装させようとすると、薄い変更が大量生産される**。

### ④ ローテーション順は重要じゃない

最初は「興味の薄いドメインから始める」と工夫したが、長期的にはあまり意味がなかった。**結局すべて巡回するので、初期順序は気にしなくていい**。

### ⑤ 6h は丁度いい

3h は早すぎる (人が見る前に次が来る)。12h は遅い (1 日 2 サイクル)。6h ≒ 1 日 4 サイクルが現状最適。

---

## このまま続けるとどうなるか

開きっぱなしの疑問:

- 1 年継続したら 137 ルール → 500 ルール超。検出ロジックの整理が追いつくか?
- 同じドメインに 10 回戻ったら、AI が「もうネタがない」状態になる?
- ルール数が増えると false positive が累積する。テストでは捉えきれないユーザ報告の品質低下をどう拾うか?

これらはまだ答えがない。**Forge Loop の長期運用課題として記事の続編で書く予定**。
