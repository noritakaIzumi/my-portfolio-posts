---
title: "PHP: SSL サイトで file_get_contents できないときは openssl の疎通確認をするとよい"
date: 2020-12-26T07:00:00+09:00
description: Docker のコンテナ内から https の URL に対して、 curl できたのに PHP で file_get_contents できないパターンがあったので検証します。
news_keywords:
  - https
  - PHP
  - file_get_contents
---

{{< alert type="info" >}}
この記事は [PHP Advent Calendar 2020 - Qiita](https://qiita.com/advent-calendar/2020/php) 6 日目の記事です。
{{< /alert >}}

## :bulb: まとめ

{{< alert type="info" >}}
SSL 化されたコンテンツに対する `file_get_contents` の可否と、 443 番ポートに関する `openssl` の疎通の可否が一致した。
このことから、 `file_get_contents` と `openssl` には少なからず関連があり、 `file_get_contents` がうまくいかない場合に `openssl` での疎通確認を行うことは効果的である。
{{< /alert >}}

---

Docker のコンテナ内から https の URL に対して、 `curl` できたのに PHP で `file_get_contents` できないパターンがありました。

「？」と思ったので検証してみます。

Windows 10 Pro を使用しています。

---

## :coffee: 準備

まずはプロジェクトフォルダを好きな場所に作成します。

```bash
mkdir -p /path/to/dir
cd /path/to/dir
```

#### サーバーで使う証明書を用意する

今回は `mkcert` で証明書を作成します。
`mkcert` をインストールします。

私は Windows 使いなので chocolatey でインストールします。
PowerShell を管理者権限で開いて実行します。

```bash
cinst -y mkcert
```

初回のみ、次のコマンドを実行します。

```bash
mkcert -install
```

セキュリティに関する警告が出てきますが、「はい」をクリックします。

```html
$ mkcert -install
Created a new local CA 💥
The local CA is now installed in the system trust store! ⚡️
```

絵文字が出てくるのがかわいいですね :wink:

次に `localhost` に対する証明書を発行します。

```bash
mkdir ssl
cd ssl
mkcert localhost php-apache.com
```

```html
$ mkcert localhost php-apache.com

Created a new certificate valid for the following names 📜
 - "localhost"
 - "php-apache.com"

The certificate is at "./localhost+1.pem" and the key at "./localhost+1-key.pem" ✅

It will expire on 28 February 2023 🗓
```

ルート証明書も後で使うので、コピーしておきます。
ルート証明書の場所は次のコマンドで確認できます。

```bash
mkcert -CAROOT
```

確認したら、そこから `rootCA.pem` をコピーします。

#### 使用するファイルを用意する

今回用意するファイルは以下の通りです。

```html
/path/to/dir
│  docker-compose.yml
│  Dockerfile
│
├─html
│      hello.php
│      index.php
│
└─ssl（用意済み）
        localhost+1-key.pem
        localhost+1.pem
        rootCA.pem
```

###### `./docker-compose.yml`

今回はコンテナ内でコマンドを打ちますので、ポートを開ける必要はありません。

```yaml
```yaml
```yaml
```yaml
version: '3'

services:
  php-apache:
    container_name: php-apache.com
    build: ""
    volumes:
      - ./html:/var/www/html
```

###### `./Dockerfile`

この後の検証で何度か編集しますので、今は空のファイルを用意しておきます。

```dockerfile
```

###### `./html/index.php`

```php
<?php
$url = 'https://php-apache.com/hello.php';
echo file_get_contents($url);
```

###### `./html/hello.php`

```php
<?php
header('Content-type: text/plain; charset=UTF-8');
echo 'hello';
exit;
```

これで準備は終わりです :+1:

---

## 概要

ここからは実際の検証に入る前に、検証の概要について説明します。

今回はコンテナの状態を 4 つ作り、それぞれに対して 3 つの方法を試します。

#### コンテナの状態

- <mark>最初の状態</mark>: mkcert で作った証明書を配置して ssl を有効にしただけの状態
- 状態 2: <mark>最初の状態</mark>で、新たに CA 証明書を配置し、環境変数 `CURL_CA_BUNDLE` に設定した状態
- 状態 3: <mark>最初の状態</mark>で、 CA 証明書を配置し、 OpenSSL のディレクトリに CA 証明書へのシンボリックリンクを設定した状態
- 状態 4: <mark>最初の状態</mark>で、 CA 証明書を配置し、環境変数 `CURL_CA_BUNDLE` に設定し、 OpenSSL のディレクトリに CA 証明書へのシンボリックリンクを設定した状態

#### 方法

- PHP の file_get_contents 関数で `https://php-apache.com/` を読み込む

```bash
winpty docker exec -it php-apache.com php -r "echo file_get_contents('https://php-apache.com/'), PHP_EOL;"
```

> 成功とする基準: PHP の Error や Warning が出ない

- コンテナの中から curl コマンドで `https://php-apache.com/` にリクエストする

```bash
winpty docker exec -it php-apache.com curl https://php-apache.com/
```

> 成功とする基準: **curl に関する** エラーメッセージが出ない

- コンテナの中から openssl コマンドで `php-apache.com:443` への疎通確認を行う

```bash
winpty docker exec -it php-apache.com openssl s_client -quiet -connect php-apache.com:443
```

> 成功とする基準: `verify error` が出ない

{{< alert type="info" >}}
以上の方法を行う前には、 `docker-compose up -d` でコンテナを立ち上げ、行った後には `docker-compose down -v` でコンテナを終了します。
{{< /alert >}}

## 検証

#### <mark>最初の状態</mark>: mkcert で作った証明書を配置して ssl を有効にしただけの状態

`Dockerfile` を以下のように編集します。

```dockerfile
FROM php:apache-buster

RUN cp /usr/local/etc/php/php.ini-development /usr/local/etc/php/php.ini

# mkcert で作った証明書を配置する
COPY ./ssl/"localhost+1.pem" /etc/ssl/certs/ssl-cert-snakeoil.pem
COPY ./ssl/"localhost+1-key.pem" /etc/ssl/private/ssl-cert-snakeoil.key

# Enable SSL
RUN a2enmod ssl && a2ensite default-ssl
```

###### 結果

file_get_contents, curl, openssl ともに失敗しました。

|No.|方法|結果|
|---|---|:---:|
|1|file_get_contents|<span style="color: blue;">失敗</span>|
|2|curl|<span style="color: blue;">失敗</span>|
|3|openssl|<span style="color: blue;">失敗</span>|

###### 詳細

- file_get_contents

SSL に関するエラーが出ていることに注目します。

```html
$ winpty docker exec -it php-apache.com php -r "echo file_get_contents('https://php-apache.com/'), PHP_EOL;"
PHP Warning:  file_get_contents(): SSL operation failed with code 1. OpenSSL Error messages:
error:1416F086:SSL routines:tls_process_server_certificate:certificate verify failed in Command line code on line 1

Warning: file_get_contents(): SSL operation failed with code 1. OpenSSL Error messages:
error:1416F086:SSL routines:tls_process_server_certificate:certificate verify failed in Command line code on line 1
PHP Warning:  file_get_contents(): Failed to enable crypto in Command line code on line 1

Warning: file_get_contents(): Failed to enable crypto in Command line code on line 1
PHP Warning:  file_get_contents(https://php-apache.com/): failed to open stream: operation failed in Command line code on line 1

Warning: file_get_contents(https://php-apache.com/): failed to open stream: operation failed in Command line code on line 1
```

- curl

```html
$ winpty docker exec -it php-apache.com curl https://php-apache.com/
curl: (60) SSL certificate problem: unable to get local issuer certificate
More details here: https://curl.haxx.se/docs/sslcerts.html

curl failed to verify the legitimacy of the server and therefore could not
establish a secure connection to it. To learn more about this situation and
how to fix it, please visit the web page mentioned above.
```

- openssl

```html
$ winpty docker exec -it php-apache.com openssl s_client -quiet -connect php-apache.com:443
depth=0 O = mkcert development certificate, OU = MYCOMPUTER\\norit@MyComputer (Noritaka IZUMI)
verify error:num=20:unable to get local issuer certificate
verify return:1
depth=0 O = mkcert development certificate, OU = MYCOMPUTER\\norit@MyComputer (Noritaka IZUMI)
verify error:num=21:unable to verify the first certificate
verify return:1
...
```

#### 状態 2: <mark>最初の状態</mark>で、新たに CA 証明書を配置し、環境変数 `CURL_CA_BUNDLE` に設定した状態

参考: [[curl] HTTPS通信できない (unable to get local issuer certificate) - noknow](https://noknow.info/it/shell_script/tips/curl_ssl_certificate_probrem_60?lang=ja)

`Dockerfile` は以下の通りです。

```dockerfile
FROM php:apache-buster

RUN cp /usr/local/etc/php/php.ini-development /usr/local/etc/php/php.ini

# mkcert で作った証明書を配置する
COPY ./ssl/"localhost+1.pem" /etc/ssl/certs/ssl-cert-snakeoil.pem
COPY ./ssl/"localhost+1-key.pem" /etc/ssl/private/ssl-cert-snakeoil.key

# CA 証明書を配置する
COPY ./ssl/rootCA.pem /etc/ssl/certs/rootCA.pem
ENV CURL_CA_BUNDLE=/etc/ssl/certs/rootCA.pem

# Enable SSL
RUN a2enmod ssl && a2ensite default-ssl
```

###### 結果

curl が成功したのに対し、
file_get_contents と openssl は失敗しました。

いわゆる **`curl` できたのに `file_get_contents` できないパターン** ですね。

|No.|方法|結果|
|---|---|:---:|
|1|file_get_contents|<span style="color: blue;">失敗</span>|
|2|curl|**<span style="color: red;">成功</span>**|
|3|openssl|<span style="color: blue;">失敗</span>|

###### 詳細

- file_get_contents

```html
$ winpty docker exec -it php-apache.com php -r "echo file_get_contents('https://php-apache.com/'), PHP_EOL;"
PHP Warning:  file_get_contents(): SSL operation failed with code 1. OpenSSL Error messages:
error:1416F086:SSL routines:tls_process_server_certificate:certificate verify failed in Command line code on line 1

Warning: file_get_contents(): SSL operation failed with code 1. OpenSSL Error messages:
error:1416F086:SSL routines:tls_process_server_certificate:certificate verify failed in Command line code on line 1
PHP Warning:  file_get_contents(): Failed to enable crypto in Command line code on line 1

Warning: file_get_contents(): Failed to enable crypto in Command line code on line 1
PHP Warning:  file_get_contents(https://php-apache.com/): failed to open stream: operation failed in Command line code on line 1

Warning: file_get_contents(https://php-apache.com/): failed to open stream: operation failed in Command line code on line 1
```

- curl

PHP でエラーが起こっていますが、 **curl に関してエラーが出ていない** ので成功とします。

```html
$ winpty docker exec -it php-apache.com curl https://php-apache.com/
<br />
<b>Warning</b>:  file_get_contents(): SSL operation failed with code 1. OpenSSL Error messages:
error:1416F086:SSL routines:tls_process_server_certificate:certificate verify failed in <b>/var/www/html/index.php</b> on line <b>3</b><br />
<br />
<b>Warning</b>:  file_get_contents(): Failed to enable crypto in <b>/var/www/html/index.php</b> on line <b>3</b><br />
<br />
<b>Warning</b>:  file_get_contents(https://php-apache.com/hello.php): failed to open stream: operation failed in <b>/var/www/html/index.php</b> on line <b>3</b><br />
```

- openssl

```html
$ winpty docker exec -it php-apache.com openssl s_client -quiet -connect php-apache.com:443
depth=0 O = mkcert development certificate, OU = MYCOMPUTER\\norit@MyComputer (Noritaka IZUMI)
verify error:num=20:unable to get local issuer certificate
verify return:1
depth=0 O = mkcert development certificate, OU = MYCOMPUTER\\norit@MyComputer (Noritaka IZUMI)
verify error:num=21:unable to verify the first certificate
verify return:1
...
```

#### 状態 3: <mark>最初の状態</mark>で、 CA 証明書を配置し、 OpenSSL のディレクトリに CA 証明書へのシンボリックリンクを設定した状態

参考: [[OpenSSL] [エラー] Verification error: unable to get local issuer certificate - noknow
](https://noknow.info/it/openssl/err/unable_to_get_local_issuer_certificate)

`Dockerfile` は以下の通りです。

```dockerfile
FROM php:apache-buster

RUN cp /usr/local/etc/php/php.ini-development /usr/local/etc/php/php.ini

# mkcert で作った証明書を配置する
COPY ./ssl/"localhost+1.pem" /etc/ssl/certs/ssl-cert-snakeoil.pem
COPY ./ssl/"localhost+1-key.pem" /etc/ssl/private/ssl-cert-snakeoil.key

# OpenSSL のディレクトリに CA 証明書へのシンボリックリンクを設定する
COPY ./ssl/rootCA.pem /etc/ssl/certs/rootCA.pem
RUN ln -s /etc/ssl/certs/rootCA.pem $(openssl version -d | cut -d' ' -f2 | sed 's/"//g')/cert.pem

# Enable SSL
RUN a2enmod ssl && a2ensite default-ssl
```

###### 結果

file_get_contents と openssl が成功したのに対し、
curl は失敗しました。

|No.|方法|結果|
|---|---|:---:|
|1|file_get_contents|**<span style="color: red;">成功</span>**|
|2|curl|<span style="color: blue;">失敗</span>|
|3|openssl|**<span style="color: red;">成功</span>**|

###### 詳細

- file_get_contents

```html
$ winpty docker exec -it php-apache.com php -r "echo file_get_contents('https://php-apache.com/'), PHP_EOL;"
hello
```

- curl

```html
$ winpty docker exec -it php-apache.com curl https://php-apache.com/
curl: (60) SSL certificate problem: unable to get local issuer certificate
More details here: https://curl.haxx.se/docs/sslcerts.html

curl failed to verify the legitimacy of the server and therefore could not
establish a secure connection to it. To learn more about this situation and
how to fix it, please visit the web page mentioned above.
```

- openssl

```html
$ winpty docker exec -it php-apache.com openssl s_client -quiet -connect php-apache.com:443
depth=1 O = mkcert development CA, OU = MYCOMPUTER\\norit@MyComputer (Noritaka IZUMI), CN = mkcert MYCOMPUTER\\norit@MyComputer (Noritaka IZUMI)
verify return:1
depth=0 O = mkcert development certificate, OU = MYCOMPUTER\\norit@MyComputer (Noritaka IZUMI)
verify return:1
...
```

#### 状態 4: <mark>最初の状態</mark>で、 CA 証明書を配置し、環境変数 `CURL_CA_BUNDLE` に設定し、 OpenSSL のディレクトリに CA 証明書へのシンボリックリンクを設定した状態

`Dockerfile` は以下の通りです。

```dockerfile
FROM php:apache-buster

RUN cp /usr/local/etc/php/php.ini-development /usr/local/etc/php/php.ini

# mkcert で作った証明書を配置する
COPY ./ssl/"localhost+1.pem" /etc/ssl/certs/ssl-cert-snakeoil.pem
COPY ./ssl/"localhost+1-key.pem" /etc/ssl/private/ssl-cert-snakeoil.key

# CA 証明書を配置する
COPY ./ssl/rootCA.pem /etc/ssl/certs/rootCA.pem
ENV CURL_CA_BUNDLE=/etc/ssl/certs/rootCA.pem

# OpenSSL のディレクトリに CA 証明書へのシンボリックリンクを設定する
RUN ln -s /etc/ssl/certs/rootCA.pem $(openssl version -d | cut -d' ' -f2 | sed 's/"//g')/cert.pem

# Enable SSL
RUN a2enmod ssl && a2ensite default-ssl
```

###### 結果

file_get_contents, curl, openssl ともに成功しました。

|No.|方法|結果|
|---|---|:---:|
|1|file_get_contents|**<span style="color: red;">成功</span>**|
|2|curl|**<span style="color: red;">成功</span>**|
|3|openssl|**<span style="color: red;">成功</span>**|

###### 詳細

- file_get_contents

```html
$ winpty docker exec -it php-apache.com php -r "echo file_get_contents('https://php-apache.com/'), PHP_EOL;"
hello
```

- curl

```html
$ winpty docker exec -it php-apache.com curl https://php-apache.com/
hello
```

- openssl

```html
$ winpty docker exec -it php-apache.com openssl s_client -quiet -connect php-apache.com:443
depth=1 O = mkcert development CA, OU = MYCOMPUTER\\norit@MyComputer (Noritaka IZUMI), CN = mkcert MYCOMPUTER\\norit@MyComputer (Noritaka IZUMI)
verify return:1
depth=0 O = mkcert development certificate, OU = MYCOMPUTER\\norit@MyComputer (Noritaka IZUMI)
verify return:1
...
```

## :memo: 最終結果と考察

あらためてそれぞれの状態での結果を一つの表にまとめてみると、<mark>file_get_contents と openssl で結果が一致</mark>していることがわかります :bulb:

|No.|方法|<mark>最初の状態</mark>|状態 2|状態 3|状態 4|
|---|---|:---:|:---:|:---:|:---:|
|1|file_get_contents|<span style="color: blue;">失敗</span>|<span style="color: blue;">失敗</span>|**<span style="color: red;">成功</span>**|**<span style="color: red;">成功</span>**|
|2|curl|<span style="color: blue;">失敗</span>|**<span style="color: red;">成功</span>**|<span style="color: blue;">失敗</span>|**<span style="color: red;">成功</span>**|
|3|openssl|<span style="color: blue;">失敗</span>|<span style="color: blue;">失敗</span>|**<span style="color: red;">成功</span>**|**<span style="color: red;">成功</span>**|

OS や PHP, Apache の状態によっては結果が変わる可能性もあるため、一概に file_get_contents と openssl の疎通可否を必要十分条件ということはできませんが、少なくともある程度有益な情報が得られたとは思います :thinking_face:

もし、同じエラーに詰まった方がいらっしゃいましたら、参考にしていただけると幸いです :wink:
