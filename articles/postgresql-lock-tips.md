---
title: "PostgreSQLのロックでハマりがちな挙動5選"
emoji: "🐘"
type: "tech"
topics:
  - "postgresql"
  - "database"
  - "lock"
published: false
---

# PostgreSQLのロックでハマりがちな挙動5選

## はじめに

PostgreSQLは、同時実行制御にMVCC（Multi-Version Concurrency Control）を使用しており、読み込みは書き込みをブロックしませんし、逆に書き込みも読み込みをブロックしません。

そして、PostgreSQLのロックには、8種類のテーブルレベルロックと4種類の行レベルロックがあり、ドキュメントに記載されているロック競合の表を見れば、どのロックがどのロックと競合するかがわかります。

しかし、実際にPostgreSQLを運用していると思わぬところでロックが競合し、処理時間が想定よりもかかったり、最悪の場合サービス停止などの事態に陥ることがあります。

この記事では、そうした直感的ではないロックの挙動を5つ取り上げて解説していきます！

## 環境
- バージョン：PostgreSQL 18
- トランザクション分離レベル：READ COMMITTED（デフォルト）

## 1. `ACCESS EXCLUSIVE`がキューに入ると後続クエリも連鎖的に待たされる

1つ目は、すぐに終わるはずの`ALTER TABLE`が原因でサービス全体が止まる可能性がある、というものです。

例えば、あるセッションがテーブル`t`に対して長時間かかる`SELECT`を実行中に、別のセッションで以下の`ALTER TABLE`を実行したとします。

```sql:セッション1
SELECT pg_sleep(600) FROM t LIMIT 1; -- 長時間かかるSELECT
```

```sql:セッション2
ALTER TABLE t ADD COLUMN name text;
```

すると、セッション1がテーブル`t`に対して`ACCESS SHARE`ロックを獲得しているため、セッション2の`ALTER TABLE`が要求する`ACCESS EXCLUSIVE`ロックは待機させられます。ここまでは想定通りの挙動だと思います。

しかし、PostgreSQLのロック待ちはFIFOのキューのようなものなので、`ACCESS EXCLUSIVE`ロックを待機している間に実行されたテーブル`t`への`SELECT`は、現在実行中のセッション1の`SELECT`とはロック競合しないにもかかわらず、待たされることになります。

![ロック待ち行列で後続のSELECTも待たされる様子の図](/images/postgresql-lock-case1-queue.drawio.png =700x)

:::message
長時間の`SELECT`以外にも、`BEGIN`で開いたトランザクション内で`SELECT`を実行したまま`COMMIT`/`ROLLBACK`せずに放置しているセッション（いわゆる *idle in transaction*）でも同じことが起きます。`ACCESS SHARE`ロックはトランザクションが終了するまで保持されるため、`SELECT`自体は一瞬で終わっていても、トランザクションを閉じ忘れているだけで`ALTER TABLE`をブロックし続けます。アプリケーションのコミット漏れや、結果取得後に放置されたコネクションが原因になりやすいので注意してください。
:::

結果として、

1. 長時間`SELECT`が実行されている
2. `ALTER TABLE`（など`ACCESS EXCLUSIVE`ロックを獲得するSQL文）がロック競合で待機させられる
3. 後続のすべての`SELECT`が2のロック待ちの後ろに並ばされる

という連鎖的なブロックが発生し、想定外に`SELECT`が待たされることでサービス停止の可能性がある、という話でした。

### 対策

- `ACCESS EXCLUSIVE`ロックを獲得するSQL文を実行するトランザクション内では`lock_timeout`を設定し、ロック待ちが長時間化しないようにする
- 長時間クエリが実行されていないかを事前に`pg_stat_activity`ビューで確認する

## 2. 外部キー制約が引き起こす「見えない」デッドロック

2つ目は、外部キー制約が暗黙的に獲得するロックによるものです。

`INSERT`文を実行すると、PostgreSQLは外部キー制約で参照しているすべての参照先テーブルの対象行に対して`FOR KEY SHARE`ロックを取得します。これは「参照先テーブルの対象行が削除・更新されると参照元の整合性が崩れる」のを防ぐための仕組みです。

