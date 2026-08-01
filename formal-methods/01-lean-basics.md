# Lean 4 基礎

この章では Lean 4 と Mathlib を使って、型・命題・証明・tactic の関係を学ぶ。
前提は関数型プログラミングの基本と、ターミナル操作である。
数学の定理を暗記する章ではなく、コンパイラに検査されるプログラムとして証明を書く章である。

## 1. インストール

公式の `elan` を使うとプロジェクトごとに Lean の版を固定できる。

```bash
curl https://raw.githubusercontent.com/leanprover/elan/master/elan-init.sh -sSf | sh
elan default stable
lake new fm-basics math
cd fm-basics
lake env lean Main.lean
```

`lake new ... math` は Mathlib を依存に含む雛形を作る。
ネットワークのない CI では、依存を事前にキャッシュし `lake exe cache get` を使う。

```bash
lake update
lake exe cache get
lake build
```

実行時の Lean とエディタ拡張の版がずれると、同じ証明が通らないことがある。
リポジトリに `lean-toolchain` と `lake-manifest.json` を保存する。

## 2. 最小ファイル

```lean
import Mathlib

def double (n : Nat) : Nat := n + n

theorem double_zero : double 0 = 0 := by
  simp [double]

example (n : Nat) : double n = n + n := by
  rfl
```

`def` は計算可能な定義、`theorem` は証明を伴う宣言、`example` は名前を残さない検証である。
証明の本体 `by ...` は、命題に対応する項を tactic で構築する記法である。

## 3. Curry--Howard 対応

| プログラム側 | 論理側 |
| --- | --- |
| 型 | 命題 |
| 値 | 証明 |
| 関数 `A → B` | A ならば B |
| ペア `A × B` | A かつ B |
| `Sum A B` | A または B |
| 空型 `False` | 偽 |

```lean
theorem implication (p q : Prop) : (p → q) → p → q := by
  intro hp h
  exact hp h

theorem and_left (p q : Prop) : p ∧ q → p := by
  intro h
  exact h.1
```

`intro` は含意や全称量化の仮定をコンテキストに追加する。
`exact` は現在のゴールと型が一致する項を置く。

```lean
theorem and_right (p q : Prop) : p ∧ q → q := fun h => h.2

theorem and_build (p q : Prop) : p → q → p ∧ q := by
  intro hp hq
  exact ⟨hp, hq⟩
```

## 4. 命題を読む

```lean
#check Nat
#check (fun n : Nat => n + 1)
#check ∀ n : Nat, n + 0 = n
#check ∃ n : Nat, n = 3
```

`∀ n, P n` は任意の `n` に対する証明を要求する。
`∃ n, P n` は値 `n` と `P n` の証明のペアを要求する。

```lean
theorem exists_three : ∃ n : Nat, n = 3 := by
  exact ⟨3, rfl⟩
```

定義上同じものなら `rfl` が使える。
計算を伴う等式を自動簡約するのが `simp` である。

## 5. 基本 tactic

### `simp`

```lean
theorem add_zero (n : Nat) : n + 0 = n := by
  simp

theorem list_map_id (xs : List α) : xs.map id = xs := by
  simp
```

`simp` は登録された簡約規則を使う。
強力だが、どの規則で変形されたか分からないときは `simp only [...]` で範囲を絞る。

### `rw`

```lean
theorem rewrite_demo (a b c : Nat) (h : a = b) : a + c = b + c := by
  rw [h]
```

`rw [← h]` なら逆方向に書き換える。
「似た式なのに変わらない」場合は、暗黙引数や式の向きを確認する。

### `apply` と `exact`

```lean
theorem trans_demo (a b c : Nat) (hab : a = b) (hbc : b = c) : a = c := by
  apply Eq.trans hab
  exact hbc
```

`apply` はゴールに合う定理を選び、必要な前提を新しいゴールにする。

### `constructor`

```lean
theorem pair_demo (p q : Prop) (hp : p) (hq : q) : p ∧ q := by
  constructor
  · exact hp
  · exact hq
```

### `cases`

