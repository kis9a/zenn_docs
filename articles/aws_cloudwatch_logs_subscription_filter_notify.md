---
title: "CloudWatch logs Subsrciption filter からSlackの特定チャンネルにログを通知してみた。"
emoji: "⏰"
type: "tech"
topics: ["go", "lambda", "cloudwatch", "terraform"]
published: true
---

## 初めに

CloudWatch logs には、subscription filter という機能があります。
これは、CloudWatch logs から 文字列やパターンでフィルターしたログを  
Kinesis stream や Lambda function にログとイベントを発火することができます。
詳細は、以下の公式ドキュメントをみてみましょう。

https://docs.aws.amazon.com/ja_jp/AmazonCloudWatch/latest/logs/SubscriptionFilters.html

今回は、CloudWatch logs subscription filter や CloudWatch log group 毎に Slack の特定チャンネルにログを通知できる Lambda 関数 書いてみたので、紹介します。検索する同様の記事がいくつか見つかりますが、Go 言語 (Go が書ける人がチームに多く)、 Subsrciption filter 名 から複数の特定チャンネルに通知できるように、汎用的にすることが目的や、terraform でのリソースデプロイを含めて記述します。

![/images/cloudwatch-logs-subscription-filter.png](/images/cloudwatch-logs-subscription-filter.png)

## Lambda function

