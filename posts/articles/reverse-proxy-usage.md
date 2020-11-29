---
title: "とある開発環境の反向代理（リバースプロキシ）"
date: 2020-11-29T00:03:21+09:00
description: 某広告効果測定の会社で開発環境の Docker 化を進めていた時にリバースプロキシの案が出てきたので、それについて振り返ります。
news_keywords:
  - 開発環境
  - Docker
  - Reverse Proxy
  - リバースプロキシ
---

## :mailbox_with_mail: 背景

むかしむかしあるところに、某広告効果測定ツールの開発チームがいました。

そこには、 Web サイトを訪問した際に裏で別ドメインの JavaScript を読み込んで実行し、そのスクリプトがさらに別ドメインの PHP にデータを送信してアクセスを計測する、という仕組みがありました。

チーム内で管理されているソースの中には JavaScript 中心のアプリ A と PHP 中心のアプリ B が別々に存在して、各々が開発環境の Docker 化を進め、各々がホスト PC の `localhost` の 80, 443 番ポートをコンテナと直接繋いでいました。

アクセスを計測する Web サイトもダミーサイト C を作り、同じようにポートを接続していました。

ところがある日、結合テストのフェーズになり、アプリ A と B 、ダミーサイト C で計 3 つのドメインを振り分けなければいけなくなりました。

すると、アプリたちがこぞって `localhost` のポートを取り合い、喧嘩になったのです。

スクリプトの読み込みにポート番号を付けてアクセスしたりはしないので、ポート番号を変えることもできません。

喧嘩の末、ダミーサイトが `localhost` を取り、他のアプリたちは仕方なく VirtualBox で VM を立ち上げ、各自の IP を `hosts` ファイルに書いてもらうという、古い方法を使うことになりました。

![Port toriaikko](/images/posts/port-toriaikko.png)

仲良く Docker を使うためにはどうすればよかったのでしょうか。

---

## :thinking_face: 方法を考える

Docker Desktop で開発するということを前提に次の 2 通りの方法を考えました。

#### :one: すべて同じコンテナに入れてしまう

1 つのコンテナの中で各アプリをディレクトリを分けて配置し、バーチャルホスト等の設定で ServerName に応じた割り振りをします。

`hosts` ファイルは次のように記述します。

```html
127.0.0.1 app-a
127.0.0.1 app-b
127.0.0.1 site-c
```

- メリット
    - 構成はシンプルに見える
    - バーチャルホストさえ書ければ楽
- デメリット
    - コンテナが重くなる

#### :two: アプリごとにコンテナを分けてリバースプロキシによりアクセスを割り振る

今回試そうと思っている方法はこちらです。アプリとダミーサイト個々のコンテナを用意しますが、それらに直接ポートの設定をするのではなく、間にリバースプロキシを挟みます。

ブラウザからのアクセスは最初はすべてリバースプロキシを通り、そこから ServerName によってアプリやダミーサイトにアクセスがいきます。

下の図でイメージしていただければ幸いです。

