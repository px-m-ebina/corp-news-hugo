# デプロイ手順

コーポレートサイトの開発・公開の流れをまとめたものです。

- `static/` 配下(トップページなどのHTML/CSS)は外注先が作成します。
- ニュース部分は自社がHugoで作成します。

## 1. 環境

| 環境 | ブランチ | 公開先 | デプロイの仕組み |
|---|---|---|---|
| 確認環境 | `develop` | GitHub Pages | GitHub Actions(`.github/workflows/deploy.yml`) |
| 本番 | `main` | Cloudflare Workers(現在は workers.dev のURL) | Cloudflare の Git 連携ビルド(`wrangler.jsonc`) |

`develop` に push(PR のマージを含む)すると、GitHub Actions が Hugo のビルドと GitHub Pages へのデプロイを自動で実行します。`main` も同様に、push すると Cloudflare が Hugo のビルドとデプロイを自動で実行します。手動で作業する必要はありません。

```
main ─────────────●────────────  本番 → Cloudflare Workers
                  ↑ PR(リリース)
develop ──●───●───●────────────  確認 → GitHub Pages
          ↑PR ↑PR
          │   └─ feature/news-xxx    自社:content/ と layouts/
          └─ feature/static-xxx      外注:static/ のみ
```

> GitHub Pages で公開できるのは 1 リポジトリにつき 1 サイトだけです。そのため GitHub Pages は develop 専用にしています。main に push しても GitHub Pages にはデプロイされません。

## 2. 担当範囲

| 担当 | 変更してよい範囲 |
|---|---|
| 外注先 | `static/` 配下のみ |
| 自社 | `content/`、`layouts/`、`archetypes/`、`hugo.toml`、`.github/`、`wrangler.jsonc` |

### 外注先に守っていただくルール

1. **`static/news/` にはファイルを置かないでください。**
   `news/` 以下は Hugo が自動で生成します。同じ場所にファイルを置くと、どちらかが上書きされます。
2. **パスは相対パスで書いてください。先頭に `/` は付けません。**
   確認環境は `https://<ユーザー名>.github.io/corp-news-hugo/` のように、ドメイン直下ではない場所で公開されるためです。

   | NG | OK |
   |---|---|
   | `href="/css/style.css"` | `href="css/style.css"` |
   | `href="/"` | `href="./"` |
   | `href="/about.html"` | `href="about.html"` |
   | `href="/news/"` | `href="news/"` |
   | `fetch('/news/index.json')` | `fetch('news/index.json')` |

   相対パスは「そのファイルの場所から見たパス」です。一段深いフォルダのページ(例:`static/company/history.html`)では `../css/style.css` のように書きます。
3. **トップページのニュース欄は、`news/index.json` から最新 5 件を読み込んで表示します。**
   JSON の各項目は `title`、`date`(`2006.01.02` 形式)、`category`、`url` です。

### 自社(Hugoテンプレート)のルール

- テンプレート内のリンクは `{{ "news/" | relURL }}` のように書きます。先頭に `/` は付けません。付けると、`/corp-news-hugo/` のようなサブディレクトリが URL に付かなくなります。

## 3. 作業の流れ

| # | 担当 | 作業 | 結果 |
|---|---|---|---|
| 1 | 外注 | `develop` から `feature/static-xxx` ブランチを作り、`static/` を作成する(ニュース部分は枠のみ)。`develop` 向けに PR を出す | ― |
| 2 | 自社 | PR を確認し、`develop` にマージする | GitHub Pages に自動デプロイ → 表示を確認 |
| 3 | 自社 | 最新の `develop` から `feature/news-xxx` ブランチを作り、ニュース記事を追加する(手順は 4 章)。`develop` 向けに PR を出す | ― |
| 4 | 自社 | `develop` にマージする | GitHub Pages で確認 |
| 5 | 自社 | `develop` から `main` へ PR を出してマージする(リリース) | Cloudflare で自動ビルド → 本番に公開 |

- `develop` と `main` には直接 push せず、必ず PR を通してマージします。
- 作業ブランチを作る前に、`git pull` で `develop` を最新にしてください。

## 4. ニュース記事の追加手順

