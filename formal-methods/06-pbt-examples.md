# PBT 実践サンプル

この章では Web、数学、物理学、ゲームの実装を Property-Based Testing で検査する。
前提は [05-pbt-basics.md](./05-pbt-basics.md) の generator、shrink、stateful test である。
すべてを本番サービスへ接続せず、外部依存は純粋な小関数または fake に置き換える。

## 1. Web: シリアライズ往復

保証したいのは、許容された注文値を JSON 化して戻すと同じ意味になることだ。
破れると、再読込で金額や ID が変わる。

```python
import json
from dataclasses import dataclass, asdict
from hypothesis import given, strategies as st

@dataclass(frozen=True)
class Order:
    id: str
    cents: int

def encode(o: Order) -> str:
    return json.dumps(asdict(o), separators=(",", ":"), sort_keys=True)

def decode(s: str) -> Order:
    x = json.loads(s)
    return Order(id=x["id"], cents=x["cents"])

@given(st.builds(Order, st.text(min_size=1, max_size=20), st.integers(0, 10**8)))
def test_order_round_trip(o):
    assert decode(encode(o)) == o
```

## 2. Web: REST / GraphQL バリデーション

```python
from hypothesis import given, strategies as st

def valid_limit(x: int) -> bool:
    return 1 <= x <= 100

@given(st.integers())
def test_limit_validator_is_exact(x):
    assert valid_limit(x) == (1 <= x <= 100)

@given(st.integers(min_value=1, max_value=100))
def test_valid_limit_never_rejected(x):
    assert valid_limit(x)
```

入力を受け入れるだけでなく、上限・下限の境界を性質として固定する。
GraphQL の resolver では、同じ validator を共有し、HTTP 層だけの検査にしない。

## 3. Web: 認可ルールのモデル比較

```python
from hypothesis import given, strategies as st

roles = st.sampled_from(["guest", "member", "admin"])
actions = st.sampled_from(["read", "write", "delete"])

def model(role, action):
    return role == "admin" or (role == "member" and action in {"read", "write"}) \
        or (role == "guest" and action == "read")

def production(role, action):
    table = {
        ("guest", "read"): True, ("member", "read"): True,
        ("member", "write"): True,
    }
    return role == "admin" or table.get((role, action), False)

@given(roles, actions)
def test_authorization_matches_model(role, action):
    assert production(role, action) == model(role, action)
```

モデルと実装を同じ辞書から作らない。
実装の default allow を一例でなく全組合せで露出させる。

## 4. Web: レート制限のステートマシン

```python
from hypothesis.stateful import RuleBasedStateMachine, rule, invariant
from hypothesis import strategies as st

class RateLimitMachine(RuleBasedStateMachine):
    def __init__(self):
        super().__init__(); self.now = 0; self.calls = []

    @rule(delta=st.integers(0, 10))
    def advance(self, delta):
        self.now += delta
        self.calls = [t for t in self.calls if t > self.now - 60]

    @rule()
    def request(self):
        accepted = len(self.calls) < 3
        if accepted: self.calls.append(self.now)
        assert accepted == (len(self.calls) <= 3)

    @invariant()
    def bounded(self):
        assert len(self.calls) <= 3

TestRateLimit = RateLimitMachine.TestCase
```

時間を進める操作を生成しないと、期限切れのバグは見つからない。
本番の時計ではなく、注入可能な clock を使う。

## 5. 数学: 数値演算

```python
from hypothesis import given, strategies as st

@given(st.integers(-10**6, 10**6), st.integers(-10**6, 10**6))
def test_add_commutative(a, b):
    assert a + b == b + a

@given(st.integers(), st.integers(), st.integers())
def test_add_associative(a, b, c):
    assert (a + b) + c == a + (b + c)
```

整数の性質を浮動小数へ無批判に移すと失敗する。

```python
from hypothesis import given, strategies as st

@given(st.floats(allow_nan=False, allow_infinity=False, width=64))
def test_float_identity_is_not_exact(x):
    assert abs((x + 1.0) - x - 1.0) <= 2.0 ** -40 or abs(x) > 2.0 ** 40
```

許容誤差を仕様に含める。`==` は数値計算の性質として危険である。

## 6. 数学: ソートと集合演算

