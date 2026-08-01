# TLA+ 基礎

この章では TLA+ を、並行・分散システムの状態遷移を検査する道具として学ぶ。
前提は変数、集合、論理式、簡単なプログラムの実行順序である。
Lean が式の証明を主役にするのに対し、TLA+ は「どの状態からどの状態へ動けるか」を主役にする。

## 1. 状態とアクション

状態は変数の値の組である。
アクションは現在値と次状態値を関係づける式で、プライム記号 `'` は次状態を表す。

```tla
---------------- MODULE Counter ----------------
VARIABLE count

Init == count = 0
Inc == count' = count + 1
Next == Inc
Spec == Init /\\ [][Next]_count
===============================================
```

`Init` は初期状態、`Next` は一ステップの遷移、`Spec` は実行全体の仕様である。
`[][Next]_count` は `Next` が起きるか、観測変数 `count` 以外だけが変わることを表す。

## 2. 不変条件

```tla
VARIABLE count
Init == count = 0
Next == count' = count + 1
TypeOK == count \in Nat
```

`TypeOK` は状態型、不変条件は到達可能な全状態で成立してほしい述語である。
TLC はモデルの状態を探索し、違反時に初期状態からのトレースを出す。

## 3. Safety と Liveness

| 種類 | 意味 | 例 | 反例 |
| --- | --- | --- | --- |
| Safety | 悪いことが起きない | 在庫は負でない | 違反状態の有限トレース |
| Liveness | 良いことがいつか起きる | 要求はいつか応答 | 無限に停滞する実行 |

Safety は「常に」を `[]` で書く。
Liveness は「いつか」を `<>` で書く。

```tla
Safety == [] (count >= 0)
Progress == <> (count > 0)
```

TLC の有限モデルで liveness を検査するには公平性と循環検出の設定が必要である。

## 4. 集合と関数

```tla
CONSTANT Users
VARIABLE owner

Init == owner = [u \in Users |-> "free"]
Claim(u) == /\\ owner[u] = "free"
             /\\ owner' = [owner EXCEPT ![u] = "claimed"]
Release(u) == /\\ owner[u] = "claimed"
              /\\ owner' = [owner EXCEPT ![u] = "free"]
Next == \\E u \in Users : Claim(u) \\/ Release(u)
```

関数更新は元の関数を変更せず、新しい関数を作る。
集合は順序を持たないため、キューの順序を表すにはシーケンスを使う。

## 5. `UNCHANGED` と無操作

```tla
VARIABLES stock, log

Init == /\\ stock = 1 /\\ log = <<>>
Audit == /\\ log' = Append(log, "audit")
          /\\ UNCHANGED stock
```

`UNCHANGED stock` はその変数が変わらないと宣言する。
アクションで更新しない変数を暗黙に放置すると、意図しない非決定性が入る。

## 6. Spec の書き方

```tla
VARIABLES x, y
vars == <<x, y>>
Init == /\\ x = 0 /\\ y = 0
Step == /\\ x' = x + 1 /\\ y' = y + x'
Spec == Init /\\ [][Step]_vars
```

`vars` に観測対象をまとめる。
変数の集合から漏れた状態変数は stuttering として扱われるので、検査結果を誤解しやすい。

## 7. TLC の回し方

ファイルを保存し、TLA+ Toolbox または VS Code 拡張でモデルを作る。
CLI では `tla2tools.jar` を取得して次のように回せる。

```bash
java -cp tla2tools.jar tlc2.TLC -config Counter.cfg Counter.tla
```

設定例:

```tla
CONSTANT Users = {"alice", "bob"}
INVARIANT TypeOK
PROPERTY Progress
```

成功は「指定した定数範囲と仕様で違反なし」を意味する。
モデルを変更したのに config が古いと、別の問題を検査してしまう。

## 8. PlusCal との関係

PlusCal はプロセスと逐次的な見た目で書き、TLA+ に変換する。
アルゴリズムを説明しやすいが、最終的な意味は生成された TLA+ にある。

```tla
---- MODULE QueuePlusCal ----
EXTENDS Naturals, Sequences
CONSTANT Items
VARIABLE queue

(* --algorithm ProducerConsumer
variables q = <<>>;
begin
  Producer:
    while TRUE do
      q := Append(q, Items[1]);
    end while;
end algorithm; *)
================================
```

PlusCal は人間の制御フローを整理する入口、素の TLA+ は集合や関係を直接書く出口と考える。

## 9. 状態爆発

クライアント数、キュー長、再試行回数を一つ増やすだけで状態数が積になる。

