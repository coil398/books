# Lean・TLA+・PBT を組み合わせる

この章では、在庫を持つ EC の購入フローを一つの題材にする。
TLA+ で並行する注文の状態遷移を検査し、PBT で実装を揺さぶり、Lean で金額と在庫の中核ロジックを証明する。
前提は [00-overview.md](./00-overview.md)、各ツールの基礎章である。

## 1. 題材と保証

購入 API は、在庫、注文、決済、再試行、返金を扱う。

| 層 | 保証したい性質 | 技術 |
| --- | --- | --- |
| プロトコル | 在庫は負にならない | TLA+ |
| プロトコル | 同じ idempotency key は一度だけ効く | TLA+ / PBT |
| 実装境界 | JSON の往復と入力検証 | PBT |
| 金額計算 | 明細合計と割引の恒等式 | Lean / PBT |
| 運用 | 外部決済の遅延、タイムアウト | PBT の fake |

「すべてを一つの道具で」は狙わない。

## 2. TLA+ で設計する

まず利用者二人、在庫一個、注文キー二つに縮約する。

```tla
---- MODULE Shop ----
CONSTANT Users, Keys, InitialStock
VARIABLE stock, orders, applied

Init == /\\ stock = InitialStock
        /\\ orders = [u \in Users |-> "none"]
        /\\ applied = {}

Purchase(u, k) == /\\ u \in Users /\\ k \in Keys
                  /\\ orders[u] = "none"
                  /\\ k \notin applied
                  /\\ stock > 0
                  /\\ stock' = stock - 1
                  /\\ orders' = [orders EXCEPT ![u] = "paid"]
                  /\\ applied' = applied \cup {k}

Retry(k) == /\\ k \in applied
           /\\ UNCHANGED <<stock, orders, applied>>

Next == \\E u \in Users, k \in Keys : Purchase(u,k) \\/ Retry(k)
TypeOK == /\\ stock \in Nat
          /\\ orders \in [Users -> {"none", "paid"}]
          /\\ applied \subseteq Keys
Safety == stock >= 0
AtMostOnce == Cardinality({u \in Users : orders[u] = "paid"})
             <= InitialStock
Spec == Init /\\ [][Next]_<<stock, orders, applied>>
================================================
```

## 3. TLC のモデル設定

```tla
CONSTANT Users = {"alice", "bob"}
CONSTANT Keys = {"k1", "k2"}
CONSTANT InitialStock = 1
INVARIANT TypeOK
INVARIANT Safety
INVARIANT AtMostOnce
```

保証範囲はこの有限モデルである。
在庫を 1 にすると、二人が同時購入できるかという最小反例を見つけやすい。

## 4. TLC の反例を読む

誤った仕様で `stock > 0` を外すと、次のようなトレースになる。

```text
State 1: stock = 1, orders = [alice |-> none, bob |-> none]
Action Purchase("alice", "k1")
State 2: stock = 0, orders = [alice |-> paid, bob |-> none]
Action Purchase("bob", "k2")
State 3: stock = -1, orders = [alice |-> paid, bob |-> paid]
```

この反例は、実装の if 文の前後ではなく、一つのトランザクション境界で直す。

## 5. TLA+ から実装契約へ

```python
from dataclasses import dataclass

@dataclass
class Order:
    user: str
    key: str
    status: str = "none"

class Shop:
    def __init__(self, stock: int):
        self.stock = stock
        self.orders: dict[str, Order] = {}
        self.applied: set[str] = set()

    def purchase(self, user: str, key: str) -> bool:
        if key in self.applied:
            return True
        if user in self.orders or self.stock <= 0:
            return False
        self.stock -= 1
        self.orders[user] = Order(user, key, "paid")
        self.applied.add(key)
        return True
```

この実装は単一スレッドの教育用であり、DB トランザクションを表していない。
本番では `stock > 0` の判定と減算を原子的にする。

## 6. PBT で操作列を生成する

```python
from hypothesis import given, strategies as st

users = st.sampled_from(["alice", "bob", "carol"])
keys = st.sampled_from(["k1", "k2", "k3"])

@given(st.lists(st.tuples(users, keys), max_size=30))
def test_shop_never_over_sells(actions):
    shop = Shop(1)
    for user, key in actions:
        shop.purchase(user, key)
    assert shop.stock >= 0
    assert len(shop.orders) <= 1
    assert len(shop.applied) <= 1
```

