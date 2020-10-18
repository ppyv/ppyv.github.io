---
title: "Hugo と GitHub Pages と Actions"
description: "このサイトの組み立て方"
slug: "ghpages"
date: 2020-10-18T12:04:30+09:00
tags: ['github','hugo']
draft: true
---

[最初のポスト]({{< relref "/post/20201011-init/index.md" >}}) とタイトルがほぼかぶっているが、
このポストでは具体的にどのようにこのサイトを組み立てたかを記録する。

# 手順
## Repository の準備
GitHub Pages の個人ページとして立ち上げることを考えていたため、 ppyv.github.io の名を持つリポジトリを立ち上げる。
リポジトリのページであれば名前はなんでもよい（ `http://<username>.github.io/<reponame>` となる）。

## Hugo の準備
このセクションの内容は [Hugo Quick Start](https://gohugo.io/getting-started/quick-start/) と大差がない。

### ルートディレクトリの初期化
`hugo new site ${sitename}` で立ち上げられるが、そのときの挙動が
「カレントディレクトリの下に、 `${sitename}` を持つディレクトリを掘って、必要なアセットを置く」となっている。
このとき `${sitename}` と同じ名前を持つディレクトリがあるとコマンドが失敗終了する。

前の手順で、 GitHub.com や GitHub Desktop などで Create repository してから Clone してくる方式をとる場合、
この制限に引っかかる。 `--force` オプションをつけるとこの制限を回避できる。

### テーマを引っ張ってくる
テーマがないとレンダリングがまともにできないので、 [Hugo Themes](https://themes.gohugo.io/) などから探してくる。

見つかったらルートディレクトリの下にある `themes` に追加する。
わたしは Git のサブモジュール機能を使ったが、単に Clone してくるだけでもよいだろう。

今回利用したのは [Hugo Theme Stack](https://github.com/CaiJimmy/hugo-theme-stack) である (Thanks Jimmy!) 。

`config.toml` に `theme = hugo-theme-stack` などと指定するのも忘れずに。

### (Optional) config.toml のチューニング
テーマには exampleSite が存在することが多い。そこから config.toml を引っ張ってくることで、設定を煮詰める土台として使うことができる。

### 記事を書く
割愛。

## GitHub Actions の準備
[GitHub Actions](https://github.co.jp/features/actions) は GitHub.com 備え付けのワークフロー自動化ツールである。
ありがたいことにパブリックリポジトリでは無料で使うことができ、プライベートリポジトリでも無料枠が用意されている。

イメージとしては GitHub 上で動く VM 上で何かさせるものではあるが、 GitHub 上に「プリセット」のようなものがいくつか存在する。
今回は次の 2 つを利用した。

- [GitHub Actions for Hugo](https://github.com/peaceiris/actions-hugo)
- [GitHub Actions for GitHub Pages](https://github.com/peaceiris/actions-gh-pages)

それぞれ Hugo のビルドを行う、ビルドされた HTML などを GitHub Pages のお作法に基づいてデプロイする Action である (Thanks peaceiris-san!) 。

`/.github/workflows/` ディレクトリ以下に YAML でワークフローを記載していく。
今回は `gh-pages.yml` ファイルに記述している。ファイル名はなんでもよい。
2020/10/18 時点でコミットしてあるファイルは次の内容になっている。

```yaml
---
name: Build sites

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-18.04
    steps:
      - name: Checkout
        uses: actions/checkout@v2
        with:
          submodules: true
          fetch-depth: 0

      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v2
        with:
          hugo-version: 'latest'
          extended: true

      - name: Build
        run: hugo --minify

      - name: deploy
        uses: peaceiris/actions-gh-pages@v3
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./public
          cname: blog.kbys.info
```

どんな環境でも大方これで稼働する[^1]が、最終行の `cname` は環境によって値が異なる[^2]。
これは GitHub Pages を独自ドメインで運用する場合に必要なパラメーターであり、必要がなければ行ごと削除する。

また on セクションのブランチ名は環境によって異なる。デフォルトブランチ名が main の人はこのままでよいし、 master の人は master にしなければならない。
このセクションは「main ブランチにプッシュがあったとき、このワークフローが発火する」ことを意味している。
ここをいじることで、異なるタイミング（Pull request が飛んできたとき、など）に発火するワークフローも組める。
例えば Lint して、その結果を PR 側に表示させることもできる。

## GitHub Pages の設定
GitHub のリポジトリページで Settings を開くと、下の方に GitHub Pages のセクションがある。

ここで次のように設定する。
- Source: Branch を `None` から `gh-pages` に、 Folder を `/ (root)` にして Save する。

これで画面上に表示された URL から成果物にアクセスできるようになった。

# 日頃の運用
記事を書いて master (main) ブランチに Commit & Push するだけで、自動的にビルド＆デプロイが行われる。

# まとめ
Hugo と GitHub Actions, GitHub Pages を組み合わせることで、簡易に Web サイトを構築・提供する方法を説明した。


[^1]: もっともこのコンフィグも GitHub Actions for Hugo リポジトリの README から引っ張ってきたものと本質的な差がない。
[^2]: 最初この設定が漏れていて、ワークフローが発火して走行するたびに独自ドメインが 404 になる事象があったことは内緒である。 GitHub Pages がカスタムドメインを「公開フォルダのルートにある、 CNAME ファイルに記録している」仕様を把握していなかったことによる凡ミスだった。
