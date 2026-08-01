# TLA+ 実践サンプル

この章では、Web、ゲーム、物理・数学寄りの三領域を状態機械として表す。
前提は [03-tlaplus-basics.md](./03-tlaplus-basics.md) の `Init`、`Next`、不変条件である。
各例は小さいモデルに意図的に縮約している。縮約したもの、落としたものを明記する。

## 1. Web: 在庫引き当てと二重購入

保証したいのは、在庫が負にならず、利用者一人につき注文が高々一つであることだ。
これが破れると、欠品、返金、会計帳簿との不一致が発生する。

```tla
---- MODULE Inventory ----
CONSTANT Users, InitialStock
VARIABLE stock, orders

Init == /\\ stock = InitialStock /\\ orders = [u \in Users |-> "none"]
Purchase(u) == /\\ orders[u] = "none"
                /\\ stock > 0
                /\\ stock' = stock - 1
                /\\ orders' = [orders EXCEPT ![u] = "paid"]
Next == \\E u \in Users : Purchase(u)
TypeOK == /\\ stock \in Nat
          /\\ orders \in [Users -> {"none", "paid"}]
NoDouble == \\A u \in Users : orders[u] = "none" \/ orders[u] = "paid"
Conserve == stock + Cardinality({u \in Users : orders[u] = "paid"}) = InitialStock
Spec == Init /\\ [][Next]_<<stock, orders>>
================================================
```

`Conserve` は購入が常に一個の在庫を消費することを検査する。
決済失敗、キャンセル、返金を追加するなら、支払い状態を増やして遷移を分ける。

## 2. Web: セッションとトークン失効

```tla
---- MODULE Session ----
CONSTANT Tokens
VARIABLE status

Init == status = [t \in Tokens |-> "valid"]
Revoke(t) == /\\ status[t] = "valid"
             /\\ status' = [status EXCEPT ![t] = "revoked"]
Use(t) == /\\ status[t] = "valid"
          /\\ UNCHANGED status
Next == \\E t \in Tokens : Revoke(t) \\/ Use(t)
TypeOK == status \in [Tokens -> {"valid", "revoked"}]
RevokedNeverValid == \\A t \in Tokens : status[t] = "revoked" => status[t] # "valid"
Spec == Init /\\ [][Next]_status
================================================
```

失効済みトークンを再び有効にする遷移がないことを保証する。
キャッシュ層が `Use` の前に古い値を返す場合は、キャッシュを別の変数としてモデル化する。

## 3. Web: リトライと冪等性

同じ要求が二度届いても効果が一度だけであることを表す。

```tla
---- MODULE Idempotency ----
CONSTANT Keys
VARIABLE applied, balance

Init == /\\ applied = {} /\\ balance = 0
Apply(k) == /\\ k \in Keys
            /\\ k \notin applied
            /\\ applied' = applied \cup {k}
            /\\ balance' = balance + 1
Retry(k) == /\\ k \in applied
            /\\ UNCHANGED <<applied, balance>>
Next == \\E k \in Keys : Apply(k) \\/ Retry(k)
AtMostOnce == balance = Cardinality(applied)
Spec == Init /\\ [][Next]_<<applied, balance>>
================================================
```

キーがDBの一意制約で守られているか、メモリだけの集合でないかは実装検査へ引き継ぐ。

## 4. Web: キャッシュ整合性

```tla
---- MODULE Cache ----
CONSTANT Values
VARIABLE origin, cache, version

Init == /\\ origin = Values[1]
        /\\ cache = origin
        /\\ version = 0
Read == /\\ cache' = cache /\\ UNCHANGED <<origin, version>>
Write(v) == /\\ v \in Values
            /\\ origin' = v
            /\\ cache' = v
            /\\ version' = version + 1
StaleRead == /\\ cache' = cache /\\ UNCHANGED <<origin, version>>
Next == Read \\/ \\E v \in Values : Write(v) \\/ StaleRead
Coherent == cache = origin
Spec == Init /\\ [][Next]_<<origin, cache, version>>
================================================
```

`StaleRead` を許すなら `Coherent` は破れる。これはバグではなく、許容する eventual consistency の仕様かもしれない。
「読めること」と「最新版であること」を分けて書くのが要点である。

## 5. ゲーム: ターン制の手番遷移

```tla
---- MODULE Turns ----
CONSTANT Players
VARIABLE turn, moves

Init == /\\ turn = Players[1] /\\ moves = 0
Play(p) == /\\ p = turn
           /\\ moves' = moves + 1
           /\\ turn' = IF turn = Players[1] THEN Players[2] ELSE Players[1]
TypeOK == /\\ turn \in Players /\\ moves \in Nat
Next == \\E p \in Players : Play(p)
TurnExclusion == \\A p \in Players : p # turn => ~ (p = turn)
Spec == Init /\\ [][Next]_<<turn, moves>>
================================================
```

現在の手番以外はプレイできないことを保証する。
プレイヤーが二人より多い場合、環状順序を別の関数にする必要がある。

## 6. ゲーム: マッチメイキングのデッドロック