TLA+ の有限モデルと同じ小さな値域で、実装の状態を確認する。
PBT は辞書の更新順や同じキーの再送を混ぜるため、接続ミスを見つける。

## 7. 冪等性の性質

```python
@given(users, keys)
def test_same_key_has_one_effect(user, key):
    shop = Shop(1)
    first = shop.purchase(user, key)
    stock_after_first = shop.stock
    second = shop.purchase(user, key)
    assert second is True
    assert shop.stock == stock_after_first
    assert len(shop.orders) <= 1
```

同じキーの再送を成功として返すか、409 とするかは API 契約で決める。
重要なのは、どちらの応答方針でも在庫の効果が一度だけであることだ。

## 8. 競合を fake で再現する

```python
class NonAtomicShop:
    def __init__(self):
        self.stock = 1
        self.pending = []

    def check(self): return self.stock > 0
    def commit(self, user):
        self.stock -= 1
        self.pending.append(user)

def test_interleaving_exposes_race():
    s = NonAtomicShop()
    assert s.check() and s.check()
    s.commit("alice"); s.commit("bob")
    assert s.stock == -1
```

このテストは本番修正ではなく、TLA+ の競合を実装の粒度で再現する。
ロックや SQL の条件付き更新を入れた実装に対して同じシナリオを再実行する。

## 9. Lean で金額の中核を切り出す

```lean
import Mathlib

def subtotal (prices : List Nat) : Nat := prices.sum
def discount (amount rate : Nat) : Nat := amount * rate / 100
def total (prices : List Nat) (rate : Nat) : Nat :=
  subtotal prices - discount (subtotal prices) rate

theorem subtotal_nonnegative (prices : List Nat) : 0 ≤ subtotal prices := by
  exact Nat.zero_le _

theorem subtotal_append (xs ys : List Nat) :
    subtotal (xs ++ ys) = subtotal xs + subtotal ys := by
  simp [subtotal]
```

保証したいのは明細を分割して集計しても同じ小計になることだ。
返品や税の丸めを追加するなら、別の定義と前提を設ける。

## 10. 割引の範囲を証明する

```lean
import Mathlib

def discount (amount rate : Nat) : Nat := amount * rate / 100

theorem discount_le_amount (amount rate : Nat) (h : rate ≤ 100) :
    discount amount rate ≤ amount := by
  unfold discount
  omega
```

この定理は自然数の整数除算を前提にしている。
rate が 100 より大きい値を許す API なら、定理の前提を満たさず証明できない。
そこで validator と型を整える。

```lean
import Mathlib

structure Percent where
  value : Nat
  bound : value ≤ 100

def boundedDiscount (amount : Nat) (rate : Percent) : Nat :=
  amount * rate.value / 100

theorem boundedDiscount_le (amount : Nat) (rate : Percent) :
    boundedDiscount amount rate ≤ amount := by
  unfold boundedDiscount
  omega
```

## 11. Lean 定理と実装の接続

Lean の `total` と Python の `total` が同じ仕様であることは、名前だけでは保証されない。
次のような property を Python 側に置く。

```python
def python_subtotal(prices: list[int]) -> int:
    return sum(prices)

from hypothesis import given, strategies as st

@given(st.lists(st.integers(min_value=0), max_size=100),
       st.lists(st.integers(min_value=0), max_size=100))
def test_python_subtotal_append(xs, ys):
    assert python_subtotal(xs + ys) == python_subtotal(xs) + python_subtotal(ys)
```

これは Lean の定理の再証明ではない。
移植実装の接続を経験的に確認する役割である。

## 12. どこまで形式化するか

| 対象 | Lean | TLA+ | PBT |
| --- | --- | --- | --- |
| 小計 | 定理 | 不要 | 移植確認 |
| 在庫競合 | 不向き | 中心 | 実装再現 |
| JSON | 型定義のみ | 抽象化 | 往復 |
| DB 障害 | 抽象化が大 | 失敗アクション | fake・統合 |
| UI | 対象外 | 対象外 | E2E / 例示 |

境界は固定ではない。
高価な性質、再利用される性質、反例が見つかった性質から形式化を深くする。

## 13. 段階的ロードマップ

### 段階1: 性質を文章にする

```text
在庫は負にならない。
同じ購入キーの効果は一度だけ。
明細を分割して集計しても小計は変わらない。
```

### 段階2: PBT を CI に置く

```bash
pytest -q tests/property
```

