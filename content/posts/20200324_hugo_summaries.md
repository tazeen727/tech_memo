---
title: "Hugoで作ったブログのトップに要約で一覧表示する"
date: 2020-03-24T12:10:42+09:00
draft: true
tags: ["Hugo",]
---

Hugoで作ったブログを開いてみると、トップページに記事全体が表示され、とんでもなく長くなる。
好みもあるが、トップページでは最近投稿した記事が一覧で見れたほうが使いやすいと思う。
<!--more-->
ブログの中にはトップページでは記事のタイトルと記事の最初の数行だけ表示して、最後に「Read More」など、
記事単独のページヘのリンクが付いているようなものがあるので、それをHugoにもやってほしい。
Hugoのテーマの問題かなと思い他のテーマ（Mainroad）とかにも変えてみたんだけど、やっぱり要約表示されない。

しかし、[MainroadのDemoページ](https://themes.gohugo.io/theme/mainroad/)を見ると
トップページはちゃんと記事の要約になっていることに気づいた。これを真似すればいいんだ！
各テーマのDemoページは、GitHubリポジトリ直下のディレクトリ`exampleSite`に配置されている。
そのうちの記事の１つ（`basic-elements.md`）をRaw表示する。

https://raw.githubusercontent.com/Vimux/Mainroad/master/exampleSite/content/post/basic-elements.md

```markdown
---
title: Basic HTML Elements
description: Example test article that contains basic HTML elements for text formatting on the Web.
date: 2018-04-16
categories:
  - "Development"
tags:
  - "HTML"
  - "CSS"
  - "Basic Elements"
---

The main purpose of this article is to make sure that all basic HTML Elements are decorated with CSS so as to not miss any possible elements when creating new themes for Hugo.
<!--more-->

## Headings
（省略）
```

なるほど、「`<!--more-->`」と書けばいいのね。そのうち`default.md`に追記しておこう。
そう思って過去記事に「`<!--more-->`」を追記して記事を表示してみるも、反映されない。
Hugo-Flexのページを見ると、Configurationセクションに以下のパラメータを発見。

```yaml
params:
  summaries: false      # Set to true to show summaries of posts on homepage
```

どうやらデフォルトがfalseなので、要約表示されなかったみたい。じゃあこれをtrueに変えよう。
自分はConfigurationをTOMLで書いてるので、`config.toml`に以下を追記。

```toml
[Params]
summaries = true
```

記事を再表示すると、ちゃんと各記事の最初の数行が表示され、最後に「Read More...」と表示された。
ただなんかHugo-FlexのRead Moreはリンクの色とかフォントが野暮ったいな…。
デフォルトがオフだしあまり使うことを想定してないのかも？
まあテーマはそのうち変えるつもりだし、細かいことはいいか。
