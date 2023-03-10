---
title: 🧑‍💻コーディングポリシー
description: コーディングで気を付けるところやこだわっているところ。
date: 2023-03-05 07:38:00+09:00
menu:
  sidebar:
    name: 🧑‍💻コーディングポリシー
    identifier: coding-policy
    weight: 11
---

{{< alert type="primary" >}}
MVC の C は画面スクロールなしで全体が見える程度の長さを目標に (40 ～ 50 行程度)
{{</ alert >}}

{{< alert type="primary" >}}
変数のスコープを狭くする -> 適宜 Closure や即時実行関数の形を書く
{{</ alert >}}

{{< alert type="primary" >}}
関数名の命名に悩むのであれば、複数の関数に分離できないか考える
{{</ alert >}}

{{< alert type="primary" >}}
変数からメソッドを呼び出したときの見た目が英語っぽくなるように
{{</ alert >}}

{{< alert type="primary" >}}
PHPUnit の C0 カバレッジ計測で C1 同等の計測ができるようにコーディングする
- if には必ず else を (return や強制終了している場合を除く)
- for 文にはカウンターを付けて、カウンターが 0 の時の分岐を記述する (その後の処理で考慮できている場合を除く)
{{</ alert >}}
