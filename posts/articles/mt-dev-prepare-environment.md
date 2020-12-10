---
title: "Movable Type 公式の開発環境を Docker で構築してみた"
date: 2020-12-10T23:34:39+09:00
description: Movable Type が公式で MT の開発環境を構築できる mt-dev を公開していたので、これを使って Docker による MT の環境構築をしてみました。
news_keywords:
  - Movable Type
  - Docker
  - mt-dev
---

ブログや Web サイトなどの作成で WordPress と並んで使用されている CMS に [Movable Type](https://www.movabletype.jp/) があります。

私も業務で使用するようになり、環境構築について調べていたのですが、 Movable Type 公式より `mt-dev` という環境構築のためのリポジトリが公開されていました[^1]ので、それを使って環境を構築してみたいと思います。

[^1]: [MTの開発環境を簡単に作れる mt-dev を公開しました - ブログ | CMSプラットフォーム Movable Type ドキュメントサイト](https://www.movabletype.jp/blog/mt-dev.html)

リポジトリは [こちら](https://github.com/movabletype/mt-dev) に公開されています。

## ホスト PC の環境

- Windows 10 Pro
- Git
- Docker (Docker Desktop for Windows)

## 構築手順

#### Git リポジトリをプルする

```bash
$ git clone git@github.com:movabletype/mt-dev.git mt-dev
$ cd mt-dev
```

#### 個人無償版ライセンスをダウンロードする

Movable Type 自体は有料のサービスなのですが、個人利用で非営利目的の場合は「個人無償版ライセンス」が利用可能です。

[Movable Type 個人無償版ダウンロード](https://www.sixapart.jp/inquiry/movabletype/personal_download.html) にアクセスして必要事項を入力し、「上記に同意して申し込む」をクリックします。

![Movable Type Download](/images/posts/movable-type-download.png)

申し込みが完了すると、入力したメールアドレスにダウンロードページ URL の書いたメールが届きます。

今回は Movable Type 7 をダウンロードしてみます。

ダウンロードしたら zip ファイルを `mt-dev` の `archive` ディレクトリの中に移動させます。

#### make と Perl をインストールする

Movable Type の環境構築中に make コマンドと Perl を使用しますので、あらかじめインストールしておいてください。

（インストール方法はここでは省略します）

インストールされて PATH に登録されていれば、以下のように make および Perl のパスが表示されるはずです。

```bash
$ which make
/c/ProgramData/chocolatey/bin/make

$ which perl
/usr/bin/perl
```

#### スクリプトを修正する

この後のスクリプトは `Git Bash` を使って実行するので、それに合わせてスクリプトを修正します。

`./bin/extract-archive` を以下のように 2 箇所修正します。

27 行目

```diff
-    docker run --rm -v $dest:/dest -v $path:/archive/$name -v $script_dir/$script:/usr/local/bin/$script -w /dest busybox:uclibc /usr/local/bin/$script /dest /archive/$name
+    docker run --rm -v $dest:/dest -v /$(pwd)/archive/$name://archive/$name -v /$script_dir/$script://usr/local/bin/$script -w //dest busybox:uclibc //usr/local/bin/$script //dest //archive/$name
```

30 行目

```diff
-docker run --rm -v $dest:/dest busybox:uclibc ls /dest/mt.cgi > /dev/null
+docker run --rm -v $dest:/dest busybox:uclibc ls //dest/mt.cgi > /dev/null
```

`./Makefile` を以下のように以下のように 2 箇所修正します。

76 行目

```diff
-		${_DC} exec $$opt db mysql -uroot -ppassword -h127.0.0.1 ${MYSQL_COMMAND_ARGS}
+		winpty ${_DC} exec $$opt db mysql -uroot -ppassword -h127.0.0.1 ${MYSQL_COMMAND_ARGS}
```

96 行目

```diff
-	${MAKEFILE_DIR}/bin/extract-archive ${BASE_ARCHIVE_PATH} ${MT_HOME_PATH} $(shell echo ${ARCHIVE} | tr ',' ' ')
+	`pwd`/bin/extract-archive `pwd`/archive ${MT_HOME_PATH} $(shell echo ${ARCHIVE} | tr ',' ' ')
```

`mt-config.cgi-original` を以下のように 2 箇所修正します。

15 行目

```diff
-CGIPath    http://192.168.7.25/cgi-bin/mt/
+CGIPath    http://localhost/cgi-bin/mt/
```

21 行目

```diff
-StaticWebPath    http://192.168.7.25/mt-static
+StaticWebPath    http://localhost/mt-static
```

スクリプト修正にあたり、以下の記事を参考にしています。

- [git-bash for windows を快適にするためのいろいろ - Qiita](https://qiita.com/sixpetals/items/a0784fa3933956463609)
- [Git Bashのttyで怒られないように - Qiita](https://qiita.com/amanoese/items/7b237e8703c3b4c7f001)

#### セットアップコマンドを実行する

スクリプトの修正が終わったら、次のコマンドを実行します。 `ARCHIVE` には先ほどダウンロードした zip ファイルのファイル名を指定します。

`mt-dev` ディレクトリで実行します。

```bash
$ make up ARCHIVE=MT7-R0000.zip
```

```html
$ make up ARCHIVE=MT7-R4703.zip
docker-compose -f ./mt/common.yml -f ./mt/mysql.yml -f ./mt/memcached.yml down --remove-orphans
Removing network mt_default
Network mt_default not found.
C:/ProgramData/chocolatey/lib/make/tools/install/bin/make.exe down-mt-home-volume
make[1]: Entering directory 'C:/Users/norit/IdeaProjects/mt-dev'
mt-dev-mt-home-tmp
make[1]: Leaving directory 'C:/Users/norit/IdeaProjects/mt-dev'
C:/ProgramData/chocolatey/lib/make/tools/install/bin/make.exe down-mt-home-volume
make[1]: Entering directory 'C:/Users/norit/IdeaProjects/mt-dev'
make[1]: Leaving directory 'C:/Users/norit/IdeaProjects/mt-dev'
docker volume create --label mt-dev-mt-home-tmp mt-dev-mt-home-tmp
mt-dev-mt-home-tmp
# TBD: random name?
`pwd`/bin/extract-archive `pwd`/archive mt-dev-mt-home-tmp MT7-R4703.zip
8cc34ffe1a0ad71a8e3faa969e695118 */c/Users/norit/IdeaProjects/mt-dev/archive/MT7-R4703.zip
C:/ProgramData/chocolatey/lib/make/tools/install/bin/make.exe up-common-invoke-docker-compose MT_HOME_PATH=mt-dev-mt-home-tmp  RECIPE="" REPO=""
make[1]: Entering directory 'C:/Users/norit/IdeaProjects/mt-dev'
MT_HOME_PATH=mt-dev-mt-home-tmp
BASE_SITE_PATH=C:/Users/norit/IdeaProjects/mt-dev/site
DOCKER_MT_IMAGE=
DOCKER_HTTPD_IMAGE=
DOCKER_MYSQL_IMAGE=
docker-compose -f ./mt/common.yml -f ./mt/mysql.yml -f ./mt/memcached.yml -f ./mt/cgi.yml  pull
Pulling httpd     ... done
Pulling db        ... done
Pulling memcached ... done
Pulling mt        ... done
docker-compose -f ./mt/common.yml -f ./mt/mysql.yml -f ./mt/memcached.yml -f ./mt/cgi.yml  up -d
Creating network "mt_default" with the default driver
Creating mt_db_1        ... done
Creating mt_mt_1        ... done
Creating mt_memcached_1 ... done
Creating mt_httpd_1     ... done
make[1]: Leaving directory 'C:/Users/norit/IdeaProjects/mt-dev'
```

#### データベースを作成する

データベースの作成コマンドを実行します。

```bash
$ make exec-mysql SQL='CREATE DATABASE IF NOT EXISTS mt /*!40100 DEFAULT CHARACTER SET utf8mb4 */'
```

```html
$ make exec-mysql SQL='CREATE DATABASE IF NOT EXISTS mt /*!40100 DEFAULT CHARACTER SET utf8mb4 */'
opt=""; if ! [ -t 0 ] ; then opt="-T" ; fi; \
        winpty docker-compose -f ./mt/common.yml -f ./mt/mysql.yml -f ./mt/memcached.yml exec $opt db mysql -uroot -ppassword -h127.0.0.1 -e 'CREATE DATABASE IF NOT EXISTS mt /*!40100 DEFAULT CHARACTER SET utf8mb4 */'
mysql: [Warning] Using a password on the command line interface can be insecure.
```

これでコマンドの実行は終わりです。

#### ブラウザで表示を確認する

ブラウザで `http://localhost/cgi-bin/mt/mt.cgi` を開き、以下のようにアカウントの作成画面が表示されることを確認します。

![Movable Type Create Account](/images/posts/movable-type-create-account.png)

アカウントやサイトの作成については説明を省きますので、好きなように遊んでみてください :smile:

## Docker コンテナの終了・次回以降の起動について

今回実行したコマンドのうち、 MySQL のデータベース作成については次回以降不要なので、以後のコマンドは以下のようになります。

コンテナの終了

```bash
$ cd /path/to/mt-dev
$ docker-compose -f ./mt/common.yml -f ./mt/mysql.yml -f ./mt/memcached.yml down --remove-orphans
```

コンテナの起動

```bash
$ cd /path/to/mt-dev
$ make up ARCHIVE=MT7-R4703.zip
```

---

みなさんもぜひ Movable Type を触ってみてください :wink:
