# LLM にツールを実行させるかどうかが分水嶺

GPT-5.6 系 3 モデル × 3 effort = 9 セルで同一の技術書をレビューさせ、処理系（TLA+ の SANY / TLC）が機械的に確定させた欠陥をどれだけ検出できるかを測った記録。

実施日: 2026-08-01

> ℹ️ 対象の教科書は [`../formal-methods/`](../formal-methods/)。本レポートは教科書の内容ではなく**レビュー手法**についての記録。

---

## 結論

1. **9 セル中 8 セルが、SANY が 1 秒で見つける構文・意味エラーを 1 件も検出しなかった。**
2. **唯一の満点（4/4）を取った `sol@max` は、read-only サンドボックスの中で実際に SANY / TLC / PlusCal Translator を実行していた。** 残り 8 セルは静的にコードを読んでいただけ。
3. したがって差を作ったのは推論の深さではなく **「ツールを実行したか」**。実行したセルは処理系と同じ結論に到達し、**さらに処理系だけでは届かない層（意味論・設計・本文との整合）も同時に拾った**。
4. **effort は「見る深さ」を変えるが「見る層」は model が決める。** ただし `sol` だけは max で質的に振る舞いが変わった。

---

## 結果

| | medium | high | max |
|---|---|---|---|
| **luna** | 0/4 — 8件 / 2.8分 | 0/4 — 7件 / 9.9分 | 0/4 — 13件 / 24.8分 |
| **terra** | 0/4 — 4件 / 3.8分 | 0/4 — 9件 / 7.6分 | 0/4 — 17件 / 29.8分 |
| **sol** | 1/4 — 18件 / 9.9分 | 1/4 — 18件 / 15.6分 | **4/4 — 37件 / 48.6分** |

スコアは後述の「正解セット」4 件の検出数。件数は各セルの自己申告 `SUMMARY` の合計。

---

## 実験設計

### 統制した条件

| 項目 | 値 |
|---|---|
| レビュープロンプト | 全 9 セル同一（MD5 `41c7440528cd3e714d94d18f6d3af5d1` を起動後に検証） |
| 対象 | `formal-methods/` の HTML 9 ファイル / 7,786 行 |
| sandbox | `read-only` |
| CWD | 本リポジトリ |
| 実行 | 9 セル同時起動（同一時刻・同一マシン） |
| **変数** | **`model` と `effort` のみ** |

model は `gpt-5.6-luna` / `gpt-5.6-terra` / `gpt-5.6-sol`、effort は `medium` / `high` / `max`。`max` は 3 モデル共通で対応する最上位（`ultra` は luna 非対応のため除外）。

### 出力スキーマ

比較可能にするため 1 件 1 ブロックの固定形式を強制し、末尾に `SUMMARY: 動かない=N 誤り=N 誤解を招く=N 些細=N / 読んだファイル数=N` を必須とした。

```
### 指摘 N
- ファイル:
- 箇所:
- 深刻度: <動かない | 誤り | 誤解を招く | 些細>
- 根拠:
- 修正案:
```

**プロンプトには既知の欠陥を一切書いていない。** チェックリスト項目にも `EXTENDS` 等のヒントを混ぜていない。

---

## 正解セットの作り方

**総指摘数は品質指標にしない。** 前段の実験で、56 件を挙げたセルの修正結果が 3 モジュールでパース不能のまま残った。件数と正しさは相関しない。

代わりに **処理系が機械的に判定した欠陥**を正解セットにした。

### 手順

1. HTML から `<pre>` ブロックを抽出し、HTML エンティティをデコード
2. `---- MODULE X ----` を含むブロックを `X.tla` として書き出す（20 個）
3. 各ファイルを `tla2sany.SANY` にかける → **19 通過 / 3 FAIL**
4. `.cfg` は `tlc2.TLC -config` で実際に読ませる → **1 FAIL**

使用ツールの実測値:

```
TLC2 Version 2.19 of 08 August 2024 (rev: 5a47802)
Implementation-Version: 2.0 2024-08-08
```

> ⚠️ `releases/latest/download/tla2tools.jar` は固定リリースではなく、master のビルドが同じ URL へ継続的に上書きされる pre-release である。再現時は上記 rev を確認すること。この点は `sol@high` が指摘した。