```python
from hypothesis import given, strategies as st

@given(st.lists(st.integers()))
def test_sorted_preserves_multiset(xs):
    assert sorted(xs) == sorted(sorted(xs))
    assert len(sorted(xs)) == len(xs)

@given(st.sets(st.integers()), st.sets(st.integers()))
def test_union_commutative(a, b):
    assert a | b == b | a
```

最初の性質は冪等性と長さ保存であり、順序を直接期待していない。
集合の性質で重複数を検査したいなら、multiset を使う。

## 7. 数学: 任意精度と丸め

```python
from decimal import Decimal, localcontext
from hypothesis import given, strategies as st

@given(st.integers(-100000, 100000), st.integers(-100000, 100000))
def test_decimal_add_exact(a, b):
    with localcontext() as ctx:
        ctx.prec = 40
        assert Decimal(a) + Decimal(b) == Decimal(a + b)
```

浮動小数で通ると思い込んだ往復性を、Decimal と比較して差を可視化する。

## 8. 物理: 積分器のエネルギー誤差

```python
from hypothesis import given, strategies as st

def step(x, v, dt):
    return x + v * dt, v - x * dt

def energy(x, v):
    return (x*x + v*v) / 2

@given(st.floats(-1, 1, allow_nan=False),
       st.floats(-1, 1, allow_nan=False),
       st.floats(0, 0.01, allow_nan=False))
def test_energy_error_is_bounded(x, v, dt):
    x2, v2 = step(x, v, dt)
    assert abs(energy(x2, v2) - energy(x, v)) < 0.02
```

これは一ステップの近似誤差を上限で縛る性質であり、長時間保存の証明ではない。
`dt` を大きくすると反例が出て、積分器の適用範囲が分かる。

## 9. 物理: 衝突判定の対称性

```python
from hypothesis import given, strategies as st

def collide(a, b):
    ax, ay, ar = a; bx, by, br = b
    return (ax-bx)**2 + (ay-by)**2 <= (ar+br)**2

points = st.tuples(st.floats(-100,100), st.floats(-100,100),
                   st.floats(0,10))

@given(points, points)
def test_collision_symmetric(a, b):
    assert collide(a, b) == collide(b, a)
```

片方だけに適用する補正を混ぜると対称性が破れる。
ゲーム物理では、NaN や極端な半径を generator で除外した理由も記録する。

## 10. 物理: 単位変換の往復

```python
from hypothesis import given, strategies as st

def meters_to_cm(x): return x * 100
def cm_to_meters(x): return x / 100

@given(st.floats(-1e6, 1e6, allow_nan=False, allow_infinity=False))
def test_unit_round_trip(x):
    assert abs(cm_to_meters(meters_to_cm(x)) - x) <= max(1e-9, abs(x)*1e-12)
```

整数の単位変換なら完全一致を要求できるが、浮動小数では誤差を契約にする。

## 11. ゲーム: インベントリ state machine

```python
from hypothesis.stateful import RuleBasedStateMachine, rule, invariant
from hypothesis import strategies as st

class InventoryMachine(RuleBasedStateMachine):
    def __init__(self):
        super().__init__(); self.model = {}; self.actual = {}

    @rule(item=st.integers(0, 5), n=st.integers(0, 3))
    def add(self, item, n):
        self.model[item] = self.model.get(item, 0) + n
        self.actual[item] = self.actual.get(item, 0) + n

    @rule(item=st.integers(0, 5), n=st.integers(0, 3))
    def remove(self, item, n):
        old = self.model.get(item, 0)
        taken = min(old, n); self.model[item] = old - taken
        self.actual[item] = max(0, self.actual.get(item, 0) - n)

    @invariant()
    def agrees(self): assert self.actual == self.model

TestInventory = InventoryMachine.TestCase
```

remove の underflow をモデルと比較する。ゲーム内通貨の負数は不正購入につながる。

## 12. ゲーム: 乱数 seed の再現性

```python
import random
from hypothesis import given, strategies as st

def roll(seed, n):
    r = random.Random(seed)
    return [r.randrange(6) + 1 for _ in range(n)]

@given(st.integers(), st.integers(0, 100))
def test_seed_replays(seed, n):
    assert roll(seed, n) == roll(seed, n)
```

