# bitcoin-notes

Bitcoinについて調査・検証した内容を整理し、Zennで公開するためのリポジトリです。

## 基本方針

- 調査中の内容は `research/`、公開する文章は `articles/` に置く
- 可能な限り一次資料を参照する
- 事実、解釈、意見を区別する
- バージョンや確認日を残し、情報の有効範囲を明確にする
- 新しい記事は `published: false` から始める

## ディレクトリ構成

```text
.
├── articles/                 # Zennの記事
├── books/                    # Zennの本
├── images/                   # 記事で使用する画像
│   └── <article-slug>/
├── research/                 # 公開前の調査メモ
│   ├── economics/
│   ├── history/
│   ├── lightning/
│   ├── mining/
│   ├── protocol/
│   └── security/
├── sources/                  # 参考資料と用語の索引
├── templates/                # 記事・調査メモのひな形
└── .github/                  # GitHubの設定
```

## セットアップ

Node.jsをインストールした環境で、次を実行します。

```bash
npm install
```

## よく使うコマンド

```bash
# ローカルプレビュー
npm run preview

# 新しい記事を作成
npm run new:article -- --slug bitcoin-example

# 新しい本を作成
npm run new:book -- --slug bitcoin-example
```

記事を作成したら、`templates/article.md` を参考に本文を組み立てます。調査から始める場合は `templates/research-note.md` をコピーして使います。

## 執筆から公開まで

1. `research/` に疑問、仮説、出典、検証結果を記録する
2. 一次資料を優先して内容を検証する
3. `articles/` に `published: false` の記事を作る
4. `npm run preview` で表示を確認する
5. Pull Requestで内容と出典を確認する
6. 公開時に `published: true` へ変更し、Zenn連携対象ブランチへマージする

詳しいルールは [CONTRIBUTING.md](./CONTRIBUTING.md) を参照してください。

## 注意

このリポジトリの内容は技術的・教育的な情報の整理を目的とし、投資助言ではありません。秘密鍵、シードフレーズ、実環境の認証情報は絶対に記録しないでください。
