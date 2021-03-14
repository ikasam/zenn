---
title: "Laravel で null ドライバを .env から指定するときは \"'null'\" と書こう"
emoji: "🈚️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["laravel"]
published: true
---

タイトルがすべてですが、例えばログ設定で `'null'` チャンネル [^logging-null] を使いたいときはこう書きましょう

```bash:.env
LOG_CHANNEL="'null'"
# or
LOG_CHANNEL='"null"'
```

# なんで？

`env()` ヘルパーが `.env` に書かれている文字列の `null` を `NULL` 値として解釈してしまうからです

実際に `.env` を以下のように書いて [^another-quoted]

```bash:.env
LOG_CHANNEL=null
```

`env()` ヘルパーの挙動を確かめると

```php
>>> var_dump(getenv('LOG_CHANNEL'))
string(4) "null"
>>> var_dump(env('LOG_CHANNEL'))
NULL
```

`NULL` 値が返ってきます

`config/logging.php` で `'null'` チャンネルを利用した際はログの出力が一切ないことを期待していますが、
この状態でログ出力を実際に試すと emergency logger が利用され、 `storage/logs/laravel.log` にログが出力されてしまいます

```apacheconf:storage/logs/laravel.log
[2021-03-14 15:46:10] laravel.EMERGENCY: Unable to create configured logger. Using emergency logger. {"exception":"[object] (InvalidArgumentException(code: 0): Log [] is not defined. at /var/www/html/vendor/laravel/framework/src/Illuminate/Log/LogManager.php:192)
[stacktrace]
#0 /var/www/html/vendor/laravel/framework/src/Illuminate/Log/LogManager.php(118): Illuminate\\Log\\LogManager->resolve()
#1 /var/www/html/vendor/laravel/framework/src/Illuminate/Log/LogManager.php(98): Illuminate\\Log\\LogManager->get()
#2 /var/www/html/vendor/laravel/framework/src/Illuminate/Log/LogManager.php(608): Illuminate\\Log\\LogManager->driver()
#3 /var/www/html/vendor/laravel/framework/src/Illuminate/Support/Facades/Facade.php(261): Illuminate\\Log\\LogManager->debug()
...(snip)...
[2021-03-14 15:46:10] laravel.DEBUG: hoge  
```

# `"'null'"` にするとどうなるの？

環境変数は `string(6) "'null'"` として解釈されますが、 `env()` ヘルパーは `string(4) "null"` を返します

```php
>>> var_dump(getenv('LOG_CHANNEL'))
string(6) "'null'"
>>> var_dump(env('LOG_CHANNEL'))
string(4) "null"
```

これは `env()` ヘルパーの実装がよしなにやってくれているからです
https://github.com/laravel/framework/blob/8.x/src/Illuminate/Support/Env.php#L93-L95

これで `'null'` チャンネルが正しく使われて、ログがどこにも出力されなくなります

```php
>>> Log::debug('hoge')
=> null
>>> # どこにもログが出力されない
```

# ログの `'null'` チャンネルっていつ使うの？

テスト実行時など、ログを出したくないときに便利です

# `'null'` 以外のドライバ名にしないの？

- ログの `'null'` チャンネル名は当初 `'none'` としてプロポーザルが出されていました [^proposal]
- 最終的に、 `null` ドライバなんだから `null` という名前をつけたほうが一貫性があって良い、で落ち着いたようです [^consistency]
- この手の問題は、将来的に `null` を残しつつ別名もサポートしたほうが便利だね、という話も出ていましたが、まだ実現はしていないようです [^alternative]

# 参考

- [[5.8] BUG: Fix quoted environment variable parsing](https://github.com/laravel/framework/pull/27691)
- [[Proposal] Null logging channel #1806](https://github.com/laravel/ideas/issues/1806)
- [[6.x] Add 'null' logging channel #5106](https://github.com/laravel/laravel/pull/5106)

[^logging-null]: ログの `'null'` チャネルは [6.2.0](https://github.com/laravel/laravel/blob/6.x/CHANGELOG.md#v620-2019-10-08) で追加されました
[^another-quoted]: `LOG_CHANNEL='null'` や `LOG_CHANNEL="null"` も同様
[^proposal]: https://github.com/laravel/ideas/issues/1806
[^consistency]: https://github.com/laravel/laravel/pull/5106#issuecomment-531220153
[^alternative]: https://github.com/laravel/framework/pull/27691#issuecomment-467872248