{{< alert type="info" >}}
私自身、リバースプロキシの理解には [こちらの記事](https://qiita.com/zawawahoge/items/a931de1464ccaa228551) を参考にさせていただきました。
{{< /alert >}}

![Reverse proxy figure](/images/posts/reverse-proxy-figure.png)

`hosts` ファイルの中身は :one: の方法と同じです。

- メリット
    - コンテナを小分けにできる
- デメリット
    - 構成は少し複雑
    - 本番と構成が違う場合があり、好まない人が多い

---

## :fire::meat_on_bone::fork_and_knife: リバースプロキシを作ってみる

[こちら](https://gitlab.com/technical-study/docker_reverse_proxy) にソースコードがありますので、ぜひお試しください。

ファイルツリーは以下のようになっています :deciduous_tree:

```html
│  docker-compose.yml
│
├─app-a
│      index.html
│      index.js
│
├─app-b
│      index.php
│
├─site-c
│       index.html
│
└─nginx
   └─conf.d
           reverse-proxy.nginx.conf
```

---

ファイルの概要を紹介しておきます。

#### :page_facing_up: `./docker-compose.yml`

アプリとサイトそれぞれにコンテナを用意しますが、ポートはリバースプロキシにしか開けません。

```yaml
version: '3'

services:
  app-a:
    container_name: app-a
    image: httpd:alpine
    volumes:
      - ./app-a:/usr/local/apache2/htdocs
  app-b:
    container_name: app-b
    image: php:apache-buster
    volumes:
      - ./app-b:/var/www/html
  site-c:
    container_name: site-c
    image: httpd:alpine
    volumes:
      - ./site-c:/usr/local/apache2/htdocs
  reverse-proxy:
    container_name: reverse-proxy
    image: nginx:alpine
    volumes:
      - ./nginx/conf.d/reverse-proxy.nginx.conf:/etc/nginx/conf.d/reverse-proxy.nginx.conf
    ports:
      - "80:80"
```

#### :page_facing_up: `./nginx/conf.d/reverse-proxy.nginx.conf`

コンテナ間の通信がコンテナ名で行えることを利用します。

```nginx
server {
    listen 80;
    server_name app-a;

    location / {
        proxy_pass http://app-a/;
        proxy_redirect off;
    }
}

server {
    listen 80;
    server_name app-b;

    location / {
        proxy_pass http://app-b/;
        proxy_redirect off;
    }
}

server {
    listen 80;
    server_name site-c;

    location / {
        proxy_pass http://site-c/;
        proxy_redirect off;
    }
}
```

#### :page_facing_up: `./site-c/index.html`

この中で `app-a/index.js` が読み込まれ、さらにその Javascript が `app-b/index.php` を読み込むようになっています。

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Hello World</title>
</head>
<body>
<h1>Hello World</h1>
<script type="text/javascript" src="//app-a/index.js"></script>
<div id="resp">Your UA is...</div>
</body>
</html>
```

#### :page_facing_up: `./app-a/index.js`

文字を出力した後、 `http://app-b/` にリクエストを送ります。

```js
document.write("Hello Javascript");

const request = new XMLHttpRequest();
request.open('GET', 'http://app-b/', true);
request.responseType = 'json';
request.addEventListener('load', () => {
    document.getElementById('resp').innerHTML = request.response['HTTP_USER_AGENT'];
});
request.send();
```

#### :page_facing_up: `./app-b/index.php`

今回は ~~無駄に~~ 2 秒待ってから UA を返すスクリプトを書いてみました。

```php
<?php
header('Content-type: text/json; charset=utf-8');
$allowOrigin = 'http://site-c';
header("Access-Control-Allow-Origin: ${allowOrigin}");
$resp = array('HTTP_USER_AGENT' => $_SERVER['HTTP_USER_AGENT']);
sleep(2);
echo json_encode($resp);
exit;
```

#### :page_facing_up: `./app-a/index.html`

空ファイルです。

---

## :computer: 実行する

コンテナをビルドして、

```sh
docker-compose up -d
```

ブラウザで `http://site-c/` にアクセスして 2 秒くらい待つと...

![Reverse proxy screenshot](/images/posts/reverse-proxy-screenshot.png)

表示できました！ :tada:

{{< alert type="info" >}}
リバースプロキシの実装にあたっては [こちらの記事](https://qiita.com/zawawahoge/items/d58ab6b746625e8d4457) を参考にさせていただきました。
{{< /alert >}}

---

## :bulb: まとめ

リバースプロキシを使うことでコンテナ 1 つ 1 つがとても軽くできたので、そこは良かったと思います。

みなさんも Docker でポートの取り合いになったときは、このリバースプロキシを検討してみてください。

（リバースプロキシには、各サーバへの負担を分散したり、リバースプロキシ自体がファイアウォールとして機能するといった利点もあります。）

{{< alert type="danger" >}}
この記事は、決してリバースプロキシが最善の方法 :angel: であるとか、それ以外の方法はダメ :imp: だということを示すものではありません。状況に応じて適切に方法を選ぶようにしてください。
{{< /alert >}}
