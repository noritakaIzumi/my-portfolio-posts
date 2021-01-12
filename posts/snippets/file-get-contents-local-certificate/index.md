---
title: "ローカル環境の時だけ自己署名証明書のサイトへの file_get_contents を通す"
date: 2020-12-03T12:57:47+09:00
description: file_get_contents でローカル環境の時だけ証明書検証をスキップする設定を入れるコード。
news_keywords:
  - Google News に伝えたいキーワード
  - Qiita のタグのようなものと思ってよい
---

環境を指定する変数 `$environment` を用意し、その値に応じて `file_get_contents` に渡す引数を変えるような関数を作成しました。

これにより、ローカル環境 `$environment = 'local'` のときに証明書の検証を無視するような設定ができるかと思います。

```php
<?php

$environment = 'local'; // prod, stg, local, etc...

function fileGetContents(String $url)
{
    global $environment;
    if ($environment == 'local') {
        $options = array(
            'ssl' => array(
                'verify_peer' => false,
                'verify_peer_name' => false,
            )
        );

        return file_get_contents($url, false, stream_context_create($options));
    }

    return file_get_contents($url);
}

echo fileGetContents("https://google.co.jp");
```

GitHub Gist: [ローカル環境の時だけ自己署名証明書のサイトへの file_get_contents を通す](https://gist.github.com/noritakaIzumi/4f99793930c1931d19affd871c8590b9)

参考: [無効な証明書なサイトに、file_get_contents する方法 - Qiita](https://qiita.com/izanari/items/f4f96e11a2b01af72846)
