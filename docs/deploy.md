# デプロイ手順

コーポレートサイトの開発・公開の流れをまとめたものです。

- `static/` 配下(トップページなどのHTML/CSS)は外注先が作成します。
- ニュース部分は自社がHugoで作成します。

## 1. 環境

確認環境と本番は、どちらも Cloudflare Workers で公開します。GitHub Pages と GitHub Actions は使いません。

| 環境 | ブランチ | 公開先 |
|---|---|---|
| 確認環境 | `develop` | Cloudflare の Preview URL(`develop-corp-news-hugo.<サブドメイン>.workers.dev`) |
| 本番 | `main` | Cloudflare の本番URL(現在は `corp-news-hugo.<サブドメイン>.workers.dev`) |

`develop` または `main` に push(PR のマージを含む)すると、Cloudflare が Hugo のビルドとデプロイを自動で実行します。手動で作業する必要はありません(詳しい流れは 5 章)。

```
main ─────────────●────────────  本番 → Cloudflare(本番URL)
                  ↑ PR(リリース)
develop ──●───●───●────────────  確認 → Cloudflare(Preview URL)
          ↑PR ↑PR
          │   └─ feature/news-xxx    自社:content/ と layouts/
          └─ feature/static-xxx      外注:static/ のみ
```

## 2. 担当範囲

| 担当 | 変更してよい範囲 |
|---|---|
| 外注先 | `static/` 配下のみ |
| 自社 | `content/`、`layouts/`、`archetypes/`、`hugo.toml`、`wrangler.jsonc` |

### 外注先に守っていただくルール

`static/` 配下のフォルダ構成は、基本的に自由です。`static/` 配下のファイルは、ビルドでそのままサイトのルートにコピーされます(例:`static/company/index.html` は `/company/` で表示されます)。ただし、次のルールを守ってください。

1. **Hugo が生成するパスと同じ場所にはファイルを置かないでください。**
   同じ場所にファイルがあると、どちらかが上書きされます。

   | 置いてはいけない場所 | 理由 |
   |---|---|
   | `static/news/` | ニュース一覧・記事・`index.json`・`index.xml` を Hugo が生成するため |
   | `static/sitemap.xml` | Hugo が生成するため |
   | `static/tags/`、`static/categories/` | 将来、タグやカテゴリのページで使う予定のため |

2. **Cloudflare が特別な意味で扱うファイルは置かないでください。**
   `_headers`(HTTP ヘッダーの設定)、`_redirects`(リダイレクトの設定)、`.assetsignore`(アップロード対象から外す設定)は、Cloudflare が設定ファイルとして読み込みます。必要なときは自社で管理します。
3. **ページの構成(URL の一覧)は事前に共有し、変更するときは連絡してください。**
   ヘッダーのナビゲーションとフッターは、`static/index.html` などの HTML と、ニュースページ用の Hugo テンプレート(`layouts/_partials/header.html`、`footer.html`)の 2 か所にあります。ページの場所を変えると(例:`about.html` を `company/` に移す)、ニュースページのナビのリンクが切れます。ナビのメニューやデザインを変えたときも、自社で Hugo 側を合わせて修正します。
4. **フォルダ名とファイル名には、半角の英小文字、数字、ハイフン(`-`)だけを使ってください。**
   日本語、スペース、大文字は使いません。URL は大文字と小文字を区別するため、`About.html` と `about.html` は別のページになります。
5. **1 ファイルあたり 25MiB までにしてください。**
   Cloudflare Workers の上限です。ファイル数にも上限があります(無料プランで 20,000 件)。動画などの大きなファイルは、別のサービスに置いてください。
6. **パスは相対パスで書いてください。先頭に `/` は付けません。**
   公開先の URL 構成が将来変わっても、リンクが壊れないようにするためです。

   | NG | OK |
   |---|---|
   | `href="/css/style.css"` | `href="css/style.css"` |
   | `href="/"` | `href="./"` |
   | `href="/about.html"` | `href="about.html"` |
   | `href="/news/"` | `href="news/"` |
   | `fetch('/news/index.json')` | `fetch('news/index.json')` |

   相対パスは「そのファイルの場所から見たパス」です。一段深いフォルダのページ(例:`static/company/history.html`)では `../css/style.css` のように書きます。
7. **トップページのニュース欄は、`news/index.json` から最新 5 件を読み込んで表示します。**
   JSON の各項目は `title`、`date`(`2006.01.02` 形式)、`category`、`url` です。

#### フォルダ構成の例

```
static/
├── index.html
├── about.html           (または company/index.html)
├── contact.html
├── css/
├── js/
├── images/
└── company/
    └── history.html     → CSS は ../css/style.css で読み込む
```

### 自社(Hugoテンプレート)のルール

- テンプレート内のリンクは `{{ "news/" | relURL }}` のように書きます。先頭に `/` は付けません。こう書いておけば、`baseURL` にサブディレクトリを含めた場合でもリンクが正しく出力されます。

## 3. 作業の流れ

| # | 担当 | 作業 | 結果 |
|---|---|---|---|
| 1 | 外注 | `develop` から `feature/static-xxx` ブランチを作り、`static/` を作成する(ニュース部分は枠のみ)。`develop` 向けに PR を出す | ― |
| 2 | 自社 | PR を確認し、`develop` にマージする | Cloudflare で自動ビルド → Preview URL で表示を確認 |
| 3 | 自社 | 最新の `develop` から `feature/news-xxx` ブランチを作り、ニュース記事を追加する(手順は 4 章)。`develop` 向けに PR を出す | ― |
| 4 | 自社 | `develop` にマージする | Preview URL で確認 |
| 5 | 自社 | `develop` から `main` へ PR を出してマージする(リリース) | Cloudflare で自動ビルド → 本番URLに公開 |

