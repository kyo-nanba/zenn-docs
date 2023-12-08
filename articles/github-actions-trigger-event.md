---
title: "GitHub Actionsのトリガーイベントでハマったので調べた話"
emoji: "📝"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["githubactions"]
published: true
published_at: 2023-12-09 00:00
---

これは[Timee Advent Calendar 2023](https://qiita.com/advent-calendar/2023/timee)シリーズ1の9日目の記事です。

https://qiita.com/advent-calendar/2023/timee

## はじめに

こんにちは、タイミーでバックエンドエンジニアをしている難波（[@kyo_nanba](https://twitter.com/kyo_nanba)）です。

Timee Advent Calendar 2023の1日目に投稿された「[タイミーのRailsアプリをシニアなエンジニアが採点したらだいぶ辛口だった](https://tech.timee.co.jp/entry/2023/12/01/100000)」という記事の対談に私も参加したのですが、この記事が思いの外多くの方に読んで頂けたようでありがとうございます。それをみて私も当初予定していたGitHub Actionsの具体的なHowの話では無くプロダクト開発のメタ的な話にしようかなと思いましたが、二匹目のドジョウを狙おうとして急に内容を変えるのもアレなので当初の予定通りの内容でいきます。

## 発生した事象

タイミーはソースコードのホスティングにGitHubを利用しており、GitHub Actionsも日常的に使用しています。そこで最近 "特定のファイルが編集された場合に実行してほしい" Actionを作成しました。

その時のAction設定ファイルは大まかには下記のようなものでした。

```yaml
name: CI

on:
  push:
    paths:
      - 'README.md'

jobs:
  run:
    runs-on: ubuntu-latest
    steps:
      - run: echo "sample action"
```

期待している動作としては「README.mdを編集したコミットがpushされた場合にActionが動く」といったものになります。実際に動かしてみたところ期待している動作が確認できたので、しばらく運用していました。

それからしばらく経ったある日になんとなくGitHub Actionsの実行ログページを見ていると、このActionが想定の数倍ほど実行されていることに気づきました。

## 何が起きていたのか

タイミーではデプロイフローの中で `git tag` を使用しています（デプロイフローそのものは今回の趣旨と異なるので説明は省きます）。今回発生した事象について調査したところ「 `git tag` がGitHubにpushされた際に README.md が編集されているか否かに関わらずActionが実行される」ことが分かりました。

## 何故そうなったのか

https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions

つまるところオフィシャルのドキュメントをよく読みましょうという話になります。

まず一点目、下記引用部分に今回の事象に該当する内容が書いてありました。

> If you define neither `tags` / `tags-ignore` or `branches` / `branches-ignore`, the workflow will run for events affecting either branches or tags.

私なりに訳すと「 `tags` / `tags-ignore` もしくは `branches` / `branches-ignore` のどちらも指定していない場合、ブランチまたはタグに影響するイベントでワークフローは実行されるよ」という感じです。

そしてもう一箇所 `paths` / `paths-ignore` についての説明の中で重要な箇所がありました。

> When using the push and pull_request events, you can configure a workflow to run based on what file paths are changed. Path filters are not evaluated for pushes of tags.

後半の一文に書いてあるようにPath filtersはtagのpushでは評価されないのです。

上記2つの仕様により任意のtagがpushされるとpathsに指定していたファイルが編集されていたか否かに関係なくActionが実行されることになります。前述の通りタイミーではデプロイフローで `git tag` を使用しているためデプロイのたびに当該Actionが実行されるという事象が発生したという流れになります。

## 対策

対策は単純です。 `tags` / `tags-ignore` / `branches` / `branches-ignore` のいずれかを指定しましょう。仕様として「4つのうちいずれも指定していない場合にブランチまたはタグに影響するイベントでワークフローは実行される」ため、なにか指定してあれば大丈夫です。今回のActionは主にPull Requestの際に実行されることを目的とした処理だったため `branches-ignore` にmainブランチを指定することで回避しました。

## まとめ

今回は弊社でGitHub Actionsを使っていて実際に遭遇した事象について紹介しました。GitHub Actionsは一定以上利用すると従量課金になるため、無駄な処理は行わないようにすることがコスト削減に繋がります。今回はドキュメントをしっかり読んで理解していれば未然に防げたものではありますが、こういうのは中々気づきにくいですね。
