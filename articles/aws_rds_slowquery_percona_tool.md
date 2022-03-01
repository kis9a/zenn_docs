---
title: "AWS Aurora MySQL の slowquery logs を取得して percona-toolkit で集計する"
emoji: "🛠"
type: "tech"
topics: ["mysql", "shell", "terraform"]
published: true
---

## 初めに

最近データベースの CPU が 100%に張り付いた状態になり、負荷の特定と改善が課題になりました。クエリログを確認したところ、slowquery が多く出ていたので、これを集計して次回の SQL 改善や負荷対策に対策しようと思っています。色々調べたところ、AWS Aurora MySQL の slowquery log の有効かと percona-toolkit で集計する方法がヒットしたのでまとめて共有します。

## Slowquery log を有効にする

まず初めに、slowquery のログを有効かする必要があります。RDS のパラメーターと設定を slowquery のログを有効化します。

> スロークエリーログは、実行に long_query_time 秒を超える時間がかかり、少なくとも min_examined_row_limit 行を検査する必要がある SQL ステートメントで構成されます。 スロークエリーログは、実行に長い時間がかかっているため最適化の候補となるクエリーを見つけるために使用できます。long_query_time の最小値およびデフォルト値は、それぞれ 0 および 10 です。 値はマイクロ秒の精度まで指定できます。

https://dev.mysql.com/doc/refman/8.0/ja/slow-query-log.html

> Aurora MySQL ログイベントを有効にすると、Amazon CloudWatch Logs でイベントをモニタリングできます。新しいクラスターロググループは、cluster-name が DB クラスター名となり、log_type がログタイプとなる次のプレフィックスの Aurora DB クラスターに自動的に作成されます。 /aws/rds/cluster/cluster-name/log_type 例えば、エクスポート関数を設定して、mydbcluster という名前の DB クラスターのスロークエリログを作成すると、スロークエリデータは、/aws/rds/cluster/mydbcluster/slowquery ロググループのスロークエリログストリームに保存されます。 クラスターのすべてのインスタンスにおけるイベントは、異なるログストリーミングを使用して、ロググループにプッシュされます。

[Amazon CloudWatch Logs への Amazon Aurora MySQL ログの発行 - Amazon Aurora](https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.Integrating.CloudWatch.html)

### terraform の場合

以下フィールのフィールドを設定しましょう。

slow_query_log = 1 　は、slowquery log を有効にします。
long_query_time に関しては、適宜変更してください。SQL が N 秒以上の場合に slowquery log を作成します。また、そのほか詳細な parameter のオプションに関してもドキュメントを参考にしてください。

```hcl
# https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/db_parameter_group#parameter
# https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.Reference.html
resource "aws_db_parameter_group" "dev" {
  name   = "${var.service}-rds-pg"
  family = "mysql8.0"

  parameter {
    name  = "slow_query_log"
    value = "1"
  }

  parameter {
    name  = "long_query_time"
    value = "3"
  }
  # ...ry
}

# https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/rds_cluster
resource "aws_rds_cluster" "dev" {
  enabled_cloudwatch_logs_exports = ["slowquery"]
  # ...ry
}

# https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/db_instance
resource "aws_db_instance" "dev" {
  enabled_cloudwatch_logs_exports = ["slowquery"]
  # ...ry
}
```

### 手動で設定している場合

以下の記事がわかりやすいため参考にしてみてください。
https://dev.classmethod.jp/articles/amazon-aurora-export-cloudwatch-logs/

設定できているかは、mysql で実際に変数を参照すれば確認できます。

```
mysql> show variables like 'long%';
+-----------------+----------+
| Variable_name   | Value    |
+-----------------+----------+
| long_query_time | 3.000000 |
+-----------------+----------+
1 row in set (0.00 sec)
```

## slowquery logs の集計

### 必要なツールのインストール

brew の場合

```bash
brew install jq
brew install percona-toolkit
```

### jq とは ？