### 確定した欠陥（正解セット）

| # | 欠陥 | 確定方法 | 検出したセル |
|---|---|---|---|
| 1 | `DiscreteSum`: `i * (i + 1) \div 2` が `*` と `\div` の優先順位衝突で Parse Error | SANY | sol@max のみ |
| 2 | `Match`: `EXTENDS FiniteSets` のみで `Cardinality(waiting) < 2` の `<` が未定義 | SANY | sol@max のみ |
| 3 | `RoundRobin`: 同じく `<=` が未定義 | SANY | sol@max のみ |
| 4 | `Trade.cfg`: record constructor のフィールド名に文字列リテラルを使い ConfigFileException | TLC | sol@medium, sol@high, sol@max |

2 と 3 は **`Cardinality` → `FiniteSets` までは 9 セル中複数が辿り着いたのに、`Cardinality(x) < 2` の比較演算子も `Naturals` 由来だという二段目に 8 セルが到達しなかった**という、系統的な盲点になっている。

---

## sol@max だけが満点だった理由

根拠の書き方が他の 8 セルと決定的に違う。

| 指摘 | 根拠の文言 |
|---|---|
| 20 | 「SANY が `Precedence conflict between ops * and \div` として**パースを中止します**」 |
| 16 | 「SANY は `Could not find declaration or definition of symbol '<'` を**報告します**」 |
| 13 | 「`-deadlock` を付けると掲載された 3 不変条件は**成功しました**」 |
| 14 | 「PlusCal Translator 1.12 は `Expected "=" or "\in" but found "P"` で**停止しました**」 |
| 24 | 「fast-check **4.9.0** の `fc.array(fc.integer())` は既定最大長が 10」 |

**過去形の実測報告になっている。** `read-only` サンドボックスの中でツールを走らせて確かめた結果であり、他の 8 セルの「コードを読んだ推論」とは種類が違う。

さらに sol@max は、処理系検証では届かない Lean の欠陥も同時に挙げた。

| 内容 |
|---|
| `import Mathlib` すると `CategoryTheory.Action` があるので `inductive Action` は `'Action' has already been declared` |
| `.nonnegative` は命題でなく証明項。定理宣言のコロン後に置くと `type expected` |
| `simp [ih]` では `xs.length + ys.length + 1 = xs.length + 1 + ys.length` の並べ替えが解けない |
| `ring` は定義を展開しないので `unfold` が要る |
| `split` は外側の `if` しか場合分けしないため偽分岐にゴールが残る |

---

## model が「見る層」を決めている

各モデルは 3 effort とも一貫して同じ層を見ていた。

| model | 一貫して見ていた層 | 代表的な指摘 |
|---|---|---|
| **luna** | 設計・意味論 | 「`Disconnect(turn)` が可能で `turn` は `UNCHANGED` なので切断済みプレイヤーに手番が残る」「`every` は結果側しか見ないので要素を削除する実装が通る」 |
| **terra** | Lean のツールチェーン・型検査 | 「`Nat.mul_le_mul_left` が返すのは `a*r ≤ a*100`、補題が要求するのは `a*r ≤ 100*a`。可換性は定義的等式ではない」「`lake new ... math` は Mathlib master の toolchain を落とす」 |
| **sol** | 処理系の実行時挙動 | 「状態空間が無限で TLC の全探索が終わらない」「終端に後続がなく deadlock エラーになる」「`.cfg` の record 構文がパースできない」 |

### effort の効き方

terra 内で比較すると単調に増える。

| | 所要 | 指摘 | 動かない |
|---|---|---|---|
| terra@medium | 3.8分 | 4 | 2 |
| terra@high | 7.6分 | 9 | 4 |
| terra@max | 29.8分 | 17 | 12 |

一方 SANY スコアは 0/4 のまま動かなかった。**量は増えるが層は変わらない。**

luna では逆転すら起きた。

| | 所要 | 動かない |
|---|---|---|
| luna@medium | 2.8分 | 2 |
| luna@high | 9.9分 | **0** |

3.5 倍の時間をかけて「動かない」の指摘がゼロになり、`SUMMARY: 動かない=0` と自己申告した。実際には 3 モジュールがパース不能だった。

