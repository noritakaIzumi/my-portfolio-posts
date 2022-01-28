---
title: "電卓をチームで開発してみよう～3. 開発者 1 のコーディング"
date: 2021-02-05T00:00:00+09:00
description: 電卓をチーム開発する際の役割分担を考える記事です。
menu:
  sidebar:
    name: 3. 開発者 1
    identifier: team-development-calculator-3
    parent: team-development-calculator
    weight: 3
---

ここからは実際に開発をしていきます。
GitHub で空のリポジトリを作成し、`developer-1` ブランチと `developer-2` ブランチを作成します。

まず、開発者 1 について、コードを書いていきます。

---

作業するブランチは `developer-1` です。

あらかじめ決めた条件を確認します。

> - 計算ロジックのファイル名は `main.js`
> - 数字を表示するエリアの id 属性は `number-display`
> - ボタンを押したときに実行する関数の名前は `sendButtonInfo`
> - `sendButtonInfo` の引数は以下の通り。
> 
> ---
> 
> `sendButtonInfo` の引数（文字列型）
> 
> |ボタン|引数|
> |:---:|:---:|
> |0|`'0'`|
> |1|`'1'`|
> |2|`'2'`|
> |3|`'3'`|
> |4|`'4'`|
> |5|`'5'`|
> |6|`'6'`|
> |7|`'7'`|
> |8|`'8'`|
> |9|`'9'`|
> |.|`'.'`|
> |＋|`'+'`|
> |－|`'-'`|
> |×|`'*'`|
> |÷|`'/'`|
> |=|`'='`|

これらをもとにして画面を作成します。

`index.html`

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Calculator</title>
</head>
<body>
<h1>Calculator</h1>
<hr>
<div>
    <h2 id="number-display"></h2>
    <div>
        <div>
            <button onclick="sendButtonInfo('7');">7</button>
            <button onclick="sendButtonInfo('8');">8</button>
            <button onclick="sendButtonInfo('9');">9</button>
            <button onclick="sendButtonInfo('/');">÷</button>
        </div>
        <div>
            <button onclick="sendButtonInfo('4');">4</button>
            <button onclick="sendButtonInfo('5');">5</button>
            <button onclick="sendButtonInfo('6');">6</button>
            <button onclick="sendButtonInfo('*');">×</button>
        </div>
        <div>
            <button onclick="sendButtonInfo('1');">1</button>
            <button onclick="sendButtonInfo('2');">2</button>
            <button onclick="sendButtonInfo('3');">3</button>
            <button onclick="sendButtonInfo('-');">－</button>
        </div>
        <div>
            <button onclick="sendButtonInfo('0');">0</button>
            <button onclick="sendButtonInfo('.');">.</button>
            <button onclick="sendButtonInfo('=');">=</button>
            <button onclick="sendButtonInfo('+');">＋</button>
        </div>
    </div>
</div>
<script src="main.js"></script>
</body>
</html>
```

これをコミットしておきます。

---

次回は開発者 2 のコードを書いていきます。
