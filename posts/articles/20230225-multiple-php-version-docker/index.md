---
title: Docker を利用して複数の PHP バージョンを切り替える
date: 2023-02-25T00:00:00+09:00
description: Docker と bash のエイリアスを使用して PHP のバージョンを切り替えます。
---

PHP で複数バージョンを切り替える方法は調べるとたくさん出てくるのですが、今回は Docker を利用してバージョンを切り替える方法を考えてみたのでご紹介します。

なお、今回は WSL の Ubuntu 22.04.1 LTS を使用します。

## Dockerfile の作成

各バージョンの設定は Dockerfile によって管理します。

まずは Dockerfile を作成するためのディレクトリを作成します。

```bash
# 今回はホームディレクトリ直下に作成します
mkdir -p ~/docker-php
cd ~/docker-php
```

次に Dockerfile を作成します。今回は最新、8.1、8.0 の 3 種類です。

- 最新 (`Dockerfile.latest`)

```dockerfile
FROM php:latest
```

- 8.1 (`Dockerfile.8_1`)

```dockerfile
FROM php:8.1
```

- 8.0 (`Dockerfile.8_0`)

```dockerfile
FROM php:8.0
```

## エイリアスの作成

今回は `~/.bash_aliases` に以下を追記します。
環境によっては `~/.bashrc` や `~/.bash_profile` となることもあります。

```bash
# Docker
# ファイル書き込みの際に root 権限にならないよう、ユーザを指定する
alias dkrun="docker run --rm -it -u $(id -u):$(id -g)"
alias dkrunroot="docker run --rm -it"

# PHP (docker)
DOCKER_PHP_DIR=~/docker-php
DOCKER_PHP_REPO=local/php
docker_php() {
  version=$1
  docker_tag=local/php:${version}
  snake_case_version=$(echo "${version}" | tr '.' '_')

  # イメージがない場合や、アップデートする場合はビルドする
  if [ -z "$(docker images --quiet --filter=reference=${DOCKER_PHP_REPO}:"${version}")" ] || [ "$2" = "--update" ]; then
    docker build --no-cache --pull --file ${DOCKER_PHP_DIR}/Dockerfile."${snake_case_version}" --tag "${docker_tag}" .
  fi

  # アップデートの場合はその後コマンドを実行しない
  if [ "$2" = "--update" ]; then
    return
  fi

  # コマンドを実行する
  if [ "$2" = "bash" ]; then
    dkrunroot -v "$(pwd)":/app -w /app "${docker_tag}" "${@:2}"
  elif [ -z "$2" ]; then
    dkrun -v "$(pwd)":/app -w /app "${docker_tag}"
  else
    dkrun -v "$(pwd)":/app -w /app "${docker_tag}" php "${@:2}"
  fi
}
# 指定されたディレクトリにある Dockerfile の一覧を読み取り、エイリアスを設定する
# すでにインストールされている PHP と区別するために dphp と付けています。
for snake_case_version in "${DOCKER_PHP_DIR}"/Dockerfile.*; do
  snake_case_version=$(echo "$snake_case_version" | rev | cut -d'/' -f1 | rev | sed 's/^Dockerfile\.//')
  if [ "$snake_case_version" == "latest" ]; then
    alias dphp='docker_php latest'
  else
    alias "dphp$(echo "$snake_case_version" | tr -d '_')=docker_php $(echo "$snake_case_version" | tr '_' '.')"
  fi
done
```

追記が終わったら、シェルを再起動するか `source ~/.bash_aliases` でエイリアスを読み込みます。

## 実行してみる

最新の PHP が実行できるか試してみましょう。

```bash
dphp --version
```

ビルドが始まり、以下のように PHP のバージョンが表示されれば OK です。

```text
PHP 8.2.3 (cli) (built: Feb 14 2023 20:22:21) (NTS)
Copyright (c) The PHP Group
Zend Engine v4.2.3, Copyright (c) Zend Technologies
```

同様に `dphp81 --version` と `dphp80 --version` も試してみましょう。

## 拡張モジュールの追加

拡張モジュールを追加するには `Dockerfile` を編集します。
今回は zip モジュールを追加してみましょう。

私が試した時点では zip モジュールは標準で入っていないので、以下のコマンドを実行しても何も出力されないはずです。

```bash
dphp -m | grep zip
```

引数に `bash` を渡すことで、コンテナにログインし、必要なコマンドをリハーサルします。

```bash
dphp bash
```

コマンドが決まったら `Dockerfile` へ追記します。

```dockerfile
FROM php:latest

RUN apt update && \
    apt install -y libzip-dev && \
    docker-php-ext-install zip
```

イメージをアップデートします。

```bash
dphp --update
```

もう一度モジュールの存在を確認します。

```bash
dphp -m | grep zip
```

`zip` と出てくれば zip モジュールがインストールされています！

## アンインストール

PHP の特定バージョンをアンインストールする場合、Docker イメージを削除するだけです。

例えば 8.1 をアンインストールする場合は以下のコマンドを実行します。

```bash
docker rmi $(docker images --quiet --filter=reference=local/php:8.1)
```

キャッシュなども含めて完全に削除する場合は `docker system prune` を実行してください。