- `develop` と `main` には直接 push せず、必ず PR を通してマージします。
- 作業ブランチを作る前に、`git pull` で `develop` を最新にしてください。
- `feature/*` ブランチを push した場合も、Cloudflare がそのブランチ用の Preview URL を発行します。PR を出す前の確認に使えます。

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

## 5. デプロイの設定(Cloudflare Workers & Pages)

### ビルドとデプロイの流れ

ビルドはすべて Cloudflare のサーバーで行われます。GitHub 側(GitHub Actions)では何も実行されません。

```
GitHub に push(develop / main / feature/*)
   ↓ Cloudflare がpushを検知
① リポジトリを取得(clone)
② ビルドコマンドを実行
   - Hugo v0.169.0 をダウンロード
   - ./hugo --gc --minify --environment preview … で public/ を生成
③ デプロイコマンドを実行
   - main          → npx wrangler deploy           → 本番URLに公開
   - main 以外     → npx wrangler versions upload  → Preview URLを発行
     ※「main 以外のブランチのビルド」が有効な場合
```

- ③の wrangler は `wrangler.jsonc` を読み込み、`assets.directory` に指定した `./public` をアップロードします。
- 手元で `hugo` を実行してビルドする必要はありません。`hugo server -D` は表示の確認にだけ使います。
- `public/` は `.gitignore` で除外しているので、push には含まれません。Cloudflare が毎回ビルドして作ります。
- ビルドが行われたかどうかは、Cloudflare の管理画面で該当する Worker を開き、Deployments(または Builds)のビルドログで確認できます。ログに `./hugo --gc --minify …` の実行結果が出ていれば、Cloudflare でビルドされています。

設定は、リポジトリ内の `wrangler.jsonc` と Cloudflare の管理画面の 2 か所にあります。

### リポジトリ側:`wrangler.jsonc`

| 項目 | 値 | 意味 |
|---|---|---|
| `name` | `corp-news-hugo` | Worker 名(管理画面の Project name と一致させる) |
| `assets.directory` | `./public` | Hugo の出力先を公開する |
| `workers_dev` | `true` | workers.dev の本番URLで公開する |
| `preview_urls` | `true` | main 以外のブランチ用に Preview URL を発行する |

`public/` はリポジトリに含めていません。Cloudflare 側でビルドしてから公開します。

### Cloudflare の管理画面(Settings → Build)

| 項目 | 設定値 |
|---|---|
| Git リポジトリ | `px-m-ebina/corp-news-hugo` |
| 本番ブランチ | `main` |
| ビルドコマンド | 下記(全ブランチ共通) |
| デプロイコマンド(main) | `npx wrangler deploy` |
| main 以外のブランチのビルド | **有効**にする |
| main 以外のブランチのデプロイコマンド | `npx wrangler versions upload` |

ビルドコマンド:

```bash
curl -sL https://github.com/gohugoio/hugo/releases/download/v0.169.0/hugo_0.169.0_linux-amd64.tar.gz | tar xz hugo && ./hugo --gc --minify --environment preview --baseURL "https://corp-news-hugo.<サブドメイン>.workers.dev/"
```

- 本番ドメインに切り替えるまでは `--environment preview` にしておきます。preview 環境では `<meta name="robots" content="noindex">` が出力され、検索エンジンに登録されません。
- ビルドコマンドは全ブランチで共通です。そのため Preview URL でも `--baseURL` には本番URLが入ります。ページ内のリンクは相対パスなので問題なく動きます。ただし RSS・sitemap・`news/page/1/`(リダイレクト用ページ)は本番URLを指します。
- `--baseURL` を指定しないと `hugo.toml` の仮の値(`https://www.example.co.jp/`)が使われます。
- Hugo は通常版を使っています。SCSS を導入する場合は `hugo_extended_0.169.0_linux-amd64.tar.gz` に変えてください。
- Preview URL は、URL を知っていれば誰でも閲覧できます。関係者以外に見せたくない場合は、Cloudflare Access で保護してください。

## 6. 本番ドメインへの切り替え手順(将来)

1. `hugo.toml` の `baseURL` を本番ドメインに変更する。
2. ビルドコマンドから `--baseURL` を外す。
3. 本番だけ `--environment production` でビルドされるよう、ビルドコマンドを分岐させる。ビルドコマンドは全ブランチ共通なので、ブランチ名で判定します。

   ```bash
   ... && if [ "$WORKERS_CI_BRANCH" = "main" ]; then ENV=production; else ENV=preview; fi && ./hugo --gc --minify --environment $ENV
   ```

   これで本番では noindex タグが外れ、検索エンジンに登録されるようになります。
4. Cloudflare で独自ドメインを Worker に割り当てる。
5. `static/robots.txt` を用意する場合は、このタイミングで本番用の内容にする。

## 7. 未対応事項

- [ ] 本番(`main`)に入っている確認用の表記「from dev …」(`static/index.html` の title・ロゴ・フッター)を元に戻す
- [ ] Cloudflare の管理画面で「main 以外のブランチのビルド」を有効にする
- [ ] Cloudflare のビルドコマンドに `--baseURL` を追加する
- [ ] GitHub のリポジトリ設定(Settings → Pages)で GitHub Pages を無効にする
- [ ] `develop` と `main` にブランチ保護を設定する(PR 経由のマージに限定)
