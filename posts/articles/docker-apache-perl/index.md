---
title: 【Docker】Apache と Perl を別コンテナで動かす
date: 2023-01-18T00:00:00+09:00
description: Docker において Apache と Perl のコンテナを分ける方法を紹介しています。
---

業務において Movable Type の開発案件を引き受けることがあるのですが、Docker 環境で開発するにあたり Apache と Perl でコンテナを分けることで、環境の軽量化を図りたいと思いました。

今回は軽量化を実現するために私がいろいろ調べて実践した方法をご紹介します。

{{< alert type="danger" >}}
ここではあくまで開発環境を構築するための方法をご紹介しています。
セキュリティ的にはベストでないものもありますので、商用環境にそのまま利用しないでください。
{{</ alert >}}

{{< alert type="warning" >}}
この記事では Apache と Perl の部分のみ取り扱います。
Movable Type に必要な設定や、データベース・PHP のコンテナなどは省略します。
{{</ alert >}}

## ファイル

### docker-compose.yaml

Perl と Apache は別コンテナで稼働させます。
また、httpd は一番軽量な Alpine Linux ベースのものを使用します。

```yaml
version: '3'

services:
  perl:
    build:
      context: ./docker/perl
    volumes:
      - type: bind
        source: ./example
        target: /usr/local/apache2/vhosts/localhost
  httpd:
    build:
      context: ./docker/httpd
    volumes:
      - type: bind
        source: ./docker/httpd/httpd-vhosts.conf
        target: /usr/local/apache2/conf/extra/httpd-vhosts.conf
      - type: bind
        source: ./example
        target: /usr/local/apache2/vhosts/localhost
    ports:
      - "80:80"
```

### Perl コンテナ

マルチステージビルドを採用します。

```dockerfile
FROM perl:5.36.0-threaded-bullseye AS build

RUN apt update

COPY cpanfile cpanfile
RUN cpanm --notest --installdeps .

FROM perl:5.36.0-slim-threaded-bullseye AS deploy

RUN apt update \
    && apt install -y fcgiwrap spawn-fcgi

COPY --from=build /usr/local/lib/perl5 /usr/local/lib/perl5

ARG FCGI_PORT=9000
RUN sed -ie "s/^FCGI_SOCKET=.*$/FCGI_PORT=\"${FCGI_PORT}\"\nFCGI_ADDR=\"0.0.0.0\"/g" /etc/init.d/fcgiwrap
EXPOSE ${FCGI_PORT}

CMD ["bash", "-c", "service fcgiwrap start && tail -f /dev/null"]
```

ビルドステージでは、必要なモジュールを cpanfile からインストールします。
デプロイステージでは、ビルドツールの入っていない軽量な Perl イメージを元にして、ビルドステージからインストールしたモジュールをコピーし、Perl を CGI 化して動かすためのセットアップをします。

マルチステージビルドに関しては、こちらの [Gist](https://gist.github.com/natanlao/fc8923ca5ef326f36cd580d457a2e411) を参考にしました。

試しにモジュールを一つ入れてみることにします。

```perl
requires 'HTML::Entities';
```

### Apache コンテナ

```apache:docker/httpd/httpd-vhosts.conf
<VirtualHost *:80>
    ServerName localhost
    DocumentRoot "/usr/local/apache2/vhosts/localhost/htdocs"

    Alias /cgi-bin/ "/usr/local/apache2/vhosts/localhost/cgi-bin/"

    <Directory "/usr/local/apache2/vhosts/localhost/htdocs">
        Options None
        AllowOverride None
        Require all granted

        DirectoryIndex index.html
    </Directory>

    <Directory "/usr/local/apache2/vhosts/localhost/cgi-bin">
        AllowOverride None
        Options None
        Require all granted

        <FilesMatch "\.cgi$">
            ProxyFCGIBackendType GENERIC
            SetHandler "proxy:fcgi://perl:9000"
        </FilesMatch>
    </Directory>
</VirtualHost>
```

ファイル名が `.cgi` で終わる場合にプロキシを設定し、Perl コンテナの Perl を利用するようにします。

```dockerfile:docker/httpd/Dockerfile
FROM httpd:2.4.54-alpine3.17

RUN sed -i -Ee 's/#(LoadModule proxy_module modules\/mod_proxy.so)/\1/g' /usr/local/apache2/conf/httpd.conf \
    && sed -i -Ee 's/#(LoadModule proxy_fcgi_module modules\/mod_proxy_fcgi.so)/\1/g' /usr/local/apache2/conf/httpd.conf \
    && sed -i -Ee 's/#(Include conf\/extra\/httpd-vhosts.conf)/\1/g' /usr/local/apache2/conf/httpd.conf
```

プロキシの設定を有効にします。

### サンプルファイル

`example/cgi-bin/index.cgi` に Perl のバージョンを表示するファイルを置きます。

```perl:example/cgi-bin/index.cgi
#!/usr/bin/env perl
use strict;
use warnings FATAL => 'all';

print "Content-type: text/html\n\n";
print "Hello World!<br>";
print "Perl Version: $^V<br>";
```

WSL 等の Linux 上にファイルを置いている場合は実行権限を付与してください。

```sh
chmod a+x example/cgi-bin/index.cgi
```

`example/htdocs/index.html` にもサンプルファイルを置きます。

```html:example/htdocs/index.html
Hello HTML!
```

## 動作確認

ファイルを作成したらコンテナを立ち上げて動作確認をします。

```sh
docker-compose up -d --build
```

- `http://localhost/index.html` にアクセスして `Hello HTML!` と表示されることを確認します。
- `http://localhost/cgi-bin/index.cgi` にアクセスして以下のように表示されることを確認します。

```html
Hello World!
Perl Version: v5.36.0
```

動作確認後はコンテナを終了します。

```sh
docker-compose down --remove-orphans -v
```
