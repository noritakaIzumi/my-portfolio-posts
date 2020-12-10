---
title: "Windows のシャットダウン時にうっかり VirtualBox VM のシャットダウンを忘れてしまう方へ"
date: 2020-12-26T07:00:00+09:00
description: VirtualBox VM のシャットダウンを行うスクリプトを作成します。
news_keywords:
  - VirtualBox
  - シャットダウン
---

Windows で VirtualBox を使っている方にはわかるかもしれないのですが、普段の業務終了時に VM を終了せずに PC をシャットダウンしようとして↓のようなメッセージが表示されたことはないですか？ :smiling_imp:

> VirtualBox Interface
> このアプリがシャットダウンを妨げています。

私もこれが出ると「うわ・・・またやっちまった・・・」となります :sweat:

そして一旦デスクトップに戻りタスクマネージャを開き `VirtualBox Interface` を終了し・・・

せっかく早く帰れると思ったのに :pensive:

今回は、そんなお悩みをシェルスクリプトで解決してみようと思います :blue_heart:

## 対策？

対策は簡単です :bulb:

ズバリ、

**PC をシャットダウンする前に VM をすべて終了する**

です :wink:

というか、これを忘れてしまうから困っているわけなんですが :sweat_smile:

## シャットダウンのコマンドを用意しよう

Windows のシャットダウンはコマンドで実行することができます。

```bash
shutdown -s -t 0
```

`-s` はシャットダウン、 `-t 0` は処理が行われるまでの時間（秒）を表しています。
この例の場合だと、「0 秒経ったらシャットダウン」、つまり即時シャットダウンのことを意味しています。

シャットダウンコマンドについては [こちらの記事](https://www.atmarkit.co.jp/ait/articles/0601/14/news013.html) を参考にさせていただきました。

## 起動中の VM をすべて終了するコマンドを用意しよう

VirtualBox には VM の操作をコマンドラインで行える `VBoxManage` が用意されています。
これを利用して、起動中の VM を一覧で表示し、それをシャットダウンの引数に渡すようにします。

なお、 `bash: VBoxManage: command not found` になるよという方は、 `VBoxManage` を環境変数に登録してください。

`VBoxManage` の場所に関しては [【VirtualBox】VBoxManageを使う - Qiita](https://qiita.com/shinaK/items/ddd7fb51d26f641af041) が参考になるかと思います。

```bash
VBoxManage list runningvms | cut -d' ' -f1 | xargs -I{} VBoxManage controlvm {} acpipowerbutton
```

なお、 VBoxManage のコマンドについては [Virtualboxの仮想マシンをコマンドラインから操作 - Qiita](https://qiita.com/tukiyo3/items/5ecea7f95cb961f07194) を参考にさせていただきました。

## スクリプトを完成させよう

必要なコマンドはそろいましたので、いよいよスクリプトを作っていこうと思います。
方針は簡単で、起動中の VM をすべて終了してから Windows をシャットダウンさせるだけなのですが、落とし穴が一つあります。
**VM を終了するコマンドは、 VM のシャットダウンシグナルを送るのみですぐに処理が終わり、 VM のシャットダウンを待ちません。**
なので、上記 2 つのコマンドをそのまま並べて実行すると、また VM が終了しないまま PC のシャットダウンに入ってしまいます :cold_sweat:

これを回避するために、起動中の VM の一覧を定期的にコマンドで確認し、出力がなくなってから PC のシャットダウンに入るような処理を記述します。

```bash:shutdown_vms.sh
#!/usr/bin/env bash
INTERVAL=3 # 起動中の VM 一覧を確認する周期

VBoxManage list runningvms | cut -d' ' -f1 | xargs -I{} VBoxManage controlvm {} acpipowerbutton

while [ $(VBoxManage list runningvms | wc -l) -ne 0 ]
do
    echo "shutting down VMs..."
    sleep ${INTERVAL}
done
```

そして、シャットダウン時に実行するファイルを作成します。

```bash:shutdown_entrypoint.sh
#!/usr/bin/env bash

./shutdown_vms.sh
wait

echo "bye"
sleep 1

shutdown -s -t 0
```

これで完成です :exclamation: :relaxed:

## 動作確認しよう

1. VM をいくつか立ち上げておきます。
1. 上で作成した `shutdown_vms.sh` と `shutdown_entrypoint.sh` を同じディレクトリに配置します。
1. エクスプローラの画面で `shutdown_entrypoint.sh` をダブルクリック、あるいは Git Bash で `./shutdown_entrypoint.sh` を実行します。

こんな感じで出力されていたらうまくいっています :wink:

```html
shutting down VMs...
shutting down VMs...
shutting down VMs...
bye
```

みなさんも身の回りの面倒なことを自動化して快適に開発し、快適におうちに帰りましょう :house_with_garden: :yum:
