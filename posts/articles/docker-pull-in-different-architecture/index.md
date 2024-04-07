---
draft: true
title: "同じ Docker コンテナを使っているのに Mac の人だけインストールできないアプリケーションがあった話"
date: "2024-04-07T09:45:55+09:00"
description: Docker コンテナのアーキテクチャを揃えます。
---

## TL;DR

{{< alert type="info" >}}
Docker のイメージのアーキテクチャを揃えるとうまくいくことがあります。
でもパフォーマンスの問題とトレードオフです。
{{< /alert >}}

現職で Windows ユーザと Mac ユーザが両方いる状況で Docker を使用した開発を行う際、
Docker コンテナを使って環境を揃えたはずなのになぜか Mac の人だけ特定のアプリをインストールできないということがありました。

## :umbrella: 事象確認

Volta という Node.js のバージョン管理ツールを Ubuntu のコンテナ内にインストールする場合を見てみます。

まずは私の Windows 環境でインストールした場合です。

```text
root@e1e4ec59f075:/# curl https://get.volta.sh | bash
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 10930  100 10930    0     0  18747      0 --:--:-- --:--:-- --:--:-- 18780
  Installing latest version of Volta (1.1.1)
    Checking for existing Volta installation
    Fetching archive for Linux, version 1.1.1
######################################################################## 100.0%
    Creating directory layout
  Extracting Volta binaries and launchers
    Finished installation. Updating user profile settings.
Updating your Volta directory. This may take a few moments...
success: Setup complete. Open a new terminal to start using Volta!
```

うまくいきました。

次にチームメンバーの Mac 環境でインストールした場合です。

```text
root@7aadcab7e1a3:/# curl https://get.volta.sh | bash
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 10930  100 10930    0     0   3010      0  0:00:03  0:00:03 --:--:--  3026
Error: Sorry! Volta currently only provides pre-built binaries for x86_64 architectures.
```

えー、、せっかく Docker 使ってるのに・・・
わたしの PC がだめなのかな。。。 :pleading_face:

## :bulb: 解決方法

いえ、そんなことはありません。
エラーメッセージをよく読んでみましょう。

```text
Error: Sorry! Volta currently only provides pre-built binaries for x86_64 architectures.
```

x86_64 のアーキテクチャを使わなければならないようです。

アーキテクチャを指定するには、普段の `docker pull` コマンドを少し変えて以下のようにします。

```shell
docker pull --platform linux/arm64 ubuntu:rolling
```

ということで Mac の人にも x86_64 アーキテクチャのイメージをプルしてもらって再度試した結果・・・

```text
root@e1e4ec59f075:/# curl https://get.volta.sh | bash
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 10930  100 10930    0     0  18747      0 --:--:-- --:--:-- --:--:-- 18780
  Installing latest version of Volta (1.1.1)
    Checking for existing Volta installation
    Fetching archive for Linux, version 1.1.1
######################################################################## 100.0%
    Creating directory layout
  Extracting Volta binaries and launchers
    Finished installation. Updating user profile settings.
Updating your Volta directory. This may take a few moments...
success: Setup complete. Open a new terminal to start using Volta!
```

おめでとうございます :tada:

## :thinking: Docker は OS とアーキテクチャに合うイメージを自動で取得する

`arch` コマンドで 2 人のアーキテクチャを確認してみると、

Windows の私は

```text
root@e1e4ec59f075:/# arch
x86_64
```

Mac のメンバーは

```text
root@7aadcab7e1a3:/# arch
aarch64
```

で異なっていました。
（Mac のアーキテクチャがが全部 aarch64 というわけではないです。）

Docker は OS とアーキテクチャに合うイメージを自動で取得するようになっています。
アーキテクチャがホストとコンテナで異なる場合、パフォーマンスが落ちる可能性があるそうです。

[Multi-platform images | Docker Docs](https://docs.docker.com/build/building/multi-platform/)

> When you run an image with multi-platform support, Docker automatically selects the image that matches your OS and
> architecture.

実際に x86_64 の PC で arm64 のイメージをプルすると Docker Desktop 上で以下のように表示されます。

![arm64_on_amd64](arm64_on_amd64.png)

## まとめ

アーキテクチャを揃えれば、異なるアーキテクチャの PC を使っている人も同じ環境で開発できるのは事実ですが、
PC とコンテナで異なるアーキテクチャの場合はパフォーマンスが低下してしまうというデメリットもあり、一長一短かと思います。

アーキテクチャとパフォーマンスの問題を同時に、そして厳密に解決するなら、みんな同じアーキテクチャの PC を使用した方がいいでしょう・・・
