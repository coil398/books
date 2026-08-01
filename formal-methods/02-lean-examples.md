# Lean 4 実践サンプル

この章は Lean 4 と Mathlib で、数学・物理学・ゲーム・Web アプリケーションの性質を表す。
前提は [01-lean-basics.md](./01-lean-basics.md) の `induction`、`simp`、`omega`、`ring` である。
各例は「保証したいこと」「なぜ重要か」「破れたとき」を先に書く。
コードは一つの `Main.lean` に貼って検証できる単位を目指す。

## 1. 数学: 自然数

### 例1: 加法の単位元

保証したいことは、任意の自然数 `n` に対して `n + 0 = n` である。
カウンタの初期化や集計の空集合を扱う基礎性質である。

```lean
import Mathlib

theorem nat_add_zero (n : Nat) : n + 0 = n := by
  induction n with
  | zero => rfl
  | succ n ih => simp [Nat.succ_add, ih]
```

これが破れるなら、空のバッチを追加しただけで数値が変わる。
自然数の加法定義を変えたとき、証明が壊れて検出できる。

### 例2: リストの長さ保存

```lean
import Mathlib

theorem map_length (f : α → β) (xs : List α) :
    (xs.map f).length = xs.length := by
  induction xs with
  | nil => rfl
  | cons x xs ih => simp [ih]

theorem append_length (xs ys : List α) :
    (xs ++ ys).length = xs.length + ys.length := by
  induction xs with
  | nil => simp
  | cons x xs ih => simp [ih]
```

ページネーションや固定長バッファの境界計算で、変換が要素を落とさないことを保証する。

### 例3: 最大値の存在

```lean
import Mathlib

theorem max_le_of_le {a b c : Nat} (ha : a ≤ c) (hb : b ≤ c) :
    max a b ≤ c := by
  exact max_le ha hb

example (xs : List Nat) (h : ∀ x ∈ xs, x ≤ 100) :
    ∀ x ∈ xs, x ≤ 100 := by
  exact h
```

入力制約を関数の前提として明示すると、後段で再検査する必要が減る。

## 2. 数学: 群と順序

### 例4: 群の逆元

```lean
import Mathlib

variable {G : Type*} [Group G]

theorem inv_mul_cancel_custom (x : G) : x⁻¹ * x = 1 := by
  exact inv_mul_cancel₀ x

theorem mul_inv_cancel_custom (x : G) : x * x⁻¹ = 1 := by
  exact mul_inv_cancel x
```

保証は、操作を巻き戻すと単位元になることである。
認証トークンの署名検証を群としてモデル化する場合の代数的核になる。
暗号実装全体の安全性を証明する例ではない。

### 例5: 順序の反射性・推移性

```lean
import Mathlib

theorem le_trans_custom {α : Type*} [Preorder α]
    {a b c : α} (hab : a ≤ b) (hbc : b ≤ c) : a ≤ c := by
  exact le_trans hab hbc

theorem le_refl_custom {α : Type*} [Preorder α] (a : α) : a ≤ a := by
  exact le_rfl
```

価格や権限レベルの比較が推移的であることを確認する。
推移性が壊れると、承認済みのはずの操作が順序によって拒否される。

### 例6: 初等整数論、合同

```lean
import Mathlib

example : (17 : ZMod 5) = 2 := by decide

theorem congruence_add (a b : ZMod 7) : a + b = b + a := by
  exact add_comm a b
```

`ZMod n` は剰余算術を型で表す。
暗号やハッシュの補助計算で、整数と剰余の混同を減らせる。

## 3. 物理学: 次元解析

物理量の単位を型に埋め込むと、距離と時間を加算する誤りを型エラーにできる。
以下は教育用の最小モデルで、SI 単位系全体を実装したものではない。

### 例7: 距離と時間の分離

```lean
import Mathlib

structure Meter where value : ℚ deriving DecidableEq
structure Second where value : ℚ deriving DecidableEq

def Meter.add (a b : Meter) : Meter := ⟨a.value + b.value⟩
def Meter.scale (k : ℚ) (a : Meter) : Meter := ⟨k * a.value⟩
def velocity (d : Meter) (t : Second) (h : t.value ≠ 0) : ℚ :=
  d.value / t.value

example (d : Meter) : Meter.scale 1 d = d := by
  cases d
  simp [Meter.scale]
```

