---
title: "bashでユーザ名を取得する"
emoji: "⛳"
type: "tech"
topics:
  - "bash"
published: true
published_at: "2024-09-05 19:00"
---

## あらまし
あるシェルスクリプトをDockerコンテナ内で実行すると、シェルスクリプト内で環境変数`$USER`が設定されておらずエラーとなる事象に遭いました。

結論から書くと、
- `docker run -it rockylinux:9 /bin/bash`でコンテナに接続する
- そのbashはログインシェルではなくインタラクティブシェルなので、`/etc/profile`は実行しない
- `/etc/profile`内で`$USER`変数が設定されるため、インタラクティブシェルでは`$USER`が使えない
- そのため、`$USER`の代わりに`whoami`を使用した

という感じでした。

ということで、備忘のためにbashでユーザ名を取得する方法をまとめておきます。

## `echo $USER`
環境変数$USERを使ってユーザー名を表示します。ただし、セッションによっては正しく設定されていないこともあるため注意が必要です。また、上書きすることも可能であるためその点にも注意が必要です。
```bash
$ echo $USER
shinya
$ sudo echo $USER
shinya
```

## `whoami`
現在のユーザー名をそのまま表示します。
```bash
$ whoami
shinya
$ sudo whoami
root
```

## `logname`
セッションにログインしたユーザー名を取得します。sudoの影響を受けずにログインユーザーを知りたい場合に便利です。
```bash
$ logname
shinya
$ sudo logname
shinya
```
`$USER`と同様に`/etc/profile`内で`$LOGNAME`変数が設定されますが、`$USER`と同様の注意点があるので`logname`コマンドを使うのがいいでしょう。

## `id -u -n`
現在のユーザーIDに対応するユーザー名を取得します。
```bash
$ id -u -n
shinya
$ sudo id -u -n
root
```

## `ps -o user= -p $$ | awk '{print $1}'`
Solarisなどの環境で、標準のコマンドが利用できない場合に使える方法です。
```bash
$ ps -o user= -p $$ | awk '{print $1}'
shinya
```

## まとめ
ユーザー名を取得する方法は、環境や用途、sudoの影響の有無に応じて適切に選択する必要があるので、皆様お気を付けください。

## 参考
https://stackoverflow.com/questions/19306771/how-do-i-get-the-current-users-username-in-bash