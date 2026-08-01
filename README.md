# books

技術書・教科書の書棚。ブラウザでそのまま読める HTML で置いている。

## 📖 読む

**https://coil398.github.io/books/**

ローカルで読むなら:

```sh
open index.html
```

## 蔵書

| 書名 | 内容 |
|---|---|
| [形式手法とテスト技法](./formal-methods/) | Lean 4 / TLA+ / PlusCal / Property-Based Testing。全 8 章 |
| [Pi SDK と LangChain](./pi-sdk-langchain/) | Pi SDK / LangChain / LangGraph / エージェントループ / 実装・選定・運用。全 8 章 |

## 作りの方針

- **1 冊 1 ディレクトリ**。`<book>/index.html` が目次、以降が各章
- **単一ファイル完結・外部依存ゼロ**。CDN も外部フォントも参照しないので、オフラインでも壊れない
- 各書に備わるもの: 左サイドバー固定の目次（スクロール追従）、コードブロック、コピーボタン、ライト/ダークのトグル（`localStorage` 保存）、章末の理解度チェック
- シンタックスハイライトは必要な書籍で自前実装する

## ライセンス・注意

内容は生成 AI を用いて執筆したもの。実務で使う前に一次情報での裏取りを推奨する。
