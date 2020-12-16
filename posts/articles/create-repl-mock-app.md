---
title: "Docker in Docker を体験する～ブラウザでコードを書いて実行するアプリを Docker で作ってみた～"
date: 2020-12-24T07:00:00+09:00
description: ブラウザでコードを書いてすぐに実行できる環境を作ってみようの回です。その中で Docker in Docker の構造を使うことになりました。
hero: /images/posts/Moby-logo.png
news_keywords:
  - Docker
  - Docker in Docker
menu:
  sidebar:
    name: Docker in Docker
    identifier: docker-in-docker
    parent: advent-calender-2020
    weight: 20201225
---

{{< alert type="info" >}}
こちらの記事は [Docker Advent Calendar 2020 - Qiita](https://qiita.com/advent-calendar/2020/docker) 25 日目の記事です。
{{< /alert >}}

最近は IDE でコード補完の助けを受けながら開発をしている筆者。

ずっとコード補完に頼りっきりではいつか困ったりしない？と思い始めています。

そこで今回は自分自身のコーディング力および技術力向上を兼ねて、 [Repl.it](https://repl.it/) や [PaizaCloud](https://paiza.cloud/ja/) のような、「ブラウザでコードを書いて実行するアプリ」を 1 から作ってみました。

このアプリを作るにあたり Docker in Docker の技術を使う機会がありましたので、ここで紹介させていただきたく思います。

リポジトリは [こちら](https://gitlab.com/technical-study/coding-drills) にありますので、 Docker 環境のある方はお試しくださいませ。

---

## デモ動画

{{< youtube zQze_N26Nek >}}

{{< vs >}}

---

## Docker in Docker の使いどころ

このアプリ自体の環境は Docker で構築しています。

動いているコンテナは次の 2 つです。

#### PHP + Apache コンテナ

コードを入力する画面の描画と、このあと触れる Docker コンテナへリクエストを送信します。

#### Docker コンテナ

Docker が入っているコンテナです。

この中にコードを実行するための各言語のコンテナをビルドします（この部分が Docker in Docker になります）。

また、このコンテナに Python + FastAPI の組み合わせで API サーバを構築し、 php-apache コンテナからのリクエストを受けます。

![Coding Drills Docker Container](/images/posts/coding-drills-docker-container.png)<!-- @IGNORE PREVIOUS: link -->

---

## この構成にした理由

私自身、インフラ面の経験があまりあるわけではありませんが、自分なりの理由を記しておきます。

#### コンテナ間通信を整備するためのコスト

別に Docker in Docker の構成にしなくても PHP + Apache コンテナと各言語のコンテナを並列で並べる案もあるのですが、
その場合、コンテナに SSH で通信しながらコマンドを実行したり、あるいは HTTP の通信にしてもコンテナにサーバを構築する必要があります。

これによって、構成が複雑になる可能性があるため、今回は避けました。

#### `rm -rf` 系の実行に備える

次に考えられる案が「Docker のコンテナの中に各言語をインストールする案」ですが、多くのプログラミング言語からはシェルコマンドが実行可能なため、 `rm -rf ~` をはじめとする malformed なコマンドも実行される可能性があります。

そのため、コードを実行する環境は実行の度にビルドして、実行後は壊すという方法を取ればいいのではないかと考えました。

今回はローカルで動かしているだけなので関係ないですけどｗ

---

## その他、大変だったところ

API サーバを立てて Python で Docker コマンドを実行する場面があるのですが、これを完成させるのに非常に苦戦しました。

シェルで Docker コマンドを実行するとうまくいくのになぜ Python の subprocess 経由で Docker コマンドを実行するとうまくいかないのか、と。

ずっと `dial tcp: lookup docker on 127.0.0.11:53: no such host` の類のエラーで悩んでいました。

そこで StackOverflow などを回っていたところ、 Unix ドメインソケットの話が出てきました。

[こちらの記事](https://www.ogis-ri.co.jp/otc/hiroba/technical/docker/part6.html) によると、こんなことが書いてありました。

> Docker CLI が Dockerホストの中にある場合は、Unixドメインソケット（以下、Unixソケット）を用いて Dockerデーモンと通信します。

> Docker CLI が Dockerホストの外にある場合は、TCPソケットを用いて Dockerデーモンと通信します。

シェルからの Docker コマンドが暗黙のうちに Unix ドメインソケットで通信していて、
Python の subprocess 経由で Docker コマンドは暗黙のうちに TCP ソケットで通信していたのではないか、と。

そこから、 Docker コマンドではなく直接 curl で Docker Engine API を叩く方針に変えたところ、問題が解決しました。

結局、コンテナのビルド部分だけでこんな感じのコードを書くことに。変数名が美しくない点はご了承くださいませ。

```python
import urllib.parse
import requests_unixsocket
import json

unix_socket = urllib.parse.quote('/var/run/docker.sock', safe='')
containers_path = '/v1.40/containers'

with requests_unixsocket.Session() as session:

    # Create a container
    container_name = 'python'
    operation = f'/create?name={container_name}'
    url = f'http+unix://{unix_socket}{containers_path}{operation}'

    data = json.dumps({
        'Image': 'python:3.9-slim-buster',
        'Cmd': ['python', '/code.py'],
        'Mounts': [
            {
                'Target': '/code.py',
                'Source': '/docker-python-fastapi/code.py',
                'Type': 'bind',
                'ReadOnly': True,
            },
        ],
    }).encode()
    header = {'Content-Type': 'application/json'}

    session.post(url, data=data, headers=header)
```

---

## 感想

Docker 自体には触れていたものの、 Docker in Docker にはあまりなじみがなかったため、今回とてもいい経験ができました。

もともと Docker in Docker を使うことが目的ではなく、アプリを作るうえでこの概念が出てきたので、アプリを作っている間は非常にモチベーションを保てていました。

そういった意味ではよかったかなと思います。

Docker in Docker というと Jenkins で CI を回す際によく使われるそうです。 CI で自動テストを回すところには非常に興味を持っているので、また機会があったらやってみたいと思います。

---

最後までお読みいただきありがとうございました :wink:
