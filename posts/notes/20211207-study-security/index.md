---
title: セキュリティを勉強すればいい先輩になれる？
description: セキュリティを勉強すればいい先輩になれる？
date: 2021-12-07 23:25:00+09:00
---

タイトルの通り、なれるんじゃないかと思っています。

先輩（後輩）とセキュリティ（攻撃手法）の共通点は、「絶えず変化していくということ」です。

先輩後輩ということに関して。ゆとり世代（私も含まれている）を境に若い方の思想は大きく変化してきたと思います。  
よく聴く歌は井上陽水から森山直太朗へ、ストレス耐性は低くなり（と言われている）、 
「社員は家族」から「プライベート重視」へ・・・いい先輩は、自分が思うに「後輩のことを分かってあげられる人」です。  
刻一刻と思想が変化していく若者世代を継続して理解し続けるためには、若者世代のことを常に勉強しなければなりません。

セキュリティ（攻撃手法）に関して。攻撃手法も様々なものが出てきました。  
トロイの木馬からパスワード総当たり攻撃、SQL インジェクション、DDoS攻撃など・・・  
セキュリティのスペシャリストは、日々そういった新しい攻撃手法あるいは既存ソフトウェアの脆弱性など、  
あらゆるところに常にアンテナを張って学習していかなければならないのだと考えています。

そもそもセキュリティに関心を持ち始めたのは、昔、DevOps について調べようと思った時のことでした。  
言葉遊びが好きなもんなので、DevOps ならぬ DevOops（oops: おっとっと）とかあるんじゃね？  
と思って調べてみたらあるんだなこれが。

https://devoops.ru/en/

DevOps のカンファレンスだそうです。そして検索結果を下に行くとなにやら面白そうなサイトが。

https://www.hackthebox.com/

Hack The Box (htb) という、仮想マシンに攻撃して遊ぼうというゲームがあったのです。  
試しにやってみたけどそもそも会員登録が難しかった。とにかくゲーム要素満載でした。  
会員登録をなんとか終わらせて実際に遊んでみるも、さっぱりわからず放置状態。  
そんな私に htb は救いの手を差し伸べます。

https://academy.hackthebox.com/

なんと、HTB Academy という学習用サイトができたのです！これはやるしかない、と。  
中身は基本的な Linux コマンドの使い方から Web リクエストの仕組み、  
JavaScript でセキュアなコーディングをするために何をすればいいかなど、非常に多岐にわたっています。

課金要素もあるんですが、初めの部分は無料で学習できるのでぜひやってみてください。  
そんな HTB Academy で学習していて、印象に残った話題がありました。

```text
下の数式を解いてください。
□ + □ = 10
```

HTB Academy "Get Started" より

みなさんはすぐに以下のような回答を思いつくでしょう。

```text
5 + 5 = 10
```

でも、この答えが何通りあるかを考えたことがありますか？と。え、自然数入れたら 10 数通りですか？と。

こんな解き方もありますよね？

```text
-9999 + 10009 = 10
```

```text
15 + (-5) = 10
```

でもでも実は、こんな解き方もあります。

```text
1 + 1 = 2
```

```text
10 - 0 = 10
```

```text
0 = 0
```

え？そんなものはないって？あるんです。

開発者ツールで「+」を「-」に変えて解けばいいんです。もっと言えば、全部消して好きな数式に書き換えればいいんです。

ね、いろんな解き方があるでしょ？思い込みはダメですよ。

・・・という趣旨のセクションがあって、目からうろこでした。  
常に新しい視点を持って向き合わなければいけないのがセキュリティだということなんでしょう。  
（セキュリティ学習者の心得？よくわかっていない）それは後輩とのかかわり方でも同じことがいえると思います。

みなさんもセキュリティを勉強していい先輩になりましょう（強引）