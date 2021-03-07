---
title: "Firebase Emulator のセキュリティルール設定でハマったポイント"
emoji: "📛"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Firebase", "RealtimeDatabase"]
published: true
---

# 前提条件

以下の環境及びバージョンで動作確認しています

- firebase-tools: v9.5.0
- firebase-database-emulator: v4.7.2
- 上記を Docker コンテナにインストールして `curl` で動作確認

```Dockerfile:Dockerfile
FROM openjdk:11-jre-slim

RUN apt-get update && apt-get install -y wget
RUN wget -O firebase https://firebase.tools/bin/linux/latest \
    && chmod u+x ./firebase \
    && mv firebase /usr/local/bin

EXPOSE 9000
ENTRYPOINT ["firebase", "emulators:start"]
```

# Emulator の Realtime Database インスタンスを切り替える方法

エンドポイントに `ns` クエリパラメータを付与することでインスタンスを切り替えられます。 [^specify-database-instance]

```
http://localhost:9000/path/to/my/data.json?ns=<database_name>
```

# セキュリティルールが適用される先は 1 インスタンスのみ

Emulator の Realtime Database にセキュリティルールを設定する際は `firebase.json` でセキュリティルールを記述したファイルを指定します。 [^setting-emulator-security-rules]
しかし、ここで指定したセキュリティルールは **1 インスタンスのみ** 適用されます。
具体的には、`<project-name>-default-rtdb` にのみ適用されます。

つまり、

- Emulator の Realtime Database のエンドポイントが `http://localhost:9000/`
- Project name が `example` のとき

| Endpoint                                              | セキュリティルール |
| ----------------------------------------------------- | ------------------ |
| `http://localhost:9000/.json?ns=example-default-rtdb` | 適用される         |
| `http://localhost:9000/.json?ns=example`              | 適用されない       |
| `http://localhost:9000/.json`                         | 適用されない       |

となります。

## 実際にセキュリティルールが適用されているデータベース名が知りたい

Emulator のログを確認すると良いです。
Emulator 起動中に読み込まれたセキュリティルールのファイルを更新すると、 Emulator はセキュリティルールを再適用してくれます。
この際、セキュリティルールが適用されたデータベース名が表示されます。
以下のログが出た場合、 `example-default-rtdb` にセキュリティルールが適用されています。

```
i  database: Change detected, updating rules for example-default-rtdb...
✔  database: Rules updated.
```

# インスタンスごとのセキュリティルールを確認する方法

Emulator の Realtime Database のセキュリティルールを確認する際は、 `owner` を管理者トークンとして渡す必要があります。 [^admin-auth-token]
これは以下のどちらかの方法で渡せば OK です。

- `access_token` クエリパラメータで渡す [^access-token]

```bash
$ curl -X GET 'http://localhost:9000/.settings/rules.json?access_token=owner'
{
    "rules": {
        ".read": true,
        ".write": true
    }
}
```

- `Authorization` ヘッダーで Bearer token として渡す

```bash
$ curl -X GET 'http://localhost:9000/.settings/rules.json' \
       -H 'Authorization: Bearer token'
{
    "rules": {
        ".read": true,
        ".write": true
    }
}
```

# まとめ

- Emulator の Realtime Database でセキュリティルールが適用されるインスタンスは `<project-name>-default-rtdb`
- インスタンスごとのセキュリティルールを確認する際は、 `owner` を管理者トークンとして渡す
  - `access_token` クエリパラメータ、または `Authorization` ヘッダに Bearer token として渡す

[^specify-database-instance]: https://firebase.google.com/docs/rules/unit-tests#interacting_with_the_emulator
[^setting-emulator-security-rules]: https://firebase.google.com/docs/emulator-suite/install_and_configure#security_rules_configuration
[^admin-auth-token]: https://firebase.google.com/docs/rules/unit-tests#differences_between_the_emulator_and_production
[^access-token]: https://firebase.google.com/docs/database/rest/app-management
