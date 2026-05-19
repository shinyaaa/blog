# CLAUDE.md

## このリポジトリについて

[Zenn](https://zenn.dev/) で公開しているブログ（日本語）のソースを管理するリポジトリ。
あわせて、その日本語記事を英訳した [dev.to](https://dev.to/) 向けの記事も管理する。

- **日本語版（Zenn）**: 一次ソース。`articles/`・`books/` に置く。`zenn-cli` で管理。
- **英語版（dev.to）**: 日本語記事を英訳したもの。`dev.to/` に置く。

## ディレクトリ構成

| パス        | 内容 |
|-------------|------|
| `articles/` | Zenn の記事（日本語）。Markdown 1ファイル = 1記事。 |
| `books/`    | Zenn の本（日本語）。 |
| `dev.to/`   | dev.to 向けの記事（英語）。`articles/` の記事を英訳したもの。 |
| `images/`   | 記事に埋め込む画像・図版（`.drawio` の書き出し PNG など）。 |

## ワークフロー

1. 日本語記事を `articles/` に作成・更新する（Zenn が一次ソース）。
2. その記事を英訳し、dev.to 形式のフロントマターに変換して `dev.to/` に保存する。
3. 日本語記事を更新したら、対応する英語記事も追従して更新する。

## ファイル命名

- `articles/` … Zenn の slug をそのままファイル名にする（`zenn-cli` 生成のランダム英数字、または任意の slug）。
- `dev.to/` … 内容が分かる英語の slug を付ける（例: `postgresql-lock-tips.md`）。
  対応する日本語記事と対応づけられる名前にしておくと管理しやすい。

## フロントマターの違い

英訳時は Zenn 形式から dev.to 形式へフロントマターを書き換える。

### Zenn（`articles/`）

```yaml
---
title: "PostgreSQLのロックでハマりがちな挙動5選"
emoji: "🐘"
type: "tech"          # tech / idea
topics:
  - "postgresql"
  - "database"
published: false
published_at: "2023-04-24 20:12"   # 公開済みの場合
---
```

### dev.to（`dev.to/`）

```yaml
---
title: "5 PostgreSQL Locking Behaviors That Trip People Up"
published_at: 2023-04-24          # Zenn の公開日（YYYY-MM-DD）を引き継ぐ
tags: postgresql, database
---
```

- `topics`（Zenn）→ `tags`（dev.to、カンマ区切り・最大4個）。
- `published_at`（Zenn）→ `published_at`（dev.to）。日付部分のみ `YYYY-MM-DD` 形式で引き継ぐ。
- `emoji` / `type` は dev.to には無いので削除する。
- 下書きは `published_at` の代わりに `published: false` を指定する（`published` を省略した場合も下書き扱い）。
- `description` / `cover_image` は任意。書く場合のみ追加する。

## 執筆時の注意

- 英訳は逐語訳ではなく、英語圏の読者に自然に読める文章にする。
- コードブロック・コマンド・出力例はそのまま残す（翻訳しない）。
- 画像は `images/` のものを共有して参照する。