リポジトリ：[terraform/functions/cloudwatch*error_logs*· kis9a/terraform · GitHub](https://github.com/kis9a/terraform/tree/master/functions/cloudwatch_error_logs_notify)

### ファイル構成

```
├── go.mod
├── go.sum
├── handler
│   ├── handler.go
│   └── handler_test.go
├── main
├── main.go
└── testdata
    └── cloudwatch_logs_message_events.json
```

#### main.go

```go
package main

import (
  "log"
  "main/handler"

  "github.com/aws/aws-lambda-go/lambda"
)

func main() {
  lambda.Start(handler.New().Run)
}
```

lambda.Start() とはなんでしょうか？ 定義ジャンプしてみましょう丁寧に説明されています。

```go
// Start takes a handler and talks to an internal Lambda endpoint to pass requests to the handler. If the
// handler does not match one of the supported types an appropriate error message will be returned to the caller.
// Start blocks, and does not return after being called.
//
// Rules:
//
// _ handler must be a function
// _ handler may take between 0 and two arguments.
// _ if there are two arguments, the first argument must satisfy the "context.Context" interface.
// _ handler may return between 0 and two arguments.
// _ if there are two return values, the second argument must be an error.
// _ if there is one return value it must be an error.
//
// Valid function signatures:
//
// func ()
// func () error
// func (TIn) error
// func () (TOut, error)
// func (TIn) (TOut, error)
// func (context.Context) error
// func (context.Context, TIn) error
// func (context.Context) (TOut, error)
// func (context.Context, TIn) (TOut, error)
//
// Where "TIn" and "TOut" are types compatible with the "encoding/json" standard library.
// See https://golang.org/pkg/encoding/json/#Unmarshal for how deserialization behaves
func Start(handler interface{}) {
  StartWithContext(context.Background(), handler)
}
```

first context は、 first argument must satisfy the "context.Context" (Go の sdk なので型情報が有っていいですね)、second arguments は、呼び出しもと毎に違います。コンソールの Lambda Test 部分をみたりや検索してして、 events context の例を取得すること、sdk から正しく型を参照することがドキュメントを読む最初の準備です。

```go
func (h *Handler) Run(ctx context.Context, e events.CloudwatchLogsEvent) {
  _, err := parseContext(ctx)
...

func parseContext(ctx context.Context) (*lambdacontext.LambdaContext, error) {
  lc, ok := lambdacontext.FromContext(ctx)
...
```

#### cloudwatch_logs_message_events.json

handler.Run() の第二引数に指定するイベントの例です。(e events.CloudwatchLogsEvent)
<https://docs.aws.amazon.com/ja_jp/lambda/latest/dg/services-cloudwatchlogs.html>

```json
{
  "messageType": "DATA_MESSAGE",
  "owner": "123456789012",
  "logGroup": "/aws/lambda/echo-nodejs",
  "logStream": "2019/03/13/[$LATEST]94fa867e5374431291a7fc14e2f56ae7",
  "subscriptionFilters": ["LambdaStream_cloudwatchlogs-node"],
  "logEvents": [
    {
      "id": "34622316099697884706540976068822859012661220141643892546",
      "timestamp": 1552518348220,
      "message": "REPORT RequestId: 6234bffe-149a-b642-81ff-2e8e376d8aff\tDuration: 46.84 ms\tBilled Duration: 47 ms \tMemory Size: 192 MB\tMax Memory Used: 72 MB\t\n"
    }
  ]
}
```

#### handler_test.go

実際に handler.go テストを書きながら開発していきましょう。

```go
package handler

import (
  "encoding/json"
  "fmt"
  "io/ioutil"
  "os"
  "testing"
  "text/tabwriter"

  "github.com/aws/aws-lambda-go/events"
)

func TestNotify(t *testing.T) {
  var data events.CloudwatchLogsData
  fs, err := os.Open("../testdata/cloudwatch_logs_message_events.json")
  if err != nil {
    t.Fatal(err)
  }
  bytefs, err := ioutil.ReadAll(fs)
  if err != nil {
    t.Fatal(err)
  }
  defer fs.Close()
  if err = json.Unmarshal(bytefs, &data); err != nil {
    t.Fatal(err)
  }
  if err = notifyCloudwatchLogsEvent(data); err != nil {
    t.Fatal(err)
  }
}
```

#### handler.go

```go
package handler

import (
  "context"
  "encoding/json"
  "fmt"
  "log"
  "net/http"
  "net/url"
  "strconv"
  "strings"
  "time"

  "github.com/aws/aws-lambda-go/events"
  "github.com/aws/aws-lambda-go/lambdacontext"
  "github.com/gobwas/glob"
)

var webhooks = []Webhook{
  {
    Pattern: "*LambdaStream_cloudwatchlogs-node*",
    URL:     "https://hooks.slack.com/services/xxxxxxxxxxx/xxxxxxxxxxx/xxxxxxxxxxxxxxxxxxxxxxxx",
  },
}

type Payload struct {
  Username   string         `json:"username"`
  Icon_emoji string         `json:"icon_emoji"`
  Blocks     []PayloadBlock `json:"blocks"`
}

type PayloadBlock struct {
  Type string      `json:"type"`
  Text PayloadText `json:"text"`
}

type PayloadText struct {
  Type string `json:"type"`
  Text string `json:"text"`
}

type Webhook struct {
  Pattern string `json:"pattern"`
  URL     string `json:"url"`
}

type Handler struct{}

func New() *Handler {
  return &Handler{}
}

func (h *Handler) Run(ctx context.Context, e events.CloudwatchLogsEvent) {
  _, err := parseContext(ctx)
  if err != nil {
    log.Fatal(err)
  }

  data, err := e.AWSLogs.Parse()
  if err != nil {
    log.Fatal(err)
  }
  if err = notifyCloudwatchLogsEvent(data); err != nil {
    log.Fatal(err)
  }
}

func parseContext(ctx context.Context) (*lambdacontext.LambdaContext, error) {
  lc, ok := lambdacontext.FromContext(ctx)
  if !ok {
    return nil, fmt.Errorf("could not parse Lambda Context")
  }
  return lc, nil
}

func parseTimeStamp(t int64) string {
  timestampString := fmt.Sprintf("%d", t)
  utcTimestamp, err := strconv.Atoi(timestampString[:len(timestampString)-3])
  if err != nil {
    return timestampString
  }
  return fmt.Sprintf("%s", time.Unix(int64(utcTimestamp), 0).Format("2006-01-02 15:04:05"))
}

func notifyCloudwatchLogsEvent(l events.CloudwatchLogsData) error {
  lp, err := getCloudwatchLogsEventPayload(l)
  if err != nil {
    return err
  }
  pjson, err := json.Marshal(lp)
  if err != nil {
    return err
  }
  webhookURL := getWebhook(l.SubscriptionFilters[0]).URL
  if webhookURL == "" {
    return fmt.Errorf("LogFilter Name does not match a URL")
  }
  if err = notify(string(pjson), webhookURL); err != nil {
    return err
  }
  return nil
}

func getWebhook(m string) Webhook {
  for _, w := range webhooks {
    g := glob.MustCompile(w.Pattern, '/')
    is := g.Match(m)
    if is {
      return w
    }
  }
  return Webhook{}
}

func getCloudwatchLogsEventPayload(l events.CloudwatchLogsData) (Payload, error) {
  var p Payload
  p.Username = l.LogGroup
  p.Icon_emoji = "alarm_clock"

  for _, v := range l.LogEvents {
    p.Blocks = append(p.Blocks, PayloadBlock{
      Type: "section",
      Text: PayloadText{
        Type: "mrkdwn",
        Text: strings.Join([]string{"*", l.SubscriptionFilters[0], "*"}, ""),
      },
    },
    )
    p.Blocks = append(p.Blocks, PayloadBlock{
      Type: "section",
      Text: PayloadText{
        Type: "plain_text",
        Text: strings.Join([]string{
          "TIMESTAMP: ", fmt.Sprintf("%d", v.Timestamp), " (", parseTimeStamp(v.Timestamp), ")\n",
          "ID: ", v.ID, "\n",
          "MESSAGE: ", v.Message, "\n",
        }, ""),
      },
    },
    )
  }
  return p, nil
}

func notify(str string, webhookURL string) error {
  resp, err := http.PostForm(webhookURL, url.Values{"payload": {string(str)}})
  if err != nil {
    return err
  }
  if resp.StatusCode != http.StatusOK {
    return fmt.Errorf("Notify Response Status Code is wrong\n%v+", resp)
  }
  defer resp.Body.Close()
  return nil
}
```

### スクリプトの解説

#### 新しく Subscription filter と通知先を設定する

Pattern: subscription_filter の名前に一致するパターンを記述します。  
URL: 通知先の WebHook URL を記述します。

```go
var webhooks = []Webhook{
  {
    Pattern: "*LambdaStream_cloudwatchlogs-node*",
    URL:     "https://hooks.slack.com/services/xxxxxxxxxxx/xxxxxxxxxxx/xxxxxxxxxxxxxxxxxxxxxxxx",
  },
  {
    Pattern: "subscription_filter_name",
    URL:     "https://hooks.slack.com/services/xxxxxxxxxxx/xxxxxxxxxxx/xxxxxxxxxxxxxxxxxxxxxxxx",
  },
}
```

#### github.com/gobwas/glob の使用

Glob match を使用するために以下の package を使用しています。
[GitHub - gobwas/glob: Go glob](https://github.com/gobwas/glob)
以下が Match Example です。

```go
displayMatch := func(match string, path string) {
  g := glob.MustCompile(match, '/')
  fmt.Fprintln(os.Stdout, match, "\t", path, "\t", g.Match(path))
}

displayMatch("*.js", "a.js")             // true
displayMatch("{a,b,c}.js", "a.js")       // true
displayMatch("[a-c].js", "a.js")         // true
displayMatch("[!a-c].js", "d.js")        // true
displayMatch("**.js", "a/b.js")          // true
displayMatch("?????at.js", "12345at.js") // true
displayMatch("*/*.js", "a/b.js")         // true
displayMatch("a/*.js", "a/b.js")         // true
displayMatch("a/**.js", "a/b.js")        // true
displayMatch("*testurl*", "a/testurl")   // false
displayMatch("*.js", "a/b.js")      // false
displayMatch("{a,b,c}.js", "d.js")  // false
displayMatch("[a-c].js", "d.js")    // false
displayMatch("[1-8].js", "9.js")    // false
displayMatch("b/*.js", "a/b.js")    // false
displayMatch("a/**/*.js", "a/b.js") // false
```

#### WebHook に通知する際の型

非常にさまざまなレイアウトがあります。  
今回は Block Layout を使用するためそれに合わせた方を定義します。  
[Reference: Layout blocks | Slack](https://api.slack.com/reference/block-kit/blocks)

```go
type Payload struct {
  Username   string         `json:"username"`
  Icon_emoji string         `json:"icon_emoji"`
  Blocks     []PayloadBlock `json:"blocks"`
}

type PayloadBlock struct {
  Type string      `json:"type"`
  Text PayloadText `json:"text"`
}

type PayloadText struct {
  Type string `json:"type"`
  Text string `json:"text"`
}

type Webhook struct {
  Pattern string `json:"pattern"`
  URL     string `json:"url"`
}
```

実際に送る際の例では、以下のような Json 形式になります。

```json
{
  "username": "webhookbot",
  "icon_emoji": ":ghost:",
  "blocks": [
    {
      "type": "header",
      "text": {
        "type": "plain_text",
        "text": "New request"
      }
    },
    {
      "type": "section",
      "fields": [
        {
          "type": "mrkdwn",
          "text": "*Type:*\nPaid Time Off"
        },
        {
          "type": "mrkdwn",
          "text": "*Created by:*\n<example.com|Fred Enriquez>"
        }
      ]
    },
    {
      "type": "section",
      "fields": [
        {
          "type": "mrkdwn",
          "text": "*When:*\nAug 10 - Aug 13"
        }
      ]
    },
    {
      "type": "section",
      "text": {
        "type": "mrkdwn",
        "text": "<https://example.com|View request>"
      }
    }
  ]
}
```

実際に送れる Json か否かかは、curl で POST して見てみましょう。

```
curl -X POST --data-urlencode "payload=$(cat payload.json)" https://hooks.slack.com/services/xxxxxxxxxxx/xxxxxxxxxxx/xxxxxxxxxxxxxxxxxxxxxxxx
```

また、今回とは別ですが、<https://github.com/slack-go/slack> の package の types は便利に利用できそうだとおもうので後で見てみるといいです。

## ロググループ毎に通知するに変更する

以下の部分のように変更しましょう。

```go
var webhooks = []Webhook{
  {
    Pattern: "/aws/testurl/echo-nodejs",
    URL:     "https://hooks.slack.com/services/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",
  },
}
...

webhookURL := getWebhook(l.LogGroup).URL
if webhookURL == "" {
  return fmt.Errorf("LogFilter Name does not match a URL")
}
```

## Build function for lambda

デプロイ前に、Lambda Runtime ように Build しましょう。

```
GOOS=linux go build -o main
```

## Terraform

terraform で、CloudWatch logs Subscription filter を設定する。  
コアな部分だと基本的に以下のリソースで構成されます。参考にしてください。

#### cloudwatch.tf

filter_pattern: [Using CloudWatch Logs subscription filters - Amazon CloudWatch Logs](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/SubscriptionFilters.html)

```hcl
# https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/cloudwatch_log_group
resource "aws_cloudwatch_log_group" "this" {
  name = "test"
}

# https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/cloudwatch_log_subscription_filter
resource "aws_cloudwatch_log_subscription_filter" "this" {
  name            = "${var.service}-test"
  log_group_name  = "/ecs/${var.service}-test"
  filter_pattern  = "{ ($.status = 5*) }"
  destination_arn = aws_lambda_function.this.arn
  depends_on      = [aws_lambda_permission.this]
}
```

#### lambda.tf

```hcl
data "archive_file" "this" {
  type        = "zip"
  source_file = "../functions/cloudwatch_log_filter_notify/main"
  output_path = "../functions/cloudwatch_log_filter_notify/main.zip"
}

# https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/lambda_function
resource "aws_lambda_function" "this" {
  function_name    = "${var.service}-${var.env}-cloudwatch"
  handler          = "main"
  runtime          = "go1.x"
  role             = aws_iam_role.cloudwatch_this.arn
  filename         = data.archive_file.this.output_path
  source_code_hash = data.archive_file.this.output_base64sha256
  publish          = true
  memory_size      = 128
  timeout          = 3
}

# https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/lambda_permission
resource "aws_lambda_permission" "this" {
  action        = "lambda:InvokeFunction"
  principal     = "logs.amazonaws.com"
  function_name = aws_lambda_function.this.function_name
  source_arn    = "${aws_cloudwatch_log_group.this.arn}:*"
}
```

#### iam.tf

```hcl
# https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_role
resource "aws_iam_role" "cloudwatch" {
  name = "${var.service}-${var.env}-cloudwatch"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = ["sts:AssumeRole"]
        Principal = {
          Service = "lambda.amazonaws.com"
        }
      }
    ]
  })
}

# https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_policy
resource "aws_iam_policy" "cloudwatch" {
  name = "${var.service}-${var.env}-cloudwatch"
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid = "VisualEditor0"
        Action = [
          "logs:CreateLogGroup",
          "logs:CreateLogStream",
          "logs:PutLogEvents"
        ]
        Effect = "Allow"
        Resource = [
          "*"
        ]
      }
    ]
  })
}

# https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_role_policy_attachment
resource "aws_iam_role_policy_attachment" "lambda" {
  role       = aws_iam_role.cloudwatch.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
}

resource "aws_iam_role_policy_attachment" "cloudwatch" {
  role       = aws_iam_role.cloudwatch.name
  policy_arn = aws_iam_policy.cloudwatch.arn
}
```

## 終わりに

複数のサービスからのログを Slack 通知して生産性向上やサービス改善に生かしましょう。  
部分的ではありますが、今回の記事が役に立つと幸いです。  
また、関数の改善があったり間違いがあれば逐一修正していこうと思います。
