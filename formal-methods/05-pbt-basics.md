# Property-Based Testing 基礎

この章では Property-Based Testing（PBT）を、入力生成器と性質の組として学ぶ。
前提は Python または TypeScript のユニットテストである。
PBT は乱数を増やす技法ではない。例を一般化し、壊れたときに最小反例を保存する設計技法である。

## 1. 例示テストとの違い

```python
def reverse(xs: list[int]) -> list[int]:
    return list(reversed(xs))

def test_reverse_example():
    assert reverse([1, 2, 3]) == [3, 2, 1]
```

例示テストは具体的な契約を伝える。
PBT は複数の一般性を検査する。

```python
from hypothesis import given, strategies as st

@given(st.lists(st.integers()))
def test_reverse_involution(xs):
    assert reverse(reverse(xs)) == xs
```

保証したいのは往復性であり、破れると編集画面や Undo のデータが壊れる。

## 2. Hypothesis の最小プロジェクト

```bash
python -m venv .venv
. .venv/bin/activate
python -m pip install hypothesis pytest
pytest -q
```

```python
# test_strings.py
from hypothesis import given, strategies as st

@given(st.text())
def test_identity(s: str) -> None:
    assert s == "".join(list(s))
```

失敗時には、Hypothesis が保存した seed や縮小済み入力を再実行できる。
CI のログから再現例を課題にコピーし、回帰テストとして残す。

## 3. fast-check の最小プロジェクト

```bash
npm init -y
npm install --save-dev typescript tsx fast-check vitest
npx vitest run
```

```typescript
import fc from "fast-check";
import { test } from "vitest";

test("reverse is involutive", () => {
  fc.assert(fc.property(fc.array(fc.integer()), xs => {
    return [...xs].reverse().reverse().every((x, i) => x === xs[i]);
  }));
});
```

TypeScript では型注釈とジェネレータの型を一致させる。

## 4. 性質の見つけ方

| パターン | 性質 | 典型例 |
| --- | --- | --- |
| 往復 | `decode(encode(x)) = x` | JSON、セーブデータ |
| 不変 | 操作後も条件が残る | 在庫、盤面 |
| 冪等 | 二回しても一回と同じ | PUT、失効 |
| 可換 | 順序を変えて同じ | 集合和、独立更新 |
| メタモルフィック | 入力変換に応じて出力変換 | 拡大、置換 |
| モデル比較 | 簡単なモデルと一致 | 認可、状態機械 |
| オラクル比較 | 別実装と一致 | パーサ、数値計算 |

性質が書けない場合、対象の仕様がまだ曖昧である。

## 5. ジェネレータ

```python
from hypothesis import strategies as st

user_ids = st.text(alphabet=st.characters(whitelist_categories=("Ll",)),
                   min_size=1, max_size=12)
prices = st.integers(min_value=0, max_value=1_000_000)
orders = st.fixed_dictionaries({
    "user": user_ids,
    "price": prices,
    "items": st.lists(st.integers(min_value=1, max_value=99), max_size=20),
})
```

生成器は現実の制約を表す。
入力を広げすぎて無効入力ばかりになると、重要な性質の実行回数が減る。
逆に制約を狭めすぎると、バグのある領域を生成しなくなる。

## 6. `assume` の使いどころ

```python
from hypothesis import given, assume, strategies as st

@given(st.integers(), st.integers())
def test_division_round_trip(a, b):
    assume(b != 0)
    assert (a / b) * b == a
```

この性質は浮動小数では偽なので、悪い例である。
`assume` を大量に使うと生成効率が落ち、仕様の穴を隠す。
有効な入力を最初から strategy で生成する方がよい。

## 7. shrink の仕組み

```python
from hypothesis import given, strategies as st

@given(st.lists(st.integers(), min_size=1))
def test_no_negative(xs):
    assert all(x >= 0 for x in xs)
```

