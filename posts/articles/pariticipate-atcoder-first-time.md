---
title: "AtCoder のプログラミングコンテストに初めて参加しました"
date: 2020-12-09T23:57:19+09:00
description: AtCoder のプログラミングコンテストに初めて参加しました。感想やコードを書く上で気を付けたことなどを書きます。
news_keywords:
  - AtCoder
  - Python
  - Programming
  - プログラミング
---

始めて AtCoder のプログラミングコンテストに参加したので、その感想を書きます。

今回参加したコンテストはこちらです。

[鹿島建設プログラミングコンテスト2020（AtCoder Regular Contest 110） - AtCoder](https://atcoder.jp/contests/arc110)

## 結果

6 問中 2 問完答して 3500 点満点中 700 点でした。
解けなかった残りの問題はまだ見てもいない状態ですが、手を付けた問題をしっかり解けたのは良かったと思います。

## どんな感じでコードを書いていたか

以下の点に気を付けて書きました。

- ファイル内に問題の概要をコメントで書いておく
- メインで実行する関数の引数についてはドキュメントを記載する
- エラーハンドリングを記述する
- テストコードを書く

例として、入力された自然数 n に対して、 2 から n のどれで割っても 1 余る 10^13 以下の整数を求める問題のコードです。

```python
# -*- coding: utf-8 -*-

# A - Redundant Redundancy

"""
N: 整数
2, 3, ..., N のどれで割っても 1 余る N 以上 10^13 以下の整数を出力する
"""

import sys


def solve(num: int) -> int:
    """問題を解く。

    Args:
        num: N

    Returns:
        2, 3, ..., N のどれで割っても 1 余る N 以上 10^13 以下の整数

    Examples:
        (1, 2, ..., N の最小公倍数) + 1 はこの問題の答えになる。

    """
    try:
        if num < 2 or num > 30:
            raise ValueError
        return get_least_common_multiple(num) + 1
    except ValueError:
        print('数字の指定がおかしいです', file=sys.stderr)
    return 0


def get_least_common_multiple(num: int) -> int:
    result = 1
    cur1 = 2
    while cur1 <= num:
        x = cur1
        y = result
        cur2 = 2
        while x > 1:
            if x % cur2 == 0:
                x /= cur2
                if y % cur2 == 0:
                    y /= cur2
                    continue
                else:
                    result *= cur2
            cur2 += 1
        cur1 += 1
    return result


def verify_output(num: int) -> bool:
    redundancy = solve(num)
    if redundancy < num or redundancy > 10 ** 13:
        return False
    m = 2
    while m <= num:
        if redundancy % m != 1:
            return False
        m += 1
    else:
        return True


def test():
    assert verify_output(2), 'basic case'
    assert verify_output(3), 'case 1'
    assert verify_output(10), 'case 2'
    assert verify_output(16), 'case 3'
    assert verify_output(30), 'case 4'


def main():
    n = int(input())
    print(solve(n))


if __name__ == '__main__':
    main()
    # test()
```

## 問題を解きたい気持ちが先行する

最初は「10^13 以下の整数」という観点をテストコードに含めておらず、答えを `n! + 1` の形で書いて提出して不正解判定になってしまいました。

テストに条件を盛り込めるものはすべて盛り込むのが大事だと思います。

なぜ最初に盛り込まなかったかというと、おそらく制限時間という妙なプレッシャーのようなものがあり、問題を解いてとにかく早く正解しなければという気持ちが先走っていたからです。

今回は回答時間に 5 分プラスされるという不正解のペナルティがありました。

テストコードと 5 分のペナルティを天秤にかけると、自分はテストコードかなと思いました（人によります）。

まだ慣れていないということもあるので、慣れるまでは一発で正解することを目標に、これから取り組んでいきたいです。
