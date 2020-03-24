---
title: "git submoduleについて"
date: 2020-03-22T20:53:28+09:00
draft: false
tags: ["Hugo", "Git"]
---

ブログをHugoで生成するとき、テーマをリポジトリのsubmoduleとして追加する必要がある。
追加は以下のコマンドでできる。例として使ったのは[Hugo Flex](https://themes.gohugo.io/hugo-flex/)。
<!--more-->
```bash
git submodule add https://github.com/de-souza/hugo-flex.git themes/hugo-flex
```
すると、`themes`ディレクトリの下に`hugo-flex`のリポジトリがsubmoduleとして追加され、同時にリポジトリ直下に`.gitmodule`というファイルが生成される。ファイルの内容は以下の通り。
```dark
[submodule "themes/hugo-flex"]
	path = themes/hugo-flex
	url = https://github.com/de-souza/hugo-flex.git
```
この状態でcommitして、GitHubにpushした。
ここまではOK。
上記のGitHubリポジトリから`git clone`して`hugo server`でサーバ起動してみる。
```bash
$ git clone git@github.com:tazeen727/tech_memo.git tech_memo2
Cloning into 'tech_memo2'...
remote: Enumerating objects: 24, done.
remote: Counting objects: 100% (24/24), done.
remote: Compressing objects: 100% (14/14), done.
remote: Total 24 (delta 6), reused 24 (delta 6), pack-reused 0
Receiving objects: 100% (24/24), 4.55 KiB | 1.51 MiB/s, done.
Resolving deltas: 100% (6/6), done.
$ cd tech_memo2
$ hugo server
Building sites … WARN 2020/03/22 21:27:21 found no layout file for "HTML" for kind "page": You should create a template file w
hich matches Hugo Layouts Lookup Rules for this combination.
WARN 2020/03/22 21:27:21 found no layout file for "HTML" for kind "taxonomyTerm": You should create a template file which matc
hes Hugo Layouts Lookup Rules for this combination.
（略）
```
`found no layout file`だそうだ。`localhost:1313`にアクセスしても何も表示されない。
そこで`themes/hugo-flex`ディレクトリの内容を表示する。
```bash
$ ls themes/hugo-flex/
$
```
ディレクトリが空だ。ただ、`git submodule`を見ると確かにsubmoduleとしては登録済み。
```bash
tazeen@DESKTOP-1O1KFQH:~/hugo_blog/tech_memo2$ git submodule
-a2bef69c8c27b8ff1d5daf5e0d9d037ff07dc648 themes/hugo-flex
```
しかし、ここで２行目の先頭についている「`-`」に注目。これはsubmoduleとしては登録しているけど、ファイルを持ってきていないことを表すらしい。
submoduleのファイルを取り込むには、`git submodule update -i`コマンドを用いる。
```bash
$ git submodule update -i
Submodule 'themes/hugo-flex' (https://github.com/de-souza/hugo-flex.git) registered for path 'themes/hugo-flex'
Cloning into '/home/tazeen/hugo_blog/tech_memo2/themes/hugo-flex'...
Submodule path 'themes/hugo-flex': checked out 'a2bef69c8c27b8ff1d5daf5e0d9d037ff07dc648'
$ git submodule
 a2bef69c8c27b8ff1d5daf5e0d9d037ff07dc648 themes/hugo-flex (heads/master)
```
これで`git submodule`したときの先頭の「`-`」が消え、末尾に`(heads/master)`が付いた。これでファイルも含めて最新の状態になったことを表している。
`hugo server`でサーバ起動してからアクセスすると、想定どおりテーマが反映された状態でブログを表示できた。

ちなみに、`git clone`するときにsubmoduleも一緒に取り込むオプションもある。以下の通り（Git 2.13以降）。
```bash
$ git clone --recurse-submodules git@github.com:tazeen727/tech_memo.git tech_memo3
```

`git submodule`についてはまだまだ理解不足。そんなに使う頻度の多い機能でもないと思うので（少なくとも個人では…）、忘れないように残しておけば今度使うときに役に立ちそう。

## 参考
- [Git submodule の基礎 \- Qiita](https://qiita.com/sotarok/items/0d525e568a6088f6f6bb)
- [Git submoduleの押さえておきたい理解ポイントのまとめ \- Qiita](https://qiita.com/kinpira/items/3309eb2e5a9a422199e9)
- [How to "git clone" including submodules? \- Stack Overflow](https://stackoverflow.com/questions/3796927/how-to-git-clone-including-submodules)