```bash
git switch develop && git pull
git switch -c feature/news-2026-12-01-xxx

# 記事を作成(テンプレート archetypes/news.md が使われる)
hugo new content news/2026-12-01-xxx.md
```

作成した `content/news/2026-12-01-xxx.md` を編集します。

```toml
+++
title = '記事タイトル'
date = '2026-12-01T10:00:00+09:00'
draft = false          # 公開するときは false にする
category = 'お知らせ'
description = '一覧や検索結果に表示する説明文'
+++
本文(Markdown)
```

```bash
# ローカルで確認(-D を付けると下書きも表示される)
hugo server -D
# → http://localhost:1313/news/ で確認

git add content/news/
git commit -m "news: 〇〇のお知らせを追加"
git push -u origin feature/news-2026-12-01-xxx
# → GitHub で develop 向けに PR を作成
```

**公開されない記事があるときは、次の 2 点を確認してください。**
- `draft = true` の記事は、確認環境にも本番にも公開されません。
- `date` が未来の日時の記事も公開されません。

`hugo server -D` では表示されるのに確認環境や本番に出ない場合は、上のどちらかが原因です。

## 5. デプロイの設定

### 確認環境(GitHub Pages)

- 設定ファイル:`.github/workflows/deploy.yml`
- `develop` に push(PR のマージを含む)すると、Hugo のビルドと GitHub Pages へのデプロイを自動で実行します。
- 手動で実行(workflow_dispatch)する場合も、`develop` を選んだときだけ動きます。`main` などほかのブランチを選んでもデプロイされません。
- ビルドの内容:Hugo v0.169.0(extended)で `--environment preview` を指定し、GitHub Pages の URL を `--baseURL` に渡しています。
- preview 環境では `<meta name="robots" content="noindex">` が出力され、検索エンジンに登録されません。

### 本番(Cloudflare Workers)

- 設定ファイル:`wrangler.jsonc`(Worker 名:`corp-news-hugo`、公開フォルダ:`./public`、workers.dev で公開、Preview URL は無効)
- `main` に push(PR のマージを含む)すると、Cloudflare が Hugo のビルドと Workers へのデプロイを自動で実行します。
- `public/` はリポジトリに含めていません。Cloudflare 側でビルドしてから公開します。

| 項目 | 設定値 |
|---|---|
| 本番ブランチ | `main` |
| ビルドコマンド | 下記 |
| デプロイコマンド | `npx wrangler deploy` |
| main 以外のブランチのビルド | 無効(確認は GitHub Pages で行うため) |

ビルドコマンド:

```bash
curl -sL https://github.com/gohugoio/hugo/releases/download/v0.169.0/hugo_0.169.0_linux-amd64.tar.gz | tar xz hugo && ./hugo --gc --minify --environment preview --baseURL "https://corp-news-hugo.<サブドメイン>.workers.dev/"
```

- 本番ドメインに切り替えるまでは `--environment preview` にしておきます(noindex を出力するため)。
- `--baseURL` を指定しないと `hugo.toml` の仮の値(`https://www.example.co.jp/`)が使われます。その場合、RSS・sitemap・`news/page/1/`(リダイレクト用ページ)の URL が誤ったものになります。
- Hugo は通常版を使っています。SCSS を導入する場合は `hugo_extended_0.169.0_linux-amd64.tar.gz` に変えてください。
- Hugo のバージョンを上げるときは、`deploy.yml` の `HUGO_VERSION` とこのビルドコマンドの両方を揃えて変更してください。

## 6. 本番ドメインへの切り替え手順(将来)

1. `hugo.toml` の `baseURL` を本番ドメインに変更する。
2. Cloudflare のビルドコマンドを `--environment production` に変え、`--baseURL` を外す。これで noindex タグが外れ、検索エンジンに登録されるようになります。
3. Cloudflare で独自ドメインを Worker に割り当てる。
4. `static/robots.txt` を用意する場合は、このタイミングで本番用の内容にする。

## 7. 未対応事項

- [ ] 本番(`main`)に入っている確認用の表記「from dev 14:30」(`static/index.html` の title・ロゴ・フッター)を元に戻す
- [ ] Cloudflare のビルドコマンドに `--baseURL` を追加する
- [ ] `develop` と `main` にブランチ保護を設定する(PR 経由のマージに限定)
