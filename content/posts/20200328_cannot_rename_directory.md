---
title: "ディレクトリのリネームでpermission deniedと表示される"
date: 2020-03-28T09:53:55+09:00
draft: true
tags: ["WSL1", "Visual Studio Code"]
---

VS CodeでScalaプロジェクトのパッケージのリネームがしたくなったので、VS Code上でリネームしたら、右下にこんなエラーメッセージが出た。
```
Error: EACCES: permission denied, rename '/home/tazeen/toy/renamedir/src/main/scala/example' -> '/home/tazeen/toy/renamedir/src/main/scala/sample'
```
<!--more-->
permission deniedってことは、ディレクトリに権限がないってこと？権限を見てみたが問題なさそう。
```bash
$ pwd
/home/tazeen/toy/renamedir/src/main/scala
$ ls -l
total 0
drwxrwxrwx 1 tazeen tazeen 4096 Mar 28 09:50 example
```

試しにbashからリネームできるか試すが、ダメ。

```bash
$ mv example sample
mv: cannot move 'example' to 'sample': Permission denied
```

色々いじっていると、どうやら中にファイルが含まれているとリネームできないことが分かった。

```bash
$ mv example sample
mv: cannot move 'example' to 'sample': Permission denied
$ ls -l example
total 0
-rw-rw-rw- 1 tazeen tazeen 138 Mar 28 09:50 Hello.scala
$ mv example sample
mv: cannot move 'example' to 'sample': Permission denied
$ mv example/Hello.scala ./
$ ls -l example
total 0
$ mv example sample # 中にファイルがなければ変更可
$ ls -l
total 0
-rw-rw-rw- 1 tazeen tazeen  138 Mar 28 09:50 Hello.scala
drwxrwxrwx 1 tazeen tazeen 4096 Mar 28 10:14 sample
```

ここまで調べて、Googleでいろいろ検索してたら見つけた。どうやらWSL1とVisual Studio Codeで、ファイル監視に起因する問題らしい。

- GitHub issues
    - [EACCES when renaming folder that is being watched from nodejs · Issue \#3395 · microsoft/WSL](https://github.com/microsoft/WSL/issues/3395)
    - [VSCode Remote WSL: unable to rename folder · Issue \#208 · microsoft/vscode\-remote\-release](https://github.com/microsoft/vscode-remote-release/issues/208)
- Visual Studio Code docs
    - [Developing in the Windows Subsystem for Linux with Visual Studio Code](https://code.visualstudio.com/docs/remote/wsl#_i-see-eaccess-permission-denied-error-trying-to-rename-a-folder-in-the-open-workspace)

試しにVS Codeを終了してからリネームしたら問題なくリネームできた。
WSL2では問題は発生しないそうだ。WSL1での解決策も書いてあるので、設定しておく。
パフォーマンスに影響があるらしいので、様子を見て`pollingInterval`は調整しよう。
{{< figure src="/img/20200328.png" class="center" >}}

VS Codeを再起動したら、ディレクトリのリネーム・移動ができるようになった。よし。