```lean
theorem option_cases (x : Option Nat) : x = none ∨ ∃ n, x = some n := by
  cases x with
  | none => exact Or.inl rfl
  | some n => exact Or.inr ⟨n, rfl⟩
```

データ型の全コンストラクタを場合分けするので、網羅性の穴が残りにくい。

## 6. 自然数の帰納法

```lean
theorem add_zero_induction : ∀ n : Nat, n + 0 = n := by
  intro n
  induction n with
  | zero => rfl
  | succ n ih =>
      simp [Nat.succ_add, ih]
```

帰納法では基底ケースと帰納ステップを明示する。
帰納仮定 `ih` が、ステップで使える唯一の過去情報である。

```lean
theorem length_append (xs ys : List α) :
    (xs ++ ys).length = xs.length + ys.length := by
  induction xs with
  | nil => simp
  | cons x xs ih => simp [ih]
```

## 7. よく使う自動化

### `omega`

線形整数・自然数算術なら `omega` が強い。

```lean
import Mathlib

example (x y : Int) (h₁ : x ≤ y) (h₂ : y ≤ x) : x = y := by
  omega

example (n : Nat) : n ≤ n + 3 := by
  omega
```

非線形積や関数の意味までは扱わない。

### `decide`

決定可能な命題を計算で判定する。

```lean
example : (37 : Nat) < 100 := by decide
example : (List.range 5).length = 5 := by decide
```

入力が巨大だとコンパイル時計算が重くなる。

### `ring`

可換半環・環の多項式恒等式を正規化する。

```lean
example (x y : ℤ) : (x + y)^2 = x^2 + 2*x*y + y^2 := by
  ring
```

### `linarith`

線形不等式の前提から結論を導く。

```lean
example (x y : ℚ) (h₁ : x ≤ y) (h₂ : y ≤ x) : x = y := by
  linarith
```

### `norm_num`

数値式を簡約する。

```lean
example : (12345 : ℤ) + 55 = 12400 := by norm_num
```

## 8. 型を使った仕様化

悪い状態を値として表現できないようにする。

```lean
structure Quantity where
  value : ℚ
  nonnegative : 0 ≤ value

def Quantity.add (a b : Quantity) : Quantity :=
  { value := a.value + b.value
    nonnegative := by linarith [a.nonnegative, b.nonnegative] }
```

この型を使う関数は、負の数量を受け取らない。
ただし、`value` を直接公開すると利用者が不変条件を無視して作る設計もあり得るので、API の公開範囲を考える。

## 9. 証明が通らないとき

まずゴールとコンテキストを読む。

```lean
example (n : Nat) : n + 1 = 1 + n := by
  -- ゴール: n + 1 = 1 + n
  -- `simp` だけでは足りない版もある
  simpa [Nat.add_comm]
```

次に、最小の失敗例へ切り出す。

```lean
example (a b : Nat) : a + b = b + a := by
  exact Nat.add_comm a b
```

エラーの分類は次の通り。

| 表示 | 典型原因 | 対処 |
| --- | --- | --- |
| type mismatch | 項の型が違う | `#check` と注釈 |
| unsolved goals | 前提・場合分け不足 | `intro`, `cases`, `induction` |
| unknown constant | import や名前が違う | `#check`、Mathlib 検索 |
| tactic failed | tactic の対象外 | 手動変形・別 tactic |
| maximum recursion | 定義や simp のループ | `simp only`、定義を見直す |

## 10. `sorry` の扱い

```lean
theorem future_proof : 2 + 2 = 4 := by
  sorry
```

`sorry` は仮の証明を受け入れ、警告を出す。
設計中に依存関係を先に進める用途には便利だが、保証ではない。

```bash
lake build 2>&1 | rg "declaration uses 'sorry'"
```

CI では `set_option autoImplicit false` と warning の扱いを厳しくする。
リリース対象は `sorry` の一覧をゼロにする。

## 11. 実行可能な例