`sol` だけが max で質的に変わった（ツールを実行し始めた）。**effort を上げれば必ずそうなるわけではない**ことは、luna@max と terra@max がどちらも 0/4 だったことが示している。

---

## 複数セル一致は信頼度の指標になる

| 指摘 | 一致セル数 |
|---|---|
| Lean `Nat.div_le_of_le_mul` の乗算順序による型不一致 | **5**（terra 全 3 + sol@high + luna@max） |
| `app` / `door.ts` / `collection.ts` の実装欠落 | 5 |
| `fc.assert` が vitest の `test()` に未登録 | 4 |
| `.cfg` の record constructor パースエラー | 3（sol 全 3） |

3 モデルが独立に同じ結論へ達した指摘は、処理系で未検証でも採用してよい水準にある。実際 Lean の型不一致は `elan` 未導入で検証できなかったが、5 セル一致を根拠に修正した。

---

## LLM が処理系検証の穴を 3 回指摘し返した

この実験で最も示唆的な点。**検証スクリプトを書いたのは人間（＋Claude）側だが、その穴を LLM レビューが埋めた。**

| 指摘した側 | 内容 | 処理系検証側の穴 |
|---|---|---|
| sol@medium, sol@high | `.cfg` の record constructor がパースできない | SANY は `.tla` しか見ていなかった |
| luna@max | `MODULE` 宣言のないブロックが 8 個ある | 抽出フィルタが `MODULE` 正規表現で弾いていた |
| sol@high | 使用中の jar が内容の変わる pre-release | ツールのバージョン固定を怠っていた |

**片方だけでは必ず穴が残る。**

---

## 運用への示唆

1. **処理系で落とせるものを先に落とす。** SANY は 1 秒、9 セルの LLM レビューは合計 2 時間半以上。同じ 3 件を前者は全部、後者は（sol@max を除き）ゼロ。順序を逆にすると高くつく
2. **LLM に「ツールを実行してよい」と明示的に許可し、実行を促す。** 今回のプロンプトは実行を禁じても要求もしていなかった。sol@max が自発的に実行したのは偶然の側面がある
3. **並列度は 3 程度に抑える。** 9 並列にした結果、sol@max の 1 回目が 60 分でハーネスに kill された（他の max セルは 24〜30 分で完走）。単独・デタッチで再実行して 48 分で完走した
4. **複数モデルを競わせるより、層の違いを利用して組み合わせる。** luna（設計）・terra（Lean）・sol（処理系）は競合ではなく補完だった

---

## 限界

- **各セル N=1。** 同じ構成でもう一度回せば違う数字が出る
- 正解セットは TLA+ に偏っている。Lean は `elan` 未導入で処理系検証ができず、Python / TypeScript も機械的な正解セットを作れなかった
- 対象は「一度修正を経た」状態の教科書。修正前を対象にした前段の実験とは指摘数を直接比較できない
- 深刻度の判定は各モデルの自己申告
- sol@max のみ実行条件が違う（9 並列で 60 分 kill → 単独デタッチで再実行）。**負荷条件が他セルと揃っていない**
- 「ツールを実行したか」は各モデルの報告文言から推定したもので、実行ログを直接取得したわけではない

---

## 再現手順

```sh
# 1. TLA+ モジュールを抽出して SANY にかける
#    <pre> ブロック → HTML エンティティをデコード → MODULE 宣言を含むものを .tla へ
java -cp tla2tools.jar tla2sany.SANY Match.tla

# 2. .cfg を TLC に読ませる
java -cp tla2tools.jar tlc2.TLC -config Trade.cfg Trade.tla

# 3. 各セルを起動（変数は model/effort のみ）
cat review-prompt.md | codex exec --json --skip-git-repo-check \
  -m "$MODEL" -c model_reasoning_effort="$EFFORT" \
  -s read-only -C "$BOOKS" \
  -o "$OUT_LAST" "" > "$OUT_EVENTS"
```

長時間ジョブは `run_in_background` の 60 分制限に当たるため、`python3 -c "import os; os.setsid()"` でプロセスグループを切り離してから実行し、foreground のポーリングで完了を待つ。