```tla
---- MODULE Match ----
CONSTANT Players
VARIABLE waiting, matched

Init == /\\ waiting = {} /\\ matched = {}
Join(p) == /\\ p \in Players /\\ p \notin waiting /\\ p \notin matched
           /\\ waiting' = waiting \cup {p} /\\ UNCHANGED matched
Pair(a, b) == /\\ a \in waiting /\\ b \in waiting /\\ a # b
              /\\ matched' = matched \cup {{a, b}}
              /\\ waiting' = waiting \ {a, b}
Next == \\E p \in Players : Join(p)
        \\/ \\E a, b \in Players : Pair(a, b)
NoDeadlockForTwo == Cardinality(waiting) = 2 => <> (Cardinality(waiting) < 2)
Spec == Init /\\ [][Next]_<<waiting, matched>>
================================================
```

「二人待っているならいつかペアになる」は liveness である。
公平性なしでは、常に `Join` ばかり選ばれる反例が出るので、`WF` を追加して検査する。

## 7. ゲーム: アイテム取引のアトミック性

```tla
---- MODULE Trade ----
CONSTANT Players, Items
VARIABLE inventory, offers

Init == /\\ inventory = [p \in Players |-> {}]
        /\\ offers = {}
Offer(a, b, i) == /\\ i \in inventory[a]
                   /\\ offers' = offers \cup {[from |-> a, to |-> b, item |-> i]}
                   /\\ UNCHANGED inventory
Accept(o) == /\\ o \in offers
             /\\ o.item \in inventory[o.from]
             /\\ inventory' = [inventory EXCEPT
                 ![o.from] = @ \ {o.item}, ![o.to] = @ \cup {o.item}]
             /\\ offers' = offers \ {o}
Next == \\E a, b \in Players, i \in Items : Offer(a,b,i)
        \\/ \\E o \in offers : Accept(o)
NoDup == \\A p \in Players : inventory[p] \intersect inventory[p] = inventory[p]
Spec == Init /\\ [][Next]_<<inventory, offers>>
================================================
```

送信者から削除し受信者へ追加する処理を一アクションにすることで、半分だけ移る状態を許さない。
実装が二つのDB書き込みなら、トランザクション境界を別に検査する。

## 8. PlusCal と素の TLA+ を同じ題材で比較

PlusCal 版は人間のアルゴリズム順に読む。

```tla
---- MODULE LockPlusCal ----
VARIABLE lock

(* --algorithm Mutex
variables lock = "free";
process (P \in {"A", "B"})
begin
Acquire:
  await lock = "free";
  lock := self;
Critical:
  skip;
Release:
  lock := "free";
end process;
end algorithm; *)
================================
```

同じ意図を素の TLA+ で書く。

```tla
---- MODULE LockRaw ----
CONSTANT Proc
VARIABLE owner

Init == owner = "free"
Acquire(p) == /\\ owner = "free" /\\ owner' = p
Release(p) == /\\ owner = p /\\ owner' = "free"
Next == \\E p \in Proc : Acquire(p) \\/ Release(p)
MutualExclusion == owner = "free" \/ owner \in Proc
Spec == Init /\\ [][Next]_owner
================================
```

PlusCal の変換後に、`Acquire` の原子性や `Release` の所有者条件を必ず確認する。

## 9. 物理・数学: 離散化の不変量

```tla
---- MODULE DiscreteSum ----
CONSTANT N
VARIABLE i, total

Init == /\\ i = 0 /\\ total = 0
Step == /\\ i < N
        /\\ i' = i + 1
        /\\ total' = total + i'
Done == i = N
Invariant == total = i * (i - 1) / 2
Next == Step
Spec == Init /\\ [][Next]_<<i, total>>
================================================
```

離散積分の途中値が閉形式 `i(i-1)/2` と一致することを検査する。
`N` を自然数に制限し、整数除算の意味を確認する。

## 10. 物理・数学: 資源保存

```tla
---- MODULE Conservation ----
CONSTANT Capacity, Agents
VARIABLE resource

Init == resource = [a \in Agents |-> Capacity]
Move(a, b) == /\\ a # b /\\ resource[a] > 0
              /\\ resource' = [resource EXCEPT ![a] = @ - 1, ![b] = @ + 1]
Total == Sum({resource[a] : a \in Agents})
Invariant == Total = Cardinality(Agents) * Capacity
Next == \\E a, b \in Agents : Move(a,b)
Spec == Init /\\ [][Next]_resource
================================================
```

資源移動で総量が保存されることを保証する。
`Capacity` を各エージェントの初期量と呼んでいるだけで、上限を守る仕様ではない。

## 11. 相互排除とラウンドロビン公平性

```tla
---- MODULE RoundRobin ----
CONSTANT Proc
VARIABLE turn, inCS

Init == /\\ turn = Proc[1] /\\ inCS = {}
Enter(p) == /\\ p = turn /\\ p \notin inCS /\\ inCS' = inCS \cup {p}
Leave(p) == /\\ p \in inCS /\\ inCS' = inCS \ {p}
Advance(p) == /\\ p = turn /\\ p \notin inCS
              /\\ turn' = Proc[1 + (Index(Proc, p) % Len(Proc))]
              /\\ UNCHANGED inCS
Next == \\E p \in Proc : Enter(p) \\/ Leave(p) \\/ Advance(p)
MutualExclusion == Cardinality(inCS) <= 1
Spec == Init /\\ [][Next]_<<turn, inCS>>
================================================
```