バグがあると、Hypothesis はリストを短くし、値を小さくし、構造を単純化する。
失敗例 `[0, -1]` は、巨大な業務入力より原因を読める。
カスタム型は縮小可能な構造を持たせる。

```python
from dataclasses import dataclass
from hypothesis import strategies as st

@dataclass
class Command:
    name: str
    amount: int

commands = st.builds(Command,
                     name=st.sampled_from(["add", "remove"]),
                     amount=st.integers(0, 100))
```

## 8. seed と再現

Hypothesis は失敗例をデータベースに保存する。
fast-check は `seed` と `path` をログに出す。

```typescript
import fc from "fast-check";

fc.assert(fc.property(fc.array(fc.integer()), xs => {
  return xs.length < 100;
}), { seed: 12345, path: "0:1" });
```

乱数 seed はデバッグの入口であり、仕様の証拠ではない。
生成器やライブラリ版が変わると同じ seed でも入力が変わるため、縮小済み入力を回帰テストにする。

## 9. ステートフルテスト

```python
from hypothesis.stateful import RuleBasedStateMachine, rule, invariant

class SetModel(RuleBasedStateMachine):
    def __init__(self):
        super().__init__()
        self.expected: set[int] = set()
        self.actual: set[int] = set()

    @rule(x=st.integers())
    def add(self, x):
        self.expected.add(x)
        self.actual.add(x)

    @rule(x=st.integers())
    def remove(self, x):
        self.expected.discard(x)
        self.actual.discard(x)

    @invariant()
    def agrees(self):
        assert self.actual == self.expected

TestSetModel = SetModel.TestCase
```

モデルは簡単であるほどよい。
同じ実装をモデルにコピーすると、同じバグを共有して比較が無意味になる。

## 10. モデルベーステスト

```typescript
type State = "closed" | "open";
type Cmd = "open" | "close";

const step = (s: State, c: Cmd): State =>
  c === "open" && s === "closed" ? "open" :
  c === "close" && s === "open" ? "closed" : s;

fc.assert(fc.property(fc.array(fc.constantFrom<Cmd>("open", "close")), cs => {
  let model: State = "closed";
  for (const c of cs) model = step(model, c);
  return model === "closed" || model === "open";
}));
```

実装 API の応答とモデル状態を各コマンド後に比較する。

## 11. オラクル比較

```python
from decimal import Decimal
from hypothesis import given, strategies as st

@given(st.integers(-10000, 10000), st.integers(-10000, 10000))
def test_add_matches_decimal(a, b):
    assert Decimal(a) + Decimal(b) == Decimal(a + b)
```

オラクルが独立していないと比較にならない。
別実装も同じ丸め規則を共有する場合、境界値を別途追加する。

## 12. CI で回す

```yaml
name: property-tests
on: [push, pull_request]
jobs:
  python:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: "3.12" }
      - run: pip install hypothesis pytest
      - run: pytest -q
```

PR では少ない例数、夜間ジョブでは多い例数という二層構成にする。
失敗を flaky として再実行だけで隠さない。

## 13. 失敗例の育て方

1. 生成器が作った最小反例を保存する。
2. バグの原因を一文で説明する。
3. 例示テストとして固定する。
4. 一般化できる性質を PBT に残す。
5. 仕様の欠陥ならモデルや Lean の定理にも反映する。

## 14. アンチパターン

`random` を直接使い、失敗入力を保存しない。
性質の代わりに「例外が出ない」だけを検査する。
`assume` でほとんどの入力を捨てる。
生成器が本番制約と違う。
カバレッジ率を正しさと取り違える。
重い外部 API を全生成例から呼び、CI を不安定にする。

## 15. 他手法との接続

TLA+ の反例トレースを command 列の fixture にして PBT で実行できる。
Lean の定理を PBT の性質として再実装すると、実装との接続を確認できる。
三手法を同じ在庫題材で組み合わせる例は [07-combining.md](./07-combining.md) にある。
実例は [06-pbt-examples.md](./06-pbt-examples.md) へ進む。

## 16. strategy の設計表

| 入力 | 生成器 | 境界 |
| --- | --- | --- |
| ID | `text` | 空、長大、Unicode |
| 金額 | `integers` | 0、最大、負数 |
| 配列 | `lists` | 空、一個、大量 |
| JSON | `recursive` | 深さ、重複キー |
| 操作列 | `lists` of commands | 空、同一操作、長列 |

境界値を手で追加するだけでなく、境界を strategy の定義にする。

```python
from hypothesis import strategies as st

small_amount = st.one_of(
    st.just(0), st.just(1), st.just(100), st.just(101),
    st.integers(0, 1_000_000)
)
```

## 17. 複合データと依存関係

```python
from hypothesis import strategies as st

@st.composite
def nonempty_pair(draw):
    xs = draw(st.lists(st.integers(), min_size=1))
    i = draw(st.integers(0, len(xs) - 1))
    return xs, xs[i]
```

依存する値は `composite` で生成し、`assume` を減らす。
依存制約が多い場合は、データ型自体を変えて不正状態を表現できなくする。

## 18. shrink とデバッグ

失敗した入力は次の順に読む。

```text
1. どの性質が失敗したか
2. 縮小後の入力は何か
3. 失敗を起こす最小フィールドは何か
4. generator は本番の制約を表しているか
5. 修正後の入力を回帰テストにするか
```

巨大な入力をログに貼るだけでは、問題の構造が見えない。
縮小器がうまく働かない型は、状態を小さな dataclass や enum に分割する。

## 19. 可換性と順序

```python
from hypothesis import given, strategies as st

@given(st.sets(st.integers()), st.sets(st.integers()))
def test_set_merge_order(a, b):
    assert (a | b) == (b | a)
```

可換でない業務処理を誤って可換と仮定しない。
注文イベントは順序が意味を持つため、集合ではなくリストで生成する。

## 20. メタモルフィックテスト

```python
from hypothesis import given, strategies as st

@given(st.lists(st.integers()))
def test_sort_translation(xs):
    k = 7
    left = sorted([x + k for x in xs])
    right = [x + k for x in sorted(xs)]
    assert left == right
```

正解を計算できない処理でも、入力変換と出力変換の関係を使える。

## 21. 例数と統計

```python
from hypothesis import settings

@settings(max_examples=300, deadline=None)
@given(st.lists(st.integers(), max_size=50))
def test_property(xs):
    assert len(xs) == len(list(xs))
```

例数を増やす前に、失敗を検出する generator になっているか確認する。
`deadline=None` は重いテストを許す設定で、無制限の本番外部呼び出しを許す意味ではない。

## 22. TypeScript の command generator

```typescript
import fc from "fast-check";

type Command = { kind: "add"; n: number } | { kind: "clear" };
const commandArb = fc.oneof(
  fc.record({ kind: fc.constant("add" as const), n: fc.integer() }),
  fc.constant({ kind: "clear" as const })
);

fc.assert(fc.property(fc.array(commandArb), commands => {
  const xs: number[] = [];
  for (const c of commands) c.kind === "add" ? xs.push(c.n) : xs.splice(0);
  return Array.isArray(xs);
}));
```

union 型を使うと、コマンドの引数不足がコンパイル時に分かる。

## 23. CI の分割

```yaml
jobs:
  fast:
    steps:
      - run: pytest -q tests/property --hypothesis-profile=ci
  nightly:
    steps:
      - run: pytest -q tests/property --hypothesis-profile=nightly
```

高速ジョブは PR のフィードバック、夜間ジョブは広い探索を担う。
失敗した property は同じコードの再実行だけで済ませず、seed と縮小例をアーティファクトに残す。

## 24. まとめ

PBT の品質はケース数でなく、性質、生成器、縮小器、失敗運用の組で決まる。
実例は [06-pbt-examples.md](./06-pbt-examples.md)、三手法の接続は [07-combining.md](./07-combining.md) に進む。
