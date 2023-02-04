---
title: An unexpected error は "happened" より "occurred" だとレビューした、あの時の私は正しかったのか
date: 2023-01-20T00:00:00+09:00
description: エラー文言でよく見る "happened" と "occurred"。その違いについてとことん調べます。
---

## TL;DR

{{< alert type="info" >}}
正しいとも言えるし、正しくないとも言えます。
"happened" と "occurred" はエラーを **誰目線で解釈するか** によって変わり、

- プログラム目線で解釈すると "happened"
- 我々開発者目線で解釈すると "occurred"

になるのではないかと考えます。
{{</ alert >}}

## 本編

プログラム内で外部サービスを利用するとき、そこで想定外のエラーが起こるのはつきものです。
try-catch などを用いた例外処理はよく見かけますね。
その処理に関してコードレビューをしたことがあったのですが、今回はそのレビューについて振り返りたいと思います。

### 背景

これは今から 2 年くらい前、私が新卒で入った会社でコードレビューをしていた時のことです。
チームメンバーが実装したコードを見たところ、想定外のエラーを捕捉するところで次のようにログ出力していました。

```
An unexpected error happened.
```

Google 翻訳に入れると「予期しないエラーが発生しました。」となります（記事執筆時点）。

それを見て、当時の私はプルリクに

> An unexpected error は起こるべくして起こるものなので happened よりも occurred がいいと思います。

というコメントを書きました。

### 単語の意味

"happen" と "occur" を 英英辞典で調べてみましょう。
今回は [Oxford Learner's Dictionaries](https://www.oxfordlearnersdictionaries.com/) を使用します。

#### happen

> to take place, especially without being planned (起こること、特に計画されていないことが)

https://www.oxfordlearnersdictionaries.com/definition/english/happen?q=happen

この解釈を見るに、 **何かが偶然起こる** という意味合いが強そうです。
今回取り上げている「想定外のエラー」は予定されていないこととも取れますので、 "happen" は適切に見えます。

#### occur

> to happen (起こること)

https://www.oxfordlearnersdictionaries.com/definition/english/occur?q=occur

to happen って・・・基本同じってことですか？？

### もうちょっと詳しく

同じ、では納得がいかないので、もう少し調べてみます。
Google 検索で「happen vs occur[^1]」と検索します。

こちらの記事では・・・

https://www.differencebetween.com/difference-between-happen-and-vs-occur/

> What is the difference between Happen and Occur?
> - The word happen is used to convey the sense of ‘take place.’
> - The word occur is used in the sense of ‘come into being as an event or a process’.

- happen: 「起こる」ということを伝えるために使われる
- occur: 「出来事または過程として発生する」という意味で使われる

こちらでは・・・

https://peraperabu.com/happen-occur-chigai/

> happenとoccurの違いは、「起こっている」物事が明確にわかっているかそうでないかです。

「過程」というと、物事の進行にすでに組み込まれているような意味合いが出てきますし、起こっている物事が明確なときに "occur" を使うとすると、「プログラムでエラーが起こった」というのはある意味明確です。そうすると "occur" の方が適切にも見えます。

### 私なりの解釈とまとめ

#### `An unexpected error happened.`

プログラム内部で外部サービスを利用する関数があった場合、「接続できなかった」や「リクエストの内容が不正だった」などのように原因がはっきりしているエラーもあるし、「なぜかダメだった」のように原因不明なエラーもあります。
**関数さんにとっては** 、原因不明なエラー (想定外のエラー) が起こったときに「一体何があったんだ？」 "What's happened?" となるんだと思います。で、「んー、よう分からん」ということをログに出力したり、例外を投げ直したりするということですね。

#### `An unexpected error occurred.`

原因がわかっていようがわかっていまいが、「プログラム内部でエラーが起こる」という事象は **開発者から見ても他部署の人から見ても明らか** です。
このログが出たときは「エラーが起こっている」という明確な事実がそこにあるのだから、開発者が一体となって復旧および原因の解明に努めなければならないと思います。

### ちなみに

レビュー文で「起こるべくして起こる」という表現を使いましたが、"occur" にそのような意味を書いている辞書や記事は見つからなかったので、理由としては適切ではないかもです。

[^1]: 何か 2 つのものを比較する際に `vs` を間に入れるのはエンジニアにとって必須スキルの一つです。こちらに [記事](https://qiita.com/dodonki1223/items/955819806297ee554b31#%E3%82%B5%E3%83%BC%E3%83%93%E3%82%B9%E3%82%84%E3%83%A9%E3%82%A4%E3%83%96%E3%83%A9%E3%83%AA%E3%81%AE%E6%AF%94%E8%BC%83%E3%81%AB%E3%81%AF-vs-%E3%82%AD%E3%83%BC%E3%83%AF%E3%83%BC%E3%83%89%E3%82%92%E4%BD%BF%E3%81%88) を書いてくださった方もいらっしゃいます。ぜひ身につけましょう。
