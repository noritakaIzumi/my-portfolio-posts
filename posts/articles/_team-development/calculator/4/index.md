---
title: "電卓をチームで開発してみよう～4. 開発者 2 のコーディング"
date: 2021-02-12T00:00:00+09:00
description: 電卓をチーム開発する際の役割分担を考える記事です。
menu:
  sidebar:
    name: 4. 開発者 2
    identifier: team-development-calculator-4
    parent: team-development-calculator
    weight: 4
---

今回は開発者 2 について、コードを書いていきます。

---

作業するブランチは `developer-2` です。

開発者 1 の繰り返しになる部分もありますが、あらかじめ決めた条件を確認します。

> - 計算ロジックのファイル名は `main.js`
> - 数字を表示するエリアの id 属性は `number-display`
> - ボタンを押したときに実行する関数の名前は `sendButtonInfo` で、計算後に結果をディスプレイにセットする。
> - 画面読み込み時、ディスプレイには 0 をセットする
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
> 
> ---
> 
> 計算ロジック側で必要な情報
> |情報|説明|
> |---|---|
> |ディスプレイに表示されている数字|ディスプレイに数字を設定するために必要です。|
> |直前の計算結果|数字が入力されている間も、計算をするために記憶しておく必要があります。|
> |最後に押された四則演算のボタンの情報|次に四則演算やイコールのボタンが押されたとき、計算をするために必要です。|
> |数字を入力している最中かどうか|計算結果が出た直後と、数字を入力している最中かどうかでディスプレイの数字の作り方が異なります。|
> |小数点ボタンが押されたかどうか|ディスプレイに 0 が表示されているときに 0 を押すと、小数点があれば `0.0` となりますが、小数点がない場合 `0` のままです。|

これらをもとに実装します。

```js
NumberDisplay = 'number-display';
Numbers = {
    0: '0',
    1: '1',
    2: '2',
    3: '3',
    4: '4',
    5: '5',
    6: '6',
    7: '7',
    8: '8',
    9: '9',
};
Dot = '.';
Operation = {
    add: '+',
    sub: '-',
    mul: '*',
    div: '/',
    eq: '=',
};

class Calculator {
    /**
     * ディスプレイに表示されている数字。
     * @type {string}
     */
    displayNumber = '0';
    /**
     * 直前の計算結果。
     * @type {number}
     */
    _lastResult = 0;
    /**
     * 最後に押された四則演算のボタン。
     * @type {string}
     */
    lastOperation = Operation.eq;
    /**
     * 数字を入力している最中かどうか。
     * @type {boolean}
     */
    numberBeingInput = false;

    /**
     * 直前の計算結果を文字列にキャストして取得する。
     * @returns {string}
     */
    get lastResult() {
        return this._lastResult.toString();
    }

    inputNumber(val) {
        if (!this.numberBeingInput || this.displayNumber === Numbers["0"]) {
            this.displayNumber = val;
        } else {
            this.displayNumber += val;
        }
        this.numberBeingInput = true;
    }

    inputDot() {
        if (!this.numberBeingInput) {
            this.displayNumber = Numbers["0"];
        }
        if (this.displayNumber.indexOf(Dot) < 0) {
            this.displayNumber += Dot;
        }
        this.numberBeingInput = true;
    }

    inputOperation(val) {
        // 末尾にドットが入っていても Number への変換でドットがうまいこと削除されるので、手動でドットを取り除かない
        switch (this.lastOperation) {
            case Operation.add:
                this._lastResult += Number(this.displayNumber);
                break;
            case Operation.sub:
                this._lastResult -= Number(this.displayNumber);
                break;
            case Operation.mul:
                this._lastResult *= Number(this.displayNumber);
                break;
            case Operation.div:
                this._lastResult /= Number(this.displayNumber);
                break;
            default: // イコールの場合はここを通る
                this._lastResult = Number(this.displayNumber);
        }
        this.displayNumber = this.lastResult;
        this.lastOperation = val;
        this.numberBeingInput = false;
    }

    /**
     * ボタンを入力する。四則演算のボタンが押された場合は計算する。
     * @param {string} val - ボタンの値
     */
    inputButton(val) {
        // 下の 3 パターンは互いに排反
        if (Object.values(Operation).indexOf(val) >= 0) {
            this.inputOperation(val);
        } else if (val === Dot) {
            this.inputDot();
        } else if (Object.values(Numbers).indexOf(val) >= 0) {
            this.inputNumber(val);
        }
    }
}

calculator = new Calculator();
document.getElementById(NumberDisplay).innerText = Numbers["0"];

/**
 * 何のボタンが押されたかを送信する。
 * その後、電卓で計算がある場合は計算し、結果をディスプレイにセットする。
 * @param {string} val - ボタン
 */
function sendButtonInfo(val) {
    calculator.inputButton(val);
    document.getElementById(NumberDisplay).innerText = this.displayNumber;
}
```

これをコミットしておきます。

---

次回は動作確認とまとめです。