`Advance` のインデックス表現は説明用の疑似的な部分を含む。
実際の TLA+ では `Proc` をシーケンスとしてモデル化し、`Len` と `SubSeq` で厳密に書く。
このように、動かない断片を完全なコードと誤認させないことが重要である。

## 12. モデルの境界

| 抽象化 | 得られるもの | 失うもの |
| --- | --- | --- |
| 決済を一アクション | 二重請求の順序 | ネットワーク分断 |
| キャッシュを値一つ | stale read の論理 | TTL と容量 |
| プレイヤー二人 | 手番競合 | 大規模マッチング |
| 離散時間 | 不変量 | 連続系の数値誤差 |

失ったものは [06-pbt-examples.md](./06-pbt-examples.md) で実装テストに回す。
中核の代数的性質は [02-lean-examples.md](./02-lean-examples.md) のように Lean へ切り出す。

## 13. まとめ

TLA+ の例では、状態変数を増やすと現実の失敗モードを表せる一方、状態爆発も増える。
最初は 2 利用者、容量 1、短い履歴で反例を探し、発見した設計欠陥を実装へ反映する。
次章は、実装に対して性質を大量に走らせる PBT である。

## 14. Web モデルの拡張課題

在庫モデルに `cancel` を追加する。
キャンセルは支払い前後で意味が異なる。
支払い前なら在庫を戻すが、支払い後は返金イベントが必要になる。
この違いを一つの `Cancel` に押し込むと、保存不変量が曖昧になる。

```tla
Cancel(u) == /\\ orders[u] = "reserved"
             /\\ orders' = [orders EXCEPT ![u] = "cancelled"]
             /\\ stock' = stock + 1
```

保証したいのは、予約済みだけがキャンセル可能であることだ。
`paid` もキャンセル可能にするなら、返金の台帳を別変数にする。

## 15. ゲームモデルの拡張課題

ターン制ゲームに切断を追加する。

```tla
Disconnect(p) == /\\ p \in Players
                 /\\ connected' = connected \ {p}
                 /\\ UNCHANGED turn
```

接続がないプレイヤーに手番を渡さないという Safety と、接続中のプレイヤーへいつか進むという Liveness を分ける。
切断と再接続の順序を変えると、デッドロックの反例が変化する。

## 16. 物理モデルの検証手順

```text
単位を定数名に含める
時間刻みを小さくする
初期値を二種類以上置く
不変量を一ステップで代数的に確認する
長時間の循環を liveness で調べる
数値誤差を PBT で測る
```

TLA+ の整数モデルは、浮動小数の丸めを自動的に再現しない。
丸め幅を抽象化して別モデルにするか、実装テストへ渡す。

## 17. モデル品質のチェックリスト

- [ ] 仕様の一アクションを日本語で説明できる
- [ ] 初期状態が空でない
- [ ] 実装に存在する失敗を一つ以上含めた
- [ ] 不変条件が恒真式になっていない
- [ ] 定数が競合を消していない
- [ ] 順序が必要な履歴を集合にしていない
- [ ] liveness の公平性を説明できる
- [ ] 抽象化で失った性質を PBT に記録した

## 18. まとめ

実例の目的は TLA+ の文法を網羅することではなく、設計判断を反例に変換することだ。
Web では原子性と冪等性、ゲームでは手番と所有権、物理では保存量と近似範囲を中心にモデル化する。

## 19. 反例を実装へ渡すテンプレート

```text
モデル名:
定数:
違反した性質:
状態列:
競合したアクション:
実装で対応する API:
修正の atomicity:
PBT にする操作列:
```

反例をそのままチームの共有語彙にする。
反例の状態を削除せず、仕様変更で無効になった理由も記録する。

## 20. 実務での停止条件

モデル検査が終わらないときは、クライアントや履歴を増やす前に抽象化を見直す。
反例が十個出たら、同じ原因を分類してから修正する。
Liveness は Safety のモデルが安定してから追加する。
この順序で、検査時間とレビュー負荷を制御する。

## 21. 次へ

抽象モデルで見つけた順序を実装の操作列に移す方法は [05-pbt-basics.md](./05-pbt-basics.md) と [07-combining.md](./07-combining.md) に続く。

モデルの反例は、実装者を責める材料ではなく、状態境界を改善する設計資料として扱う。

## 22. 実例の共通原則

原子性、冪等性、保存、所有権、公平性は、Web・ゲーム・物理で名前を変えて現れる。

これらは PBT の性質にも変換できる。

反例を実装で再現し、修正後も残す。

モデルと実装の対応を文書化する。

その対応が統合の入口になる。

仕様は一度書いて終わりではない。

運用で得た失敗を再びモデルへ返す。

これが継続的な検証のループである。

次は PBT の generator を設計する。