### 段階3: TLA+ の最小モデル

二人、在庫一つ、キー二つで反例を探す。

### 段階4: Lean の中核関数

丸め、割引、集計など純粋計算を型付き定義にする。

### 段階5: 変更プロセスへ組み込む

仕様の変更時に、TLA+ モデル、PBT 性質、Lean 定理のどれが変わるかをレビューする。

## 14. チームへの売り込み方

ツール名ではなく、過去の障害と反例で説明する。
「二重請求をゼロにする」と断言せず、「二人・在庫一個の競合を PR ごとに検査する」と範囲を明示する。
証明を書く担当者だけでなく、反例を直す担当者を決める。
最初の成功は、短い反例が本番バグの原因説明になったときである。

## 15. よくある失敗

仕様を実装の写経にする。
TLC の定数を一つにして競合を消す。
PBT のモデルを実装のコピーにする。
Lean と本番コードの定義が乖離する。
`sorry` や flaky retry を完了扱いにする。
すべての層を一度に形式化して導入が止まる。

## 16. 変更時のレビュー質問

| 質問 | 見る場所 |
| --- | --- |
| 新しい状態はあるか | TLA+ 変数と TypeOK |
| 新しい順序はあるか | Next と liveness |
| 金額の恒等式は変わるか | Lean theorem |
| 入力境界は増えたか | PBT generator |
| 反例を再現できるか | fixture / seed |
| 形式化していない外部依存は何か | fake・統合テスト |

## 17. まとめ

TLA+ は購入フローの競合順序を先に破壊して設計を鍛える。
PBT は API と状態機械の実装を実際の型・シリアライザ・時計で揺さぶる。
Lean は金額や集計の数学的核を全入力で固定する。
三つを重ねると、設計、実装、計算の境界を別々の保証で覆える。
次の改善は、実際の障害ログから最小モデルと最小反例を追加することである。

## 18. 証拠の対応表

| 証拠 | 反例の形 | 修正先 |
| --- | --- | --- |
| TLA+ Safety 違反 | 状態トレース | トランザクション設計 |
| TLA+ Liveness 違反 | 無限待ち | スケジューラ・公平性 |
| PBT 失敗 | 縮小入力・操作列 | 実装・fixture |
| Lean 失敗 | 未証明ゴール | 定義・前提 |

同じ失敗を別ツールの成功で打ち消してはいけない。
Lean の定理が通っても、TLA+ の競合反例は残る。

## 19. 変更の流れ

```text
要求変更
  -> 不変条件を更新
  -> TLA+ の遷移と定数を更新
  -> TLC で最小モデル
  -> PBT の操作と JSON を更新
  -> Lean の純粋関数と定理を更新
  -> CI で三つの証拠を再生成
```

どれか一つだけ更新すると、仕様と実装が静かに乖離する。

## 20. CI の分離

```yaml
jobs:
  tla:
    steps:
      - run: java -cp tla2tools.jar tlc2.TLC -config Shop.cfg Shop.tla
  lean:
    steps:
      - run: lake build
  pbt:
    steps:
      - run: pytest -q
```

ジョブを分けると、どの保証が失敗したかが明確になる。
成功ログにはモデル定数、Lean の toolchain、PBT の例数を記録する。

## 21. 導入を止めないルール

一つの PR で新しいツールを三つ同時に導入しない。
最初は過去障害を一件だけ再現する。
証明できない外部要素を隠さず、テストの担当として記録する。
反例の説明をレビュー本文に残す。

## 22. 最終チェックリスト

- [ ] TLA+ の状態変数が実装状態に対応する
- [ ] 一アクションの atomicity が明示される
- [ ] PBT が同一キー・空入力・境界値を生成する
- [ ] shrink 済み反例が保存される
- [ ] Lean の定義と移植実装の対応が明記される
- [ ] `sorry` を完了の証拠にしていない
- [ ] 形式化しない部分が別テストで覆われる
- [ ] チームが保証範囲を一文で言える

## 23. まとめ

統合の価値はツールの数ではない。
設計の順序、実装の入力、計算の全域性を、それぞれ最も適した証拠で結ぶことにある。

## 24. 一文で説明する

```text
TLA+ が同時実行の穴を探し、PBT が実装の境界を揺さぶり、
Lean が純粋な計算の核を全入力で固定する。
```

この一文をチームの README に置き、各ジョブのログから対応する証拠へリンクする。
