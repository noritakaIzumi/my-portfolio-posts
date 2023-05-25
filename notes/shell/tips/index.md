---
title: Tips
weight: 10
menu:
  notes:
    name: Tips
    identifier: notes-shell-tips
    parent: notes-shell
    weight: 10
---

{{< note title="Random password" >}}

外部サイトに頼らず、OS からパスワードを生成する仕組み。  
外部サイトがコンプライアンスに違反していた場合に、この方法に乗り換えることが可能です。

Ref: https://serverfault.com/questions/283294/how-to-read-in-n-random-characters-from-dev-urandom

```bash
# the first argument: char count
randpw() {
  local COUNT=$1
  head -c 1000 /dev/urandom | tr -dc '!-~' | fold -w "${COUNT}" | head -n 1
}

```

12 文字のパスワードを生成する場合は `randpw 12` を実行します。

{{< /note >}}