もっとも一般的で多機能なコマンドラインでの json parser です  
[GitHub - stedolan/jq: Command-line JSON processor](https://github.com/stedolan/jq)

### percona-toolkit とは ？

MySQL に関連する、複雑なシステムタスクを簡単に実行するためのコマンドラインツールのコレクションです
[GitHub - percona/percona-toolkit: Percona Toolkit](https://github.com/percona/percona-toolkit)

今回使用する pt-query-digest コマンドは、

> MySQL のクエリをログ、プロセスリスト、tcpdump の結果から分析します。
> usage: pt-query-digest [OPTIONS] [FILES] [DSN]

pt-query-digest に関してはこちらが参考になります。
[スローログの集計に便利な「pt-query-digest」を使ってみよう | Think IT（シンクイット）](https://thinkit.co.jp/article/9617)

## 集計スクリプト

### Slowquery log group を取得する

とりあえず、slowquery と名前のつくロググループの一覧を取得します。

```bash
#!/bin/bash
AWS_PROFILE="default" # change to your aws profile
aws logs describe-log-groups --profile "$AWS_PROFILE" \
--output json | jq -r '.logGroups | .[] | .logGroupName' | grep slowquery
```

\# set variables の項目にある変数を適宜編集しましょう。  
上のスクリプトで取得した LOG_GROUP を設定します。  
改善前、改善後、で違いが判断できるように start time, end time をメモしておきましょう。

```bash
#!/bin/bash

## set variables
LOG_GROUP="/aws/rds/cluster/service-cluster/slowquery"
AWS_PROFILE="default" # change to your aws profile
START_TIME="2021-10-01 12:00:00"
END_TIME="2022-03-01 12:00:00"

# parse to microseconds unixtime
timestamp_start="$(($(TZ=JST-9 date -jf "%Y-%m-%d %H:%M:%S" "$START_TIME" +%s) * 1000))"
timestamp_end="$(($(TZ=JST-9 date -jf "%Y-%m-%d %H:%M:%S" "$END_TIME" +%s) * 1000))"

# colorable
readonly cf="\\033[0m"
readonly red="\\033[0;31m"

# secion header
display_section_header() {
  echo -e "${red}==================== $1 ====================${cf}"
}

# get log streams
cloudwatch-log-streams() {
  aws logs describe-log-streams --log-group-name "$1" --profile "$AWS_PROFILE" --output json | jq .
}

# get log events
cloudwatch_log_events() {
  aws logs get-log-events --profile "$AWS_PROFILE" \
    --log-group-name "$LOG_GROUP" \
    --log-stream-name "$1" \
    --start-time "$timestamp_start" \
    --end-time "$timestamp_end" \
    --query 'events[].message' | jq -r '.[]'
}

while read -r stream; do
  # get stream events
  display_section_header "$stream"
  events="$(cloudwatch_log_events "$stream")"

  # list SQL Query
  # Query を実行回数順に一覧表示します
  # grep -v で特定のパターン対象外にします
  display_section_header "Queries"
  echo "$events" | grep -v ^SET | grep -v ^# | grep -v ^use | sort | uniq -c | sort -r

  # pt-query-digest
  display_section_header "Pt-query-digest"
  echo "$events" | pt-query-digest --limit 10000

done <<<"$(cloudwatch_log_streams "$LOG_GROUP" | jq -r '.logStreams | .[] .logStreamName')"
```

## sort | uniq -c | sort -r が遅い場合

以下の記事を参考にして、awk で書き直すといいでしょう。  
今回自分が集計したものでは、100000 件ほどだったので、  
数十秒で終わりそれほど気になりませんでしたが、もっとデータ量が多い場合は以下を使うと better です。

https://qiita.com/kazinoue/items/e7b98512186bace00097

```diff
- sort | uniq -c | sort -r
+ awk '{ v[$0]++ } END { for ( k in v ) print v[k] "\t" k }' | tail -r
```

## 集計結果の見方

**stream 名**
一般的には、instance 名になっていると思います。

**queries**
実行頻度が、高い順位に表示します。パフォーマンス改善の優先順位の目安にします。

**pt-query-digest**
ログストリーム(db instance) 毎に Overall: Profile: Query: の順で見ていきます。  
Query: は、結果が悪い(slow)順に並べられているので基本的に上から見ていきます。  
その他の項目に関しては、実際の下のような例を見てもらうといいでしょう。

```
==================== instance-1 ====================
==================== Queries ====================
298 SELECT count(1) FROM `dev`;
159 SELECT count(1) FROM `items`;
64 SELECT count(1) FROM `dev_details`;
==================== Pt-query-digest ====================
# Overall: 366 total, 3 unique, 0.21 QPS, 1.73x concurrency ______________
# Time range: 2022-03-01T01:00:10 to 2022-03-01T01:28:37
# Attribute          total     min     max     avg     95%  stddev  median
# ============     ======= ======= ======= ======= ======= ======= =======
# Exec time          2960s      3s     18s      8s     15s      4s      8s
# Lock time           37ms    21us     4ms    99us    98us   330us    52us
# Rows sent          2.26k       1      21    6.32   20.43    8.01    0.99
# Rows examine     107.18M       0   1.17M 299.87k   1.14M 480.31k       0
# Query size        18.44k      29      88   51.60   84.10   24.39   34.95

# Profile
# Rank Query ID                            Response time   Calls R/Call V/
# ==== =================================== =============== ===== ====== ==
#    1 0x8XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX  1328.0743 44.9%   159 8.3527  1.43 SELECT dev
#    2 0x8XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX   820.9552 27.7%   124 6.6206  2.07 SELECT dev_details
#    3 0x9XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX   811.0785 27.4%    83 9.7720  1.07 SELECT items

# Query 1: 0.09 QPS, 0.78x concurrency, ID 0x8XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX at byte 51851
# Attribute    pct   total     min     max     avg     95%  stddev  median
# ============ === ======= ======= ======= ======= ======= ======= =======
# Count         43     159
# Exec time     44   1328s      3s     18s      8s     14s      3s      8s
# Lock time     48    18ms    21us     3ms   110us    84us   401us    42us
# Rows sent      6     159       1       1       1       1       0       1
# Rows examine   0       0       0       0       0       0       0       0
# Query size    29   5.43k      35      35      35      35       0      35
# String:
# Databases    dev
# Hosts        xxxxxxxxxxxxx
# Users        writer
# Query_time distribution
#   1us
#  10us
# 100us
#   1ms
#  10ms
# 100ms
#    1s  ################################################################
#  10s+  ##########################
# Tables
#    SHOW TABLE STATUS FROM `dev` LIKE 'devs'\G
#    SHOW CREATE TABLE `dev`.`devs`\G
# EXPLAIN /*!1000 PARTITIONS*/
SELECT count(1) FROM `devs`\G
```

## 終わりに

SQL 改善の方でも知見を得たら随時共有していこうと思います。データベースに関してもまだ初学者なので一歩ずつ勉強していきたいと思っています。そのほかで、もっといい方法とあればぜひコメントで教えていただけると幸いです。