```lean
import Mathlib

def factorial : Nat → Nat
  | 0 => 1
  | n + 1 => (n + 1) * factorial n

theorem factorial_two : factorial 2 = 2 := by norm_num [factorial]

def main : IO Unit := do
  IO.println s!"2! = {factorial 2}"
```

`def main` は証明と違って通常のプログラムとしてコンパイルできる。
数学的な定義と IO を分離すると、Lean の定理を実装の中心に置きやすい。

## 12. Mathlib の探し方

```lean
#check List.map
#check List.map_append
#check Nat.add_comm
#check Finset.card
#check Set.mem_inter_iff
```

候補を見つけたら、型を読む。
名前を推測して長い証明を書くより、既存定理の前提を合わせる方が保守しやすい。

```lean
example (xs ys : List Nat) : (xs ++ ys).reverse = ys.reverse ++ xs.reverse := by
  simp [List.reverse_append]
```

## 13. アンチパターン

`simp` を闇雲に繰り返すと、証明の意図が消える。
巨大な一行 tactic は、Mathlib の版変更に弱い。
型キャストを放置すると `Nat`、`Int`、`Rat` の境界で詰まる。
定義を急いで抽象化しすぎると、ゴールが読めない。
まず小さな `example` で定理を試し、それから名前付き定理に昇格させる。

## 14. 次章へのリンク

数学・物理・ゲーム・Web の実例は [02-lean-examples.md](./02-lean-examples.md) にある。
TLA+ の状態モデルとの違いは [03-tlaplus-basics.md](./03-tlaplus-basics.md) を参照する。
PBT との境界は [05-pbt-basics.md](./05-pbt-basics.md)、統合例は [07-combining.md](./07-combining.md) に進む。

## 15. 章末演習

```lean
import Mathlib

theorem list_length_map (f : α → β) (xs : List α) :
    (xs.map f).length = xs.length := by
  induction xs with
  | nil => rfl
  | cons x xs ih => simp [ih]
```

この例で保証したいのは、`map` が要素数を変えないことである。
破れるとページサイズ計算やバッチ処理の上限が壊れる。
`rfl` が使える場所と、帰納仮定が必要な場所を観察する。

## 16. implicit 引数と universe

```lean
import Mathlib

def identity (x : α) : α := x

example (x : Nat) : identity x = x := by rfl
example (x : String) : identity x = x := by rfl
```

`α` は自動暗黙変数である。
教材では便利だが、公開ライブラリでは `set_option autoImplicit false` を検討する。

```lean
set_option autoImplicit false
def explicitIdentity {α : Type} (x : α) : α := x
```

型の曖昧さを放置すると、別の型クラスが選ばれて証明が不安定になる。

## 17. `calc` で意図を書く

```lean
import Mathlib

theorem calc_example (a b c : ℤ) (h : a = b) : a + c = b + c := by
  calc
    a + c = b + c := by rw [h]
```

複数の変形は一行の tactic より `calc` が読みやすい。

```lean
theorem calc_ring (x y : ℚ) : (x + y) * (x - y) = x^2 - y^2 := by
  ring
```

## 18. 名前空間と再利用

```lean
namespace Cart

def total (xs : List Nat) : Nat := xs.sum

theorem total_append (xs ys : List Nat) :
    total (xs ++ ys) = total xs + total ys := by
  simp [total]

end Cart
```

名前空間を使うと、別ドメインの `total` と衝突しにくい。
証明にもドメイン名を付けるとレビューで検索しやすい。

## 19. 依存型の最小例

```lean
import Mathlib

def head? : (xs : List α) → xs ≠ [] → α
  | x :: _, _ => x
  | [], h => (h rfl).elim
```

引数の後ろに証明を取る関数は、不正状態を呼び出し側に要求する。
実務では `Option`、`Nonempty`、構造体のどれが API に合うかを選ぶ。

## 20. 仕上げの運用規約

定理は小さく分け、`sorry` を issue に紐付ける。
自動 tactic と手動補題を混ぜ、失敗箇所が業務概念を示すようにする。
Lean の版と Mathlib の版を固定し、エディタの表示だけを証拠にしない。