グローバル RNG を使うと、別テストの実行順に左右される。

## 13. ゲーム: セーブ / ロード往復

```python
import json
from hypothesis import given, strategies as st

def save(state): return json.dumps(state, sort_keys=True)
def load(blob): return json.loads(blob)

states = st.fixed_dictionaries({
    "level": st.integers(0, 100),
    "coins": st.integers(0, 10**6),
    "items": st.lists(st.text(min_size=1, max_size=8), max_size=10),
})

@given(states)
def test_save_load(state): assert load(save(state)) == state
```

バージョン移行を含む場合は、旧スキーマも generator に残す。

## 14. ゲーム: バランス調整の単調性

```python
from hypothesis import given, strategies as st

def damage(power, armor): return max(0, power - armor)

@given(st.integers(0, 1000), st.integers(0, 1000), st.integers(0, 1000))
def test_damage_monotone_power(p, q, armor):
    if p <= q: assert damage(p, armor) <= damage(q, armor)

@given(st.integers(0, 1000), st.integers(0, 1000), st.integers(0, 1000))
def test_damage_antitone_armor(power, a, b):
    if a <= b: assert damage(power, b) <= damage(power, a)
```

装備を強くして結果が悪くなる不具合を捉える。

## 15. 失敗例の運用

反例は seed だけでなく入力値、ライブラリ版、環境、性質名とともに保存する。
本番で起きた入力は generator に追加し、縮小後の例は回帰テストにする。
テストが遅い場合、性質を削らず、入力を構造化して shrink を効かせる。
次章の統合例では、TLA+ の設計反例をこの方法で実装へ接続する。

## 16. 境界値を表にする

| 領域 | 最小 | 通常 | 最大 | 不正 |
| --- | --- | --- | --- | --- |
| ページサイズ | 0 / 1 | 20 | 100 | -1 |
| ダメージ | 0 | 50 | 1000 | NaN |
| 在庫 | 0 | 10 | 上限 | 負数 |
| セッション年齢 | 0 | TTL 内 | TTL | TTL 超過 |

テストの入力範囲と、API が返すエラーの契約を同じ表で管理する。

## 17. Web の偽装依存

```python
from hypothesis import given, strategies as st

class FakePayment:
    def __init__(self): self.calls = []
    def charge(self, key, cents):
        self.calls.append((key, cents))
        return key not in {"fail"}

@given(st.text(min_size=1, max_size=10), st.integers(0, 10000))
def test_payment_key_is_observable(key, cents):
    fake = FakePayment()
    fake.charge(key, cents)
    assert fake.calls == [(key, cents)]
```

外部決済を毎回本当に呼ぶのではなく、呼び出し回数と引数を観測する。

## 18. 物理の不正値

```python
from hypothesis import given, strategies as st

finite = st.floats(allow_nan=False, allow_infinity=False)

@given(finite)
def test_abs_nonnegative(x):
    assert abs(x) >= 0
```

NaN を受け入れるかは、除外ではなく仕様として決める。

## 19. ゲームの回帰例

```python
def test_known_replay():
    assert roll(42, 4) == roll(42, 4)
```

PBT の反例から固定した例は、短い回帰テストとして残す。
seed だけでなく、ゲームルールの版も保存する。

## 20. まとめ

実例では、往復、モデル比較、誤差境界、状態機械、再現性、単調性を使い分けた。
同じ入力 generator を使い回すのではなく、保証対象に合わせて狭く設計する。

## 21. 実例からの持ち帰り

```text
往復性: 保存・送信・復元の境界
モデル比較: 認可と状態機械
数値誤差: 許容幅を契約にする
対称性: 入力の交換で結果を比較する
単調性: バランス調整の退行を防ぐ
```

それぞれの性質を名前付きテストにすると、失敗時の説明が短くなる。
PBT は仕様を隠すためではなく、仕様を反復実行可能な文にするために使う。

## 22. 統合へ

反例を保存したら、設計モデル・実装テスト・純粋計算の証明を同じ題材で結ぶ。
その一連の流れを [07-combining.md](./07-combining.md) で扱う。

失敗入力をチームの共有資産に変えることが、PBT の実務上の完了条件である。

その資産を仕様変更時に再利用する。
