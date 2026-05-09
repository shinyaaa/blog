---
title: "大量DBオブジェクト下におけるrecovery_init_sync_methodの有効性を確認する"
emoji: "🐘"
type: "tech"
topics:
  - "postgresql"
published: true
published_at: "2024-10-12 13:30"
---

## はじめに
PostgreSQL 14で導入された[`recovery_init_sync_method`](https://www.postgresql.jp/document/16/html/runtime-config-error-handling.html#GUC-RECOVERY-INIT-SYNC-METHOD)について、このパラメータが導入された経緯とその有効性を確認します。


## 経緯
以下のメーリングリストで議論されていますが、次のような経緯で`recovery_init_sync_method`が導入されました。
- PostgreSQLはクラッシュリカバリ開始前に、データ損失を避けるためにデータディレクトリ全体を再帰的に同期（`fsync`）する
- データディレクトリ内のファイル数（≒DBオブジェクト数）が多い場合、同期に時間がかかる
- `syncfs`システムコールを使ってファイルシステム全体を同期すれば、パフォーマンスを改善できるかもしれない
- パフォーマンスが悪化する場合や`syncfs`が使えないプラットフォームもあるので、`recovery_init_sync_method`パラメータで同期方法を切り替えられるようにしよう

https://www.postgresql.org/message-id/flat/11bc2bb7-ecb5-3ad0-b39f-df632734cd81%40discourse.org


## 検証環境
以下の環境で検証を実施しました。

```bash
$ psql -c "SELECT version()"
                                                  version                                                  
-----------------------------------------------------------------------------------------------------------
 PostgreSQL 14.13 on x86_64-pc-linux-gnu, compiled by gcc (GCC) 11.4.1 20231218 (Red Hat 11.4.1-3), 64-bit
(1 row)
$ cat /etc/redhat-release 
Rocky Linux release 9.4 (Blue Onyx)
$ uname -r
5.14.0-427.22.1.el9_4.x86_64
```

なお、`syncfs`を使えない環境（Windowsなど）ではもちろん使用できません。

また、Linux 5.8以前では、ディスクへの書き込み中に発生したI/OエラーがPostgreSQLに報告されないことがあり、関連エラーメッセージはカーネルログにのみ現れるかもしれません。

https://kernelnewbies.org/Linux_5.8#line-88

## 検証方法
- PostgreSQLのソースコードに時間計測用のコードを追加して、`SyncDataDirectory()`にかかる時間をログに出力
```diff
$ git diff
diff --git a/src/backend/access/transam/xlog.c b/src/backend/access/transam/xlog.c
index 7654ad76de..fd356c1edb 100644
--- a/src/backend/access/transam/xlog.c
+++ b/src/backend/access/transam/xlog.c
@@ -6624,8 +6624,12 @@ StartupXLOG(void)
        if (ControlFile->state != DB_SHUTDOWNED &&
                ControlFile->state != DB_SHUTDOWNED_IN_RECOVERY)
        {
+               time_t start_time, end_time;
                RemoveTempXlogFiles();
+               start_time = time(NULL);
                SyncDataDirectory();
+               end_time = time(NULL);
+               elog(LOG, "time taken for SyncDataDirectory(): %ld", end_time - start_time);
        }
 
        /*
```
- テーブルを1万個～1,000万個作成したうえで、`pg_ctl stop -m immedite`でPostgreSQLを即時シャットダウン
    - なおテーブル作成方法は以下
```psql
=# SELECT format('CREATE TABLE t%s (i INT)', generate_series(1, 10000)) \gexec
```
- `pg_ctl start`でPostgreSQLを起動


## 検証結果
検証結果は以下のようになりました。

| テーブル数 | `fsync` [s] | `syncfs` [s] | 
| ---------- | ------- | -------- | 
| 10,000     | 2       | 0        | 
| 100,000    | 17      | 0        | 
| 1,000,000  | 167     | 0        | 
| 10,000,000 | 1,708   | 0        | 

※Cのtime()関数を使ったので1秒以下は計測できていません。

予想通りの結果ではありますが、テーブル数が100万を超えると許容できないくらいの時間がかかります。実際には、クラッシュリカバリ時にはWALを適用する時間もかかるので、起動時間全体としてはもっと長くなると思われます。

今回は、同期方法を`syncfs`にすると性能が改善することを確認できました。しかし、例えば、以下のような場合は性能が悪くなることも考えられます。（検証はしていないので想像です）
- DBオブジェクト数が少ない場合は、ファイルシステム全体を同期するほうが時間がかかる可能性がある
- 他のソフトウェアとファイルシステムを共有している場合は、PostgreSQL以外のソフトウェアによる書き込みが同期されることで、時間がかかる可能性がある

## まとめ
PostgreSQL 14で導入された`recovery_init_sync_method`の有効性を検証しました。

PostgreSQLで大量DBオブジェクトを扱う際にクラッシュリカバリ開始前のデータディレクトリ同期が高速化することを確認できました。

逆に言うと、PostgreSQL 13までを使っている場合でDBオブジェクト100万オーダを超えてくると、なかなか厳しいのではないかと思いました。