保証したいのは `Meter` と `Second` が別型であることだ。
単位を取り違えると、シミュレーションの時間刻みが距離として扱われる。

### 例8: 等加速度運動の離散式

```lean
import Mathlib

def nextPosition (x v a dt : ℚ) : ℚ := x + v * dt + (a * dt^2) / 2

theorem nextPosition_zero_dt (x v a : ℚ) :
    nextPosition x v a 0 = x := by
  simp [nextPosition]

theorem nextPosition_affine (x v a dt : ℚ) :
    nextPosition x v a dt - x = v * dt + (a * dt^2) / 2 := by
  ring
```

時間刻み 0 で位置が変わらないことは、積分器の境界条件である。
破れると、停止したシミュレーションでも状態が動く。

### 例9: エネルギー保存の代数形

```lean
import Mathlib

def energy (m v g h : ℚ) : ℚ := m * v^2 / 2 + m * g * h

theorem energy_swap (m v g h : ℚ) :
    energy m v g h = m * (v^2 / 2 + g * h) := by
  ring
```

これは物理法則そのものではなく、定義した式の代数的同値性である。
実際の保存には運動方程式、境界、誤差モデルを追加しなければならない。

## 4. ゲーム: 勝敗とスコア

### 例10: 勝敗判定の全域性

```lean
import Mathlib

inductive Result where
  | win | lose | draw
deriving DecidableEq, Repr

def result (a b : Int) : Result :=
  if a > b then .win else if a < b then .lose else .draw

theorem result_cases (a b : Int) :
    result a b = .win ∨ result a b = .lose ∨ result a b = .draw := by
  unfold result
  split <;> simp_all
```

全域性を保証すると、UI やリプレイ処理が未知の勝敗値で落ちない。
判定条件の順番を変えて同点を勝ち扱いにすると、この証明やテストが意図を露呈させる。

### 例11: スコア計算の単調性

```lean
import Mathlib

def score (base bonus : Nat) : Nat := base + bonus

theorem score_mono_bonus (base b₁ b₂ : Nat) (h : b₁ ≤ b₂) :
    score base b₁ ≤ score base b₂ := by
  exact Nat.add_le_add_left h base

theorem score_zero (base : Nat) : score base 0 = base := by
  simp [score]
```

ボーナスを増やしたのにスコアが下がらないことを保証する。
ランキングの逆転や報酬の不正減算を防ぐ中核性質である。

### 例12: 盤面遷移の不変量

```lean
import Mathlib

structure Board where
  pieces : Fin 9 → Bool

def place (b : Board) (i : Fin 9) : Board :=
  { b with pieces := fun j => if j = i then true else b.pieces j }

theorem place_true (b : Board) (i : Fin 9) :
    (place b i).pieces i = true := by
  simp [place]
```

合法な置き手が指定マスを占有済みにすることを保証する。
`i` を無検証の整数にすると盤外アクセスが起こるため、`Fin 9` を使っている。

## 5. Web: アクセス制御

### 例13: 認可述語の健全性

```lean
import Mathlib

inductive Role | guest | member | admin deriving DecidableEq
inductive Action | read | write | delete deriving DecidableEq

def allowed : Role → Action → Bool
  | .guest, .read => true
  | .member, .read => true
  | .member, .write => true
  | .admin, _ => true
  | _, _ => false

theorem guest_cannot_delete : allowed .guest .delete = false := by decide
theorem admin_can_delete : allowed .admin .delete = true := by decide
```

保証したいのは、公開したくない組合せが明示的に拒否されることだ。
逆向きのデフォルト許可は権限昇格につながる。

### 例14: ページネーションの非重複

```lean
import Mathlib

def page (xs : List α) (size offset : Nat) : List α :=
  (xs.drop offset).take size

theorem page_empty_size (xs : List α) (offset : Nat) :
    page xs 0 offset = [] := by
  simp [page]

theorem page_length_le (xs : List α) (size offset : Nat) :
    (page xs size offset).length ≤ size := by
  exact List.length_take_le _ _
```

ページサイズ上限と空ページの性質を保証する。
実際のページ間非重複は offset の関係を仮定して別定理にする。

### 例15: 状態遷移の到達可能性

