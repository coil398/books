# books

技術書・教科書の置き場。Obsidian の vault としてそのまま開ける。

## 蔵書

| ディレクトリ | 内容 |
|---|---|
| [`formal-methods/`](./formal-methods/) | 形式手法とテスト技法の教科書（Lean 4 / TLA+ / Property-Based Testing） |

## 読み方

### Obsidian で読む

このリポジトリのルートを vault として開く。`.md` がノートとして読める。

- 目次サイドバー → Outline コアプラグイン（見出しから自動生成）
- コードブロックのコピー → Reading View に標準搭載
- テーマ切替 → Settings → Appearance（アプリ全体設定）

> ⚠️ Obsidian は `.html` をノートとして開けない（[公式のサポート形式](https://obsidian.md/help/Files+and+folders/Accepted+file+formats)に含まれない）。HTML 版を Obsidian 内で読みたい場合は Local HTML Browser 等のコミュニティプラグインが要る。

### ブラウザで読む

各書の `html/index.html` を開く。単一ファイル完結で外部依存ゼロなので、オフラインでも動く。

```sh
open formal-methods/html/index.html
```

HTML 版のほうが機能が多い:

- 左サイドバー固定の目次（スクロール追従）
- 行番号付きコードブロック + コピーボタン + 言語ラベル
- **Lean 4 / TLA+ / PlusCal の自前シンタックスハイライト**（Obsidian の PrismJS はどちらも非対応）
- ライト/ダークのページ内トグル（`localStorage` 保存）
- 章末の理解度チェック（開閉式）

## 構成の方針

- 1 冊 1 ディレクトリ
- `<book>/*.md` が本文、`<book>/html/` がブラウザ版
- `.obsidian/` は個人の環境設定なので追跡しない
