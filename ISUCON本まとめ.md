# ISUCON 本まとめ

## Chapter3 基礎的な負荷試験

### nginx のアクセスログ集計

- `webapp/etc/nginx/conf.d/default.conf`に次のように記述することでアクセスログを JSON 形式で集計できる

```conf
log_format json escape=json '{"time": "$time_iso8601",'
                             '"host": "$remote_addr",'
                             '"port": "$remote_port",'
                             '"method": "$request_method",'
                             '"uri": "$request_uri",'
                             '"status": "$status",'
                             '"body_bytes": "$body_bytes_sent",'
                             '"referer": "$http_referer",'
                             '"ua": "$http_user_agant",'
                             '"request_time": "$request_time",'
                             '"response_time": "$upstream_response_time"}';

server {
    listen 80;
    client_max_body_size 10m;
    root /public/;

    location / {
        proxy_set_header Host $host;
        proxy_pass http://app:8080;
    }
}
```

ここで`$request_time`は nginx がクライアントからリクエストを受け取ってからレスポンスを返すまでの時間
`$upstream_response_time`は Web アプリケーションが処理を終えて nginx にレスポンスを返すまでの時間
この２つは基本的にほぼ一致するが、`$request_time`と`$upstream_response_time`の差分が大きい場合はネットワークのトラブルやクライアント側の回線にボトルネックがあると予測される。

### アクセスログを解析するツール alp

- `brew install alp`でインストールできる
- `alp json -file access.log`のようにつかう
- alp はデフォルトでは URI のクエリ文字列は無視するが、無視したくない場合は`-q, --query-string`オプションを使用する
- パスの一部にパラメータが含まれている場合は正規表現を使って集計ができる。たとえば`diary/entry/1234`と`diary/entry/5678`を同一視したい場合は`-m "diary/entry/[0-9]+"`のようにオプションを指定する。

### 負荷試験を行うコマンド ab コマンド

- macOS には標準でインストールされている
- ab コマンドはコマンドライン引数で指定した URL へ、指定した多重度でリクエストを送信する

```
ab -c 1 -n 10 http://localhost/
```

- 指定した回数や時間で HTTP リクエストを送信したあと、標準出力に結果が出力される

### アクセスログのローテーション

- 負荷試験ごとにログファイルを分けないと、適切に集計結果を分析することができない
- ローテーション（ログファイルの改名）を行うことが重要
- コンテナ/インスタンス内のファイルを改名（例:access.log -> access_old.log）すればよいが、それだけだと nginx は改名後のファイルにログを書き続けてしまう。
- 実際にログが出力されるファイルを切り替えるためには**nginx を再起動**するか、もしくは**nginx の master プロセスのシグナルを送信する**必要がある
- 本番環境では後者のほうが安全だが、isucon のようなパフォーマンスチューニングの大会では前者がお手軽
- nginx を再起動するには`systemctl restart nginx`を実行すれば良い

### 負荷試験とパフォーマンスチューニングの流れ

1. まずサーバーに負荷をかける
   ```
   ab -c 1 -t 30
   ```
2. ホスト上で top コマンドを実行して CPU の使用状況などをモニタリングする
   - 注目するべきところは以下
     - ホスト全体で使える CPU のうち、どの程度が使われているか
     - CPU が大きく使われているプロセスはどのプロセスか
3. 上の 2 でたとえば MySQL に大きく CPU が使われていることがわかった場合、MySQL のボトルネックを解消する

- スロークエリログを出力して、それを解析する。具体的にはまずスロークエリログを出力するために`webapp/etc/mysql/conf.d/my.cnf`に次を記述する

```
[mysqld]
slow_query_log = 1
slow_query_log_file = /var/log/mysql/mysql-slow.log
long_query_time = 0
```

- この変更を適用するために MySQL の再起動をする。EC2 で動作している場合は root ユーザーで`systemctl restart mysql`を実行する。docker で動作している場合は`docker compose down`をしてから`docker comopse up`を行う。

- 出力したログを集計するために`mysqldumpslow`コマンドを用いる

```
mysqldumpslow /var/log/mysql/mysql-slow.log
```

- `mysqldumpslow`コマンドはログ中の実行時間の合計が長いクエリから順に表示する。

- 遅いクエリが見つかったら実際にスロークエリログに出力されたログを見てみる。
- `Rows_examind`の値が大きい場合、これは走査するレコードの数が大きいことを示す。
- EXPLAIN 文でクエリの実行計画を確認する
- Index が使用されていない場合、Index を貼ることを考える。
- 一般に、プライマリーキーしかインデックスがないテーブルからプライマリーキー以外の条件で WHERE 句で指定した条件に一致する行を見つけるためには、テーブルのすべての行を読み取る必要がある

4. チューニングをしたあとは、そのチューニングが有効であったかどうかを確認する。

- ab コマンドを再実行し、top コマンドで CPU 使用率を確認するなど

### ベンチマーカーの並列度

- ab コマンドは並列リクエストで負荷をかけることもできる

```
ab -c 2 -t 30 http:localhost/
```

- 並列リクエストを送ったときに、並列度に比例してレスポンスタイムが悪化した + サーバーが処理できる秒間リクエストが並列をを変えても変化しなかったという場合は、「直列リクエストの時点でサーバーの処理能力が飽和している」と考えられる

- 時系列順に CPU 使用率を観察するには`dstat --cpu`コマンドが使える
- アプリケーションプロセスは`systemctl status isu-ruby`で確認できる
- このとき、サーバーに複数の CPU が搭載されていてもソフトウェアのアーキテクチャや設定によってその CPU を存分に使えていないことがある。今回の場合は unicorn に設定しているワーカー数が 1 と設定されていた。ワーカープロセスを変更し再起動をする。（ワーカープロセスは一般に CPU の数倍程度に設定するのが好ましい。）