```lean
import Mathlib

inductive SessionState | anonymous | authenticated | revoked deriving DecidableEq

def canRefresh : SessionState → Bool
  | .authenticated => true
  | _ => false

theorem revoked_cannot_refresh : canRefresh .revoked = false := by decide

theorem auth_refresh : canRefresh .authenticated = true := by decide
```

失効後に更新トークンを使えないことを保証する。
実際の JWT の署名、時計、鍵ローテーションは別の境界でテストする。

## 6. 証明設計の実務

最初に不変条件を自然言語で書き、次に型または述語に落とす。
定理名は「何がいつ成立するか」を表す。
証明が長いときは、補題の境界が設計の境界になる。

```lean
import Mathlib

def nondecreasing (xs : List Nat) : Prop :=
  ∀ i j : Fin xs.length, i ≤ j → xs.get i ≤ xs.get j

theorem empty_nondecreasing : nondecreasing [] := by
  intro i
  exact Fin.elim0 i
```

上の定義は教育用であり、Mathlib の `List.Pairwise` の方が実務的なことが多い。
既存抽象を使うか、自分の証明しやすい形を選ぶかはトレードオフである。

## 7. 破綻パターン

`Bool` を `Prop` の代わりに使うと、証明可能な意味が薄くなることがある。
`Nat` で負数を表現しようとして underflow の意味を誤ることがある。
物理量を全部 `ℚ` にすると、単位や連続時間を失う。
Web の認可を純粋関数だけで証明すると、キャッシュやミドルウェアの抜けが残る。
ゲームの盤面を可変配列で扱うと、遷移の不変量が見えにくくなる。

## 8. 実行方法

```bash
lake new lean-examples math
cd lean-examples
cp /path/to/Main.lean Main.lean
lake env lean Main.lean
```

コード片を一つずつ `example` にして衝突する名前を避ける。
`import Mathlib` は教材では便利だが、製品では必要な import を絞るとビルドが軽くなる。

## 9. まとめ

数学では帰納法と代数、物理では単位と保存量、ゲームでは全域性と不変量、Web では認可と遷移を示した。
どの領域でも、形式化の価値は「正しそう」を「この前提なら必ず」に変える点にある。
次は [03-tlaplus-basics.md](./03-tlaplus-basics.md) で時間と並行性を扱う。

## 10. 例を設計へ戻す

| 例の結果 | 設計上の問い |
| --- | --- |
| `omega` が通らない | 値域を `Nat` と `Int` のどちらにするか |
| `ring` が通らない | 物理量の単位を式に含めるか |
| `cases` が増える | 状態を inductive にするか |
| 前提が多い | 不正状態を構造体で隠せるか |
| 移植の property が失敗 | 本番実装と定義が同じか |

証明の難しさは、数学の難しさだけでなく API の設計を反映する。

## 11. 実例を動かす順序

```text
import を通す
定義だけを #check する
数値の example を decide で確認する
一般化して theorem にする
一行壊して失敗位置を読む
```

この順序を守ると、ドメインの誤りと Lean の構文誤りを分離できる。

## 12. 章のまとめ

数学の帰納法、物理の単位、ゲームの全域性、Web の認可を同じ型システムで表した。
証明できないものを無理に証明せず、境界を PBT や TLA+ へ渡すことが実務的な設計である。

## 13. 実装境界の記録

```text
Lean が保証: 純粋関数、型、算術、帰納的データ
Lean が未保証: DB、時計、ネットワーク、手書き移植
TLA+ が保証: 抽象状態、競合順序、保存不変量
PBT が保証: 実装の観測された入力・操作列
```

この境界をコードレビューの冒頭に置く。
形式化されていないものを「暗黙に正しい」と扱わない。

## 14. 章末の確認

- [ ] 各定理に前提がある
- [ ] `Nat` の underflow を意識した
- [ ] 単位を値の名前だけに頼っていない
- [ ] ゲームの状態遷移が全コンストラクタを覆う
- [ ] Web の default deny を確認した
- [ ] 実装との接続を PBT または統合テストに残した

## 15. 次へ

証明の全域性と実装の不確実性を分けたら、状態の順序を [03-tlaplus-basics.md](./03-tlaplus-basics.md) で扱う。

実例の共通点は、型・述語・補題で境界を明示することである。

証明の前提もプロダクト仕様の一部としてレビューする。

不変条件の名前は、ドメイン語彙で付ける。

これによりコードと説明が接続する。

次章では時間方向を導入する。