例えば、テーブル`t`に`INSERT`すると、参照先のテーブル`s`の対象行に対して`FOR KEY SHARE`ロックが自動で取得されます。`FOR KEY SHARE`と`FOR UPDATE`は競合するため、複数のセッションがそれぞれ「`s`の異なる行を`FOR UPDATE`でロックしてから、互いの行を外部キー参照する`INSERT`を実行する」という操作を逆順で行うと、循環待ちが発生してデッドロックになります。

```sql:セッション1
BEGIN;
SELECT * FROM s WHERE id=1 FOR UPDATE; -- s.id=1の行をFOR UPDATEでロック
```

```sql:セッション2
BEGIN;
SELECT * FROM s WHERE id=2 FOR UPDATE; -- s.id=2の行をFOR UPDATEでロック
```

```sql:セッション1
INSERT INTO t (s_id) VALUES (2); -- s.id=2の行のFOR KEY SHAREが欲しい（セッション2待ち）
```

```sql:セッション2
INSERT INTO t (s_id) VALUES (1); -- s.id=1の行のFOR KEY SHAREが欲しい（デッドロック検出）
```

何が直感的ではないかというと、

- `INSERT`文しか書いていないアプリケーションコードが、外部キー参照先のテーブルの行を暗黙的にロックしている
- アプリケーションのSQL文を眺めても、ロック取得が表面化しない

という点です。

### 対策

- 外部キー参照先テーブルの行を`FOR UPDATE`でロックしてから、別の行を外部キーとして`INSERT`するパターンが複数セッションで逆順になるとデッドロックになるため、そのような設計を避ける
- どうしても必要なら、ロック取得順序をアプリケーション全体で揃える（例：`s`の行は常に`id`昇順でロックするなど、一方向のロック順序を徹底する）
- デッドロックは必ずエラーとして検出されるため、リトライ前提で実装する

## 3. 一意制約の重複チェックが招く`INSERT`同士のデッドロック

3つ目は、`INSERT`のみの2つのトランザクションが、互いに相手の挿入済みの値を逆順で挿入しようとするだけでデッドロックする話です。

以下のようなSQLを実行します。

```sql:セッション1
BEGIN;
INSERT INTO t (id) VALUES (1);
```

```sql:セッション2
BEGIN;
INSERT INTO t (id) VALUES (2);
```

```sql:セッション1
INSERT INTO t (id) VALUES (2); -- セッション2のコミットを待つ
```

```sql:セッション2
INSERT INTO t (id) VALUES (1); -- デッドロック検出
```

なぜこのような挙動になるかというと、`PRIMARY KEY`や`UNIQUE`制約の重複チェックは、**他のトランザクションが同じ値を挿入中である場合、そのトランザクションの完了を待つ**動作になっているからです。これは「相手がコミットすれば一意制約違反、ロールバックすれば挿入成功」を確定させるための仕組みです。

つまり、上の例では、

- セッション1が`id=2`を挿入しようとする → セッション2が`id=2`を挿入中なので待つ
- セッション2が`id=1`を挿入しようとする → セッション1が`id=1`を挿入中なので待つ
- お互いに相手のコミットを待ち合わせ → デッドロック検出

という流れです。

### 対策

- 同じ値が複数セッションから同時挿入されうる設計を避ける（`SERIAL`/`IDENTITY`などシーケンスを使えばそもそも重複しない）
- 2つ目の対策で触れたとおり、デッドロックはエラーとして検出されるためリトライ前提で実装する

## 4. トランザクションID周回防止の自動バキュームだけは競合しても止まらない

4つ目は、自動バキュームの例外的な挙動の話です。

自動バキュームが取得する`SHARE UPDATE EXCLUSIVE`ロックが別のSQLと競合した場合、通常は自動バキュームのほうがキャンセルされ、他のSQLを長時間ブロックしないようになっています。そのため「自動バキュームは安全に動かしっぱなしでいい」という認識が一般的です。

しかし、トランザクションID周回防止のために走っている自動バキューム（`pg_stat_activity`の`query`列の末尾が`(to prevent wraparound)`のもの）だけは例外で、**競合があっても自動でキャンセルされません**。

実際にどのようなことが起きるかというと、

1. テーブルの`relfrozenxid`の年齢が`autovacuum_freeze_max_age`を超える
2. トランザクションID周回防止の自動バキュームが起動する（`SHARE UPDATE EXCLUSIVE`を保持）
3. 別プロセスがパーティション作成のために`ALTER TABLE`を発行する
4. `ALTER TABLE`は自動バキュームを待つ ← **通常はキャンセルされるはずなのにされない**
5. すべての`SELECT`がキューに並ぶ ← 1つ目で説明した連鎖ブロックが発生

という連鎖になります。1つ目のネタと組み合わさると一気に致命的になるのがポイントですね。

### 対策

- `pg_stat_activity`で`autovacuum: VACUUM ... (to prevent wraparound)`を見たら、キャンセルされない自動バキュームが動いていると認識し、完了を待つ
- 発生後の対処より予防が重要。`pg_class`の`age(relfrozenxid)`を定期監視し、`autovacuum_freeze_max_age`（デフォルト2億トランザクション）に近づいているテーブルを把握する

  ```sql
  SELECT relname, age(relfrozenxid)
  FROM pg_class
  WHERE relkind = 'r'
  ORDER BY age(relfrozenxid) DESC;
  ```

- `ALTER TABLE`などのDDLを実行する予定があるなら、対象テーブルの`age(relfrozenxid)`を事前に確認し、しきい値に近ければDDLの前に手動で`VACUUM FREEZE`しておくなどの対策を実施する

## 5. バキュームの隠れた`ACCESS EXCLUSIVE`フェーズ

最後も、バキュームの例外的な挙動の話です。

通常、バキュームは`SHARE UPDATE EXCLUSIVE`で動作しますが、**最後のtruncateフェーズ**（テーブル末尾の空ページを切り詰めてOSにディスクを返す処理）だけは`ACCESS EXCLUSIVE`を取得します。

これまでの例と同様に、`ACCESS EXCLUSIVE`の取得を待つ間に長時間`SELECT`が実行されていると、1つ目で説明した連鎖ブロックと同じパターンで後続クエリが全て待たされます。

そして、ストリーミングレプリケーションのスタンバイでは構造が異なります。プライマリで`ACCESS EXCLUSIVE`が取得・解放されると、その操作がWALとしてスタンバイに届きます。スタンバイのWALリプレイプロセスがその操作を適用しようとしますが、スタンバイで実行中の長時間`SELECT`が競合するため、**WALリプレイが止まります**。WALリプレイが止まると受信済みWALの適用遅延が広がっていき、その遅延が`max_standby_streaming_delay`を超えると、競合している`SELECT`が**強制キャンセル**され、リプレイが再開されます。

プライマリでは`SELECT`は自然に終わるまで待てますが、スタンバイでは外部要因（WALリプレイ）によって`SELECT`が強制キャンセルされる点が大きく異なります。

### 対策

- PostgreSQL 12以降で`vacuum_truncate = false`でバキュームのtruncateフェーズをテーブル単位で無効化できる
- PostgreSQL 18以降でサーバ全体に対する設定`vacuum_truncate`パラメータも追加された
- スタンバイで長時間クエリを走らせる場合は、`max_standby_streaming_delay`（デフォルト30秒）を大きく設定し、WALリプレイが待機できる時間を伸ばす（`-1`で無制限）

## まとめ

どのケースも、PostgreSQLのロック設計が「正しさを保つために必要な仕組み」を真面目に実装した結果として現れる挙動です。仕様としては自然でも、運用視点では落とし穴になります。

特に1つ目の連鎖ブロックは、他のケースと組み合わさることで急激に致命的な障害に発展するので、`lock_timeout`の設定と`pg_stat_activity`の継続監視は最低限の防御線として入れておきたいところです。

この記事の内容に加えて、PostgreSQLのロックに関する諸々をポッドキャストで話しています。ご興味のある方は聞いてみてください！

https://open.spotify.com/show/6mv9RwXQRivdwST6oJ0Wmf?si=48d65887882c4a2c

## 参考

https://www.postgresql.org/docs/current/explicit-locking.html

https://www.postgresql.org/message-id/d7df81620708101038k772a2cderb52bb09f5440bd1b@mail.gmail.com

https://leosjoberg.com/blog/lock-propagation-postgres/

https://xata.io/blog/migrations-and-exclusive-locks