| 対策 | 方法 | 注意 |
| --- | --- | --- |
| 対称性 | 同型のプロセスを同一視 | ID に意味があると不正確 |
| 抽象化 | 金額をカテゴリ化 | 境界バグを落とす |
| 小さい定数 | 2 クライアントから開始 | 一般性は別途説明 |
| 不変条件優先 | liveness を後回し | 進行性を見逃す |
| 履歴を圧縮 | 集合やカウンタにする | 順序を失う |

最小反例を出すことが目的なら、まず 2 プロセス、容量 1 で検査する。

## 10. 反例の読み方

反例の各行で、どのアクションがどの値を変えたかを見る。
「エラーが出た状態」だけでなく、直前の競合順序を読む。

```text
State 1: stock = 1, reserved = {}
Action: Reserve(alice)
State 2: stock = 0, reserved = {alice}
Action: Reserve(bob)
State 3: stock = -1, reserved = {alice, bob}
```

この場合、予約前の `stock > 0` と更新の原子性が抜けている。
実装のロックを議論する前に、仕様の一アクションの粒度を直す。

## 11. Safety のテンプレート

```tla
CONSTANT Capacity, Users
VARIABLES available, held

Init == /\\ available = Capacity /\\ held = [u \in Users |-> 0]
Take(u) == /\\ held[u] < 1
           /\\ available > 0
           /\\ available' = available - 1
           /\\ held' = [held EXCEPT ![u] = @ + 1]
Return(u) == /\\ held[u] = 1
             /\\ available' = available + 1
             /\\ held' = [held EXCEPT ![u] = 0]
Next == \\E u \in Users : Take(u) \\/ Return(u)
TypeOK == /\\ available \in Nat /\\ held \in [Users -> {0, 1}]
Conservation == available + Sum({held[u] : u \in Users}) = Capacity
Spec == Init /\\ [][Next]_<<available, held>>
```

`Conservation` は容量保存を直接検査する。
初期値や `Return` の条件を変えると、TLC が短い反例を返す。

## 12. Liveness と公平性

```tla
VARIABLE turn
Init == turn = "alice"
Next == turn' = IF turn = "alice" THEN "bob" ELSE "alice"
Fairness == WF_<<turn>>(Next)
Spec == Init /\\ [][Next]_turn /\\ Fairness
```

`WF_vars(A)` は、アクションが継続的に有効ならいつか実行される弱公平性である。
公平性を付けると、永遠に選ばれないプロセスを除外できるが、実装が本当に公平である証拠にはならない。

## 13. 仕様の落とし穴

`Next == TRUE` は何でも起きる仕様であり、検査が通っても意味がない。
`UNCHANGED` の漏れは不正な変化を許す。
集合で履歴を表すと重複と順序が消える。
`TypeOK` だけでは業務不変条件を保証しない。
定数を一つだけにすると、競合が消えてしまう。
反例を直す際、実装に合わせて仕様を弱めすぎない。

## 14. 章末のモデル

```tla
---- MODULE TwoPhase ----
CONSTANT Clients
VARIABLE phase

Init == phase = [c \in Clients |-> "ready"]
Prepare(c) == /\\ phase[c] = "ready"
              /\\ phase' = [phase EXCEPT ![c] = "prepared"]
Commit(c) == /\\ phase[c] = "prepared"
             /\\ phase' = [phase EXCEPT ![c] = "committed"]
Next == \\E c \in Clients : Prepare(c) \\/ Commit(c)
TypeOK == phase \in [Clients -> {"ready", "prepared", "committed"}]
NoSkip == \\A c \in Clients : phase[c] # "committed" \/ phase[c] = "committed"
Spec == Init /\\ [][Next]_phase
================================
```

`NoSkip` は弱い例だが、状態列挙の形を示す。
実務では、コミット済みなら準備済みだったという履歴情報もモデルに持たせる。

## 15. 次章へのリンク

Web、ゲーム、物理・数学寄りのモデルは [04-tlaplus-examples.md](./04-tlaplus-examples.md) にある。
Lean の全入力証明との役割分担は [00-overview.md](./00-overview.md) を参照する。
実装の大量入力検査は [05-pbt-basics.md](./05-pbt-basics.md) に進む。

## 16. 演算子の早見表

| 記法 | 意味 |
| --- | --- |
| `=` | 等しい |
| `#` | 異なる |
| `/\\` | かつ |
| `\\/` | または |
| `~` | 否定 |
| `=>` | 含意 |
| `\\A` | 全称 |
| `\\E` | 存在 |
| `'` | 次状態 |
| `[]` | 常に |
| `<>` | いつか |
| `UNCHANGED x` | x は不変 |

バックスラッシュはエディタ上で見落としやすい。
構文エラーと仕様エラーを分けるため、まず小さな式を Toolbox で評価する。

## 17. シーケンス

```tla
EXTENDS Sequences

Init == queue = <<>>
Enqueue(x) == queue' = Append(queue, x)
Dequeue == /\\ Len(queue) > 0
           /\\ queue' = Tail(queue)
QueueType == queue \in Seq(Nat)
```

集合に置き換えると FIFO 性が消える。
順序が保証に必要かを先に決める。

## 18. 関数更新の読解

```tla
VARIABLE table
Put(k, v) == table' = [table EXCEPT ![k] = v]
Increment(k) == table' = [table EXCEPT ![k] = @ + 1]
```

`@` は更新前の値を表す。
存在しないキーへの更新を許すか、`DOMAIN table` の前提を確認する。

## 19. モデルの段階的拡張

```text
段階A: 状態と一アクション
段階B: TypeOK
段階C: Safety 不変条件
段階D: 失敗・再試行
段階E: liveness と公平性
段階F: 実装のタイムアウト・分断
```

最初から全障害を入れると、反例の意味が読めなくなる。

## 20. TLC 実行前チェック

- [ ] 全定数に小さな値を与えた
- [ ] `Init` が実行可能である
- [ ] `Next` に通常遷移がある
- [ ] 更新しない変数に `UNCHANGED` がある
- [ ] `TypeOK` を invariant にした
- [ ] 監視対象の変数を `Spec` に含めた
- [ ] liveness には公平性の意味を書いた
- [ ] 反例の期待サイズを決めた

## 21. 仕様レビュー

仕様を書く人とは別の人が、実装を見ずに仕様を説明できるか確認する。
説明できない `Next` は、実装を写しただけか、抽象化が過剰である。
一アクションがトランザクション境界と対応しているかをレビューする。

## 22. まとめ

TLA+ は記法の習得より、状態の選び方が本体である。
反例を読めるサイズにし、検査結果の前提を保存する。

## 23. 練習問題

次の各問題は、まず日本語で性質を書き、その後に状態とアクションを定義する。

1. 容量 2 のキューが負の長さにならない。
2. 同じ利用者が二つのロックを同時に持たない。
3. 期限切れセッションが更新されない。
4. 二人のゲームで、手番でない人が得点しない。
5. 冪等キーを再送しても残高が二重に増えない。

```text
回答に含めるもの:
Init、Next、TypeOK、主不変条件、定数設定、期待する反例
```

## 24. 仕様を説明する文章

```text
Init は初期 DB の状態を表す。
Next は一回の API 操作を表す。
Spec は停止（stuttering）を含む実行列を表す。
Safety は許されない状態を除外する。
Liveness は公平性という仮定の下で進行を要求する。
```

この文章を仕様ファイルのコメントに残すと、数か月後もモデルの粒度を理解できる。

## 25. 次の章

具体的な在庫、セッション、ゲーム、保存量のモデルは [04-tlaplus-examples.md](./04-tlaplus-examples.md) で確認する。

## 26. モデルを小さく保つ

定数を増やす前に、既存の反例がどの性質を示しているかを分類する。
同じ性質を示す対称な状態は、対称性削減の候補になる。
ただし利用者 ID が監査や権限に意味を持つなら、削減してはいけない。

## 27. Safety の読解練習

```tla
Inv == count \in Nat /\\ count <= Limit
```

この式が保証するのは現在状態の範囲だけである。
`Next` が必ず有限時間で起きることや、実装の応答が速いことは保証しない。

## 28. Liveness の読解練習

```tla
EventuallyDone == <> (status = "done")
```

この式は、すべての実行でいつか done に到達することを要求する。
失敗時の無限リトライを許すなら、別の進行性や上限をモデルに追加する。

## 29. 仕様と実装の差

モデルの操作が DB トランザクションなら、実装の複数 SQL が同じ atomicity を持つか確認する。
モデルの時計が理想化されているなら、実装の時計跳躍を PBT または統合テストへ渡す。

## 30. まとめ直し

Init は「どこから始まるか」、Next は「何が起きるか」、不変条件は「何を壊さないか」、liveness は「何が進むか」を分担する。

この四つを混ぜずにレビューする。

モデルの変更には理由を添える。

検査の成功は範囲付きで報告する。

それが再現可能な仕様作業の基本である。

次章では実際の題材へ進む。

実例で反例を読む。
