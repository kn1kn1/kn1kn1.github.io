---
layout: post
title:  hatenablogからgithub.ioへ移行
---

2009年から使用してきたはてなブログ(はてなダイアリー)からGitHub Pagesに移行しました。

はてなブログのエクスポートで出力されるのはMovable Type形式なのですが、jekyll-importのMovable Type Importerはファイルをサポートしていないようなので、[RSS Importer](http://import.jekyllrb.com/docs/rss/)を使用しました。

```shell
$ curl http://kn1kn1.hatenablog.com/rss -o hatena-rss.xml
: (snip)
$ sudo gem install jekyll-import
: (snip)
1 gem installed
$ ruby -rubygems -e 'require "jekyll-import";
    JekyllImport::Importers::RSS.run({
      "source" => "hatena-rss.xml"
    })'
```

こんな感じでやると_postディレクトリに記事が生成されるのですが、

* 出力されるファイルがmdでなくhtml
* rssのpubDateが出力ファイルのヘッダのdateに反映されない

など注意が必要です。(私の場合はどちらもそのままにしました)

その他、追加で以下の作業を行いました。

* キーワードリンクの削除
* gist, youtube, vimeoのiframeを調整
* 画像をimagesディレクトリ配下に

リポジトリはこちらです。[https://github.com/kn1kn1/kn1kn1.github.io](https://github.com/kn1kn1/kn1kn1.github.io)

更新するとgithubの草が生えるので、今までより少しだけ更新頻度が上がるかなーというところです。
