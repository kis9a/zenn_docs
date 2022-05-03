---
title: "AWS WAF の設定 IP, Request, Rate limit 等でのアクセス制限"
emoji: "☁"
type: "tech"
topics: ["AWS", "WAF", "terraform"]
published: true
---

## WAF

WAF の一般的な FW (FireWall) との違いは、FW は、通信における送信元情報と送信先情報(IP アドレスやポート番号等)を基にアクセスを制限するのに対し、WAF は、 FW で制限できないウェブアプリケーションへの通信内容を検査することができ、さらに細かなアプリケーションへの脆弱性攻撃に対しての事前対策として制限やルールを定義することが有効です。  
詳しくは、 [IPA Web Application Firewall(WAF)読本](https://www.ipa.go.jp/security/vuln/waf.html)　を呼んでみましょう。
ここで言う一般的な FW は、AWS では、[Security Group](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-security-groups.html) のイメージです。
[AWS WAF (V2)](https://docs.aws.amazon.com/ja_jp/waf/latest/APIReference/Welcome.html) については、Classmethod さんの記事が良かったので読んでみてください。  
[AWS WAF を完全に理解する WAF の基礎から v2 の変更点まで](https://dev.classmethod.jp/articles/fully-understood-aws-waf-v2/)  
記事では、実際に使用した AWS WAF の設定で Terraform で記述したものを載せていこうと思います。

##### 料金

WAF の注意点ですが、WebACL やルールの作成に対しての料金がかかります。
https://aws.amazon.com/jp/waf/pricing/

![aws-waf-price-summary-1](/images/aws-waf-price-summary-1.png)

### Frontend: 特定の IP からのみアクセスを許可する

Cloudfront の Web ACL(Access Contrll List) に対して作成した WAF の ACL をアタッチします。  
IP set には、 VPN の IP と オフィスの IP を指定し、この IP のアクセスのみ許可します。  
開発環境やステージング環境など外部に公開する必要のない物などに対してこの制限が有効です。
Code では、AWS WAF V1 のリソースで作成しています。ホワイトリスト制限です。  
Terraform の ドキュメントの Example では、は逆に特定の IP をブラックリストとして指定する例が書かれていますね。 [Terraform Registry - resources/waf_web_acl](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/waf_web_acl)

```hcl
resource "aws_cloudfront_distribution" "frontend" {
  web_acl_id          = var.waf_acl_id
  # 省略
}

resource "aws_waf_ipset" "frontend_ip_limit" {
  name = "${var.service}-frontend-ip-limit"

  dynamic "ip_set_descriptors" {
    for_each = concat([for ip in var.office_ips : "${ip}/32"], [for ip in var.vpn_ips : "${ip}/32"]

    content {
      type  = "IPV4"
      value = ip_set_descriptors.value
    }
  }
}

resource "aws_waf_rule" "frontend_ip_limit" {
  depends_on  = [aws_waf_ipset.frontend_ip_limit]
  name        = "${var.service}-frontend-ip-limit"
  metric_name = "${var.service}FrontendIpLimit"

  predicates {
    data_id = aws_waf_ipset.frontend_ip_limit.id
    negated = false
    type    = "IPMatch"
  }
}

resource "aws_waf_web_acl" "frontend_ip_limit" {
  depends_on  = [aws_waf_ipset.frontend_ip_limit, aws_waf_rule.frontend_ip_limit]
  name        = "${var.service}-frontend-ip-limit"
  metric_name = "${var.service}FrontendIpLimit"

  default_action {
    type = "BLOCK"
  }

  rules {
    action {
      type = "ALLOW"
    }

    priority = 1
    rule_id  = aws_waf_rule.frontend_ip_limit.id
    type     = "REGULAR"
  }
}
```

### API: /v1/admin/\* 以下の IP 制限

次は、/v1/admin 以下のエンドポイントへのリクエストを VPN の IP と オフィスの IP に制限します。WAF V2 のリソースで作成して、statement, and_statement などの 論理ステートメント、一致ステートメント として "IP セット一致", "正規表現パターンセット"とうを使用し ACL のルースを記述しています。管理者用や、外部公開せずに一部の企業が使用したり、テスト関係者が使用する API などの要件に有効だと考えられますね。

ルールステートメントリストについては以下にドキュメントがあります。[ルールステートメントリスト - AWS WAF、AWS Firewall Manager、および AWS Shield Advanced](https://docs.aws.amazon.com/ja_jp/waf/latest/developerguide/waf-rule-statements-list.html) / Terraform での記述に関しては、[Terraform Registry - resources/wafv2_web_acl](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/wafv2_web_acl) を参考にできます。

```hcl
resource "aws_wafv2_regex_pattern_set" "admin" {
  name  = "${var.service}-admin-api-path-pattern"
  scope = "REGIONAL"

  regular_expression {
    regex_string = "/v1/admin.*"
  }
}

resource "aws_wafv2_ip_set" "admin" {
  name               = "${var.service}-admin-ips"
  scope              = "REGIONAL"
  ip_address_version = "IPV4"
  addresses          = concat([for ip in var.office_ips : "${ip}/32"], [for ip in var.vpn_ips : "${ip}/32"])
}

resource "aws_wafv2_web_acl" "admin" {
  name  = "${var.service}-admin-waf"
  scope = "REGIONAL"

  default_action {
    allow {}
  }

  rule {
    name     = "${var.service}-admin-path-ip-limit"
    priority = 2

    action {
      block {}
    }

    statement {
      and_statement {
        statement {
          regex_pattern_set_reference_statement {
            arn = aws_wafv2_regex_pattern_set.admin.arn

            field_to_match {
              uri_path {
              }
            }

            text_transformation {
              priority = 1
              type     = "NONE"
            }
          }
        }

        statement {
          not_statement {
            statement {
              ip_set_reference_statement {
                arn = aws_wafv2_ip_set.admin.arn
              }
            }
          }
        }
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = false
      metric_name                = "${var.service}-admin-path-ip-limit-rule"
      sampled_requests_enabled   = false
    }
  }

  visibility_config {
    cloudwatch_metrics_enabled = false
    metric_name                = "${var.service}-admin-path-ip-limit"
    sampled_requests_enabled   = false
  }
}

resource "aws_wafv2_web_acl_association" "admin" {
  resource_arn = aws_alb.api.arn
  web_acl_arn  = aws_wafv2_web_acl.admin.arn
}
```

### API: 特定のメールアドレスを含むリクエストをブロックする。

次は、URI に /v1/account を含む PATH に対して、body に @unkonow の文字列を含む POST リクエストをブロックします。バイトマッチステートメントという文字列一致検索を定義するものを使います。その他にも RegexPatternSetReferenceStatement や　 SizeConstraintStatement というものもあってこの時、色々できそうです。positional_constraint や field_to_match の種類も乗っているのでドキュメントをみてみるといいでしょう。

[ByteMatchStatement - AWS WAFV2](https://docs.aws.amazon.com/ja_jp/waf/latest/APIReference/API_ByteMatchStatement.html)

> A rule statement that defines a string match search for AWS WAF to apply to web requests. The byte match statement provides the bytes to search for, the location in requests that you want AWS WAF to search, and other settings. The bytes to search for are typically a string that corresponds with ASCII characters.

B2C の Web サービスでアカウント登録やログインにインセンティブをつけると、どうしても BOT から万単位のリクエストを受けたりします。最近は、メール認証のアクティベーションも自動でしてくる BOT もあり Web サービスの規模や施策に対しての対策が大変ですね。よくわからない @unkonow みたいなドメインのメールアドレスにメール認証のためのメールを AWS SES から数十万送っていると、`存在しているかわからないメールアドレスにメールを送りすぎだ。`　と AWS から警告メールがきたり、最悪の場合停止されることもあるようなので、監視を強化しつつブラックリストや、新たに対策を追加していきたいところです。

```hcl
resource "aws_wafv2_web_acl" "mail_limit" {
  name  = "${var.service}-body-limit-waf"
  scope = "REGIONAL"

  default_action {
    allow {}
  }

  rule {
    name     = "${var.service}-mail-limit"
    priority = 2

    action {
      block {}
    }

    statement {
      and_statement {
        # body
        statement {
          byte_match_statement {
            field_to_match {
              body {}
            }
            positional_constraint = "CONTAINS"
            search_string         = "@unkonow"

            text_transformation {
              priority = 1
              type     = "NONE"
            }
          }
        }

        # uri
        statement {
          byte_match_statement {
            field_to_match {
              uri_path {}
            }
            positional_constraint = "CONTAINS"
            search_string         = "/v1/account"

            text_transformation {
              priority = 1
              type     = "LOWERCASE"
            }
          }
        }

        # method
        statement {
          byte_match_statement {
            field_to_match {
              method {}
            }
            positional_constraint = "CONTAINS"
            search_string         = "post"

            text_transformation {
              priority = 1
              type     = "LOWERCASE"
            }
          }
        }
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = false
      metric_name                = "${var.service}-mail-limit-rule"
      sampled_requests_enabled   = false
    }
  }

  visibility_config {
    cloudwatch_metrics_enabled = false
    metric_name                = "${var.service}-mail-limit"
    sampled_requests_enabled   = false
  }
}

resource "aws_wafv2_web_acl_association" "mail_limit" {
  resource_arn = aws_alb.api.arn
  web_acl_arn  = aws_wafv2_web_acl.mail_limit.arn
}
```

### API: /v1/limit/\* 以下に対しての５分間のリクエスト数を制限する

Rate limit とは、API を一定時間あたりに使用できる回数をさします。
GitHub API とか、Twitter API とかにもついているあれです。
/v1/limit/\* 以下のエンドポイントに対するリクエストを５分間に 1000 に制限します。
毎回、過去 5 分間のリクエストをカウント、30 秒ごとにリクエストのレートをチェックが要件時に対しての選定基準になるところでしょう。結構不自由で不正確だと個人的には思ってしまいます。[レートベースのルールステートメント - AWS WAF、AWS Firewall Manager、および AWS Shield Advanced](https://docs.aws.amazon.com/ja_jp/waf/latest/developerguide/waf-rule-statement-type-rate-based.html)

> AWS WAF checks the rate of requests every 30 seconds, and counts requests for the prior 5 minutes each time. Because of this, it's possible for an IP address to send requests at too high a rate for 30 seconds before AWS WAF detects and blocks it. AWS WAF can block up to 10,000 IP addresses.
> AWS WAF は、30 秒ごとにリクエストのレートをチェックし、毎回、過去 5 分間のリクエストをカウントします。このため、AWS WAF がそれを検出してブロックする前に、IP アドレスが 30 秒間、高すぎるレートでリクエストを送信する可能性があります。 AWS WAF は、最大 10,000 の IP アドレスをブロックできます。

昔は、５分間で 2000 が制限の最小値で、

> 「・・・ん？」 「5 分で 2000 リクエストからしか指定できないなのか。。ウチの用途にはハマらんな。。」

みたいなことを言われていたようですが、最近の Update で 100 リクエストから指定できるようになっているようです。 [[アップデート] AWS WAF の Rate-based ルールが 100 リクエストから指定可能になりました | DevelopersIO](https://dev.classmethod.jp/articles/lower-threshold-for-aws-waf-rate-based-rules/)

```hcl
## ip rate limit
resource "aws_wafv2_regex_pattern_set" "rate_limit" {
  name  = "${var.service}-limit-api-path-set"
  scope = "REGIONAL"

  regular_expression {
    regex_string = "/v1/limit/.*"
  }
}

resource "aws_wafv2_web_acl" "rate_limit" {
  name  = "${var.service}-limit-waf"
  scope = "REGIONAL"

  default_action {
    allow {}
  }

  rule {
    name     = "${var.service}-rate-limit-api"
    priority = 2

    action {
      block {}
    }

    statement {
      rate_based_statement {
        limit              = 1000
        aggregate_key_type = "IP"

        scope_down_statement {
          regex_pattern_set_reference_statement {
            arn = aws_wafv2_regex_pattern_set.rate_limit.arn

            field_to_match {
              uri_path {
              }
            }

            text_transformation {
              priority = 1
              type     = "NONE"
            }
          }
        }
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = false
      metric_name                = "${var.service}-rate-limit-api-rule"
      sampled_requests_enabled   = false
    }
  }

  visibility_config {
    cloudwatch_metrics_enabled = false
    metric_name                = "${var.service}-rate-limit-api"
    sampled_requests_enabled   = false
  }
}

resource "aws_wafv2_web_acl_association" "rate_limit" {
  resource_arn = aws_alb.api.arn
  web_acl_arn  = aws_wafv2_web_acl.rate_limit.arn
}
```

実際にリクエストを送って実行数とアクセス制限の関係性をみてみましょう。
ab (Apatch Bench) を使用して、2000 回リクエストを送ってみます。制限されることを確認できますか？

```bash
# -v verbosity    How much troubleshooting info to print
# -c concurrency  Number of multiple requests to make at a time
# -n requests     Number of requests to perform

ab -n 2000 -c 1 -v 2 https://example-api.com/v1/account
```

##### Rate limit の注意点

以下のスクリプトは、200 のレスポンスが、制限され 403 のレスポンスになる、それが解除され 200 のレスポンスに戻る様子の時間を出力します。実際に実行してみると指定した、1000 より多くのリクエストを投げても制限されず、1500 ~ 2000 の間で自分環境では、制限され、制限解除になる時間も思ったより短かったです。ALB の monitoring タブで実際のリクエスト数をみてみると ちょうど 1000 くらいだったのでリクエストする方でキャッシュされているのかどうか？チェックされる 30 秒間にリクエストが超過しているのか、ここらへんの仕様は時間がある時に、もう少し詳しくみてみる必要があると思いました。５分間で 1000 に制限していますが、もう少し少ない数にしてもいいかもしれませんね。もしもっと正確にリクエスト数を制限する要件になったら、Backend 側で Elasticache などの key-value キャッシュストアに、同一 IP に対してのリクエスト数をインクリメントして、制限時間で expir するとかも考えています。

```bash
#!/bin/bash
get_url_status_code() {
  cat <<<"$(curl -X GET -LI "$1" -o /dev/null -w '%{http_code}\n' -s)"
}

show_date() {
  date | cut -c9-19 | tr "\ :" "_"
}

limited=false
for ((i = 0; i < 2500; i++)); do
  if [[ "$limited" == true ]]; then
    sleep 1
  fi
  code="$(get_url_status_code "https://example-api.com/v1/account/health")"
  if [[ "$code" -eq "200" ]]; then
    echo "200: $(show_date)"
    if [[ "$limited" == true ]]; then
      show_date
      exit 0
    fi
  else
    if [[ "$code" -eq "403" ]]; then
      echo "403: $(show_date)"
      limited=true
    fi
  fi
done
```

### 必要な Managed rules をまとめた WebACL を使用する。

AWS によって管理されている一般的な WAF の rules です。必要そうな、Rule をまとめて WebACL を定義し、必要なリソース等にアタッチするといいと思います。以下は使用したルールの概要とコードです。

https://docs.aws.amazon.com/ja_jp/waf/latest/developerguide/aws-managed-rule-groups-list.html

##### AWSManagedRulesUnixRuleSet

> POSIX オペレーティングシステムルールグループには、POSIX および POSIX と同等のオペレーティングシステムに固有の脆弱性の悪用 (ローカルファイルインクルージョン (LFI) 攻撃など) に関連するリクエストパターンをブロックするルールが含まれています。これにより、攻撃者がアクセスしてはならないファイルの内容を公開したり、コードを実行したりする攻撃を防ぐことができます。アプリケーションの一部が POSIX または POSIX と同等のオペレーティングシステム (Linux、AIX、HP-UX、macOS、Solaris、FreeBSD、OpenBSD など) で実行されている場合は、このルールグループを評価する必要があります。

##### AWSManagedRulesLinuxRuleSet

> Linux オペレーティングシステムルールグループには、Linux 固有のローカルファイルインクルージョン (LFI) 攻撃など、Linux 固有の脆弱性の悪用に関連するリクエストパターンをブロックするルールが含まれています。これにより、攻撃者がアクセスしてはならないファイルの内容を公開したり、コードを実行したりする攻撃を防ぐことができます。アプリケーションの一部が Linux で実行されている場合は、このルールグループを評価する必要があります。このルールグループは、POSIX operating system ルールグループと組み合わせて使用する必要があります。

##### AWSManagedRulesSQLiRuleSet

> sql Database ルールグループには、SQL インジェクション攻撃などの SQL データベースの悪用に関連するリクエストパターンをブロックするルールが含まれています。これにより、不正なクエリのリモートインジェクションを防ぐことができます。アプリケーションが SQL データベースと連結している場合は、このルールグループを評価します。

##### AWSManagedRulesAmazonIpReputationList

> Amazon IP 評価リストルールグループには、Amazon 内部脅威インテリジェンスに基づくルールが含まれています。これは、通常、ボットやその他の脅威に関連付けられている IP アドレスをブロックする場合に便利です。これらの IP アドレスをブロックすることで、ボットを軽減し、悪意のあるアクターが脆弱なアプリケーションを発見するリスクを軽減できます。

##### AWSManagedRulesKnownBadInputsRuleSet

既知の不正な入力ルールグループには、無効であることがわかっており脆弱性の悪用または発見に関連するリクエストパターンをブロックするルールが含まれています。これにより、悪意のあるアクターが脆弱なアプリケーションを発見するリスクを軽減できます。

##### AWSManagedRulesCommonRuleSet

> コアルールセット (CRS) ルールグループには、ウェブアプリケーションに一般的に適用可能なルールが含まれています。これにより、OWASP の出版物に記載されている高リスクの脆弱性や一般的な脆弱性など、さまざまな脆弱性の悪用に対する保護が提供されます。Owasp Top 10。すべての AWS WAF ユースケースでこのルールグループを使用することを検討してください。

```hcl
resource "aws_wafv2_web_acl" "managed" {
  name  = "${var.service}-web-acl-managed-rules"
  scope = "REGIONAL"

  default_action {
    allow {}
  }

  rule {
    name     = "AWSManagedRulesCommonRuleSet"
    priority = 10

    override_action {
      count {}
    }

    statement {
      managed_rule_group_statement {
        name        = "AWSManagedRulesCommonRuleSet"
        vendor_name = "AWS"

        excluded_rule {
          name = "SizeRestrictions_QUERYSTRING"
        }

        excluded_rule {
          name = "NoUserAgent_HEADER"
        }
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = false
      metric_name                = "AWSManagedRulesCommonRuleSetMetric"
      sampled_requests_enabled   = false
    }
  }

  rule {
    name     = "AWSManagedRulesKnownBadInputsRuleSet"
    priority = 20

    override_action {
      count {}
    }

    statement {
      managed_rule_group_statement {
        name        = "AWSManagedRulesKnownBadInputsRuleSet"
        vendor_name = "AWS"
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = false
      metric_name                = "AWSManagedRulesKnownBadInputsRuleSetMetric"
      sampled_requests_enabled   = false
    }
  }

  rule {
    name     = "AWSManagedRulesAmazonIpReputationList"
    priority = 30

    override_action {
      count {}
    }

    statement {
      managed_rule_group_statement {
        name        = "AWSManagedRulesAmazonIpReputationList"
        vendor_name = "AWS"
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = false
      metric_name                = "AWSManagedRulesAmazonIpReputationListMetric"
      sampled_requests_enabled   = false
    }
  }

  rule {
    name     = "AWSManagedRulesSQLiRuleSet"
    priority = 50

    override_action {
      count {}
    }

    statement {
      managed_rule_group_statement {
        name        = "AWSManagedRulesSQLiRuleSet"
        vendor_name = "AWS"
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = false
      metric_name                = "AWSManagedRulesSQLiRuleSetMetric"
      sampled_requests_enabled   = false
    }
  }

  rule {
    name     = "AWSManagedRulesLinuxRuleSet"
    priority = 60

    override_action {
      count {}
    }

    statement {
      managed_rule_group_statement {
        name        = "AWSManagedRulesLinuxRuleSet"
        vendor_name = "AWS"
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = false
      metric_name                = "AWSManagedRulesLinuxRuleSetMetric"
      sampled_requests_enabled   = false
    }
  }

  rule {
    name     = "AWSManagedRulesUnixRuleSet"
    priority = 70

    override_action {
      count {}
    }

    statement {
      managed_rule_group_statement {
        name        = "AWSManagedRulesUnixRuleSet"
        vendor_name = "AWS"
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = false
      metric_name                = "AWSManagedRulesUnixRuleSetMetric"
      sampled_requests_enabled   = false
    }
  }

  visibility_config {
    cloudwatch_metrics_enabled = false
    metric_name                = "TerraformWebACLMetric"
    sampled_requests_enabled   = false
  }
}

output "managed_rule_arn" {
  value = aws_wafv2_web_acl.managed.arn
}
```

### WAF Cloudfront basic 認証

以下の記事を参考に WAF リソースでも Cloudfront に basic 認証をつけることができます。 以下は実際にやってみたコードです。今までは、 Lambda@Edge でやって管理コストがかかっていたので、WAF だけでつけることができるのは、最高です。

https://dev.classmethod.jp/articles/aws-waf-basic-auth/

kms.tf

```hcl
resource "aws_kms_key" "dev" {
  description             = var.service
  enable_key_rotation     = true
  is_enabled              = true
  deletion_window_in_days = 7
}

resource "aws_kms_alias" "dev" {
  name          = "alias/dev"
  target_key_id = aws_kms_key.dev.key_id
}
```

~/.zshrc

```bash
function kms_encrypt() {
  if [[ "$#" -lt 1 ]]; then
    echo 'kms_encrypt $alias_suffix $input_filepath $profile $region'
  else
    keyId=$(aws kms describe-key --key-id alias/"$1" --profile "${3:=default}" --region "${4:=ap-northeast-1}" --output json | jq -r .KeyMetadata.KeyId)
    aws kms encrypt --key-id "$keyId" --region "${4:=ap-northeast-1}" --plaintext fileb://"$2" \
      --output text --query CiphertextBlob --profile "${3:=default}"
  fi
}

function kms_decrypt() {
  if [[ "$#" -lt 1 ]]; then
    echo 'kms_decrypt $input_filepath $profile $region'
  else
    aws kms decrypt --ciphertext-blob fileb://<(cat "$1" | base64 -D) --output json --profile "$2" --region "${3:=ap-northeast-1}" |
      jq .Plaintext --raw-output | base64 -D
  fi
}
```

secrets.yaml

```
basic_auth_user: gJ3eTjnIoWiS9ee4
basic_auth_password: FKRsxJ4Ngdg4bZ4d
```

```
$ kms_encrypt dev secrets.yaml aws-profile > secrets.yaml.encrypted
$ kms decrypt secrets.yaml.encrypted aws-profile
```

secrets.yaml.encrypted

```hcl
data "aws_kms_secrets" "secrets" {
  secret {
    name    = "secrets"
    payload = file("${path.module}/secrets.yaml.encrypted")
  }
}

locals {
  s = yamldecode(data.aws_kms_secrets.secrets.plaintext["secrets"])
}

output "basic_auth_info" {
  value     = join(" ", ["Basic", base64encode(join(":", ["${local.s.basic_auth_user}", "${local.s.basic_auth_password}"]))])
  sensitive = true
}

output "cloudfront_basic_authentication" {
  value =  aws_wafv2_web_acl.cloudfront_basic_authentication.arn
}

resource "aws_wafv2_web_acl" "cloudfront_basic_authentication" {
  provider = aws.virginia
  name     = "waf"
  scope    = "CLOUDFRONT"

  default_action {
    allow {}
  }

  rule {
    name     = "basic-auth"
    priority = 0

    action {
      block {
        custom_response {
          response_code = 401
          response_header {
            name  = "www-authenticate"
            value = "Basic"
          }
        }
      }
    }

    statement {
      not_statement {
        statement {
          byte_match_statement {
            positional_constraint = "EXACTLY"
            search_string         = join(" ", ["Basic", base64encode(join(":", ["${local.s.basic_auth_user}", "${local.s.basic_auth_password}"]))])
            field_to_match {
              single_header {
                name = "authorization"
              }
            }
            text_transformation {
              priority = 0
              type     = "NONE"
            }
          }
        }
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "cloudfront-basic-authentication-rule"
      sampled_requests_enabled   = true
    }
  }

  visibility_config {
    cloudwatch_metrics_enabled = true
    metric_name                = "cloudfront-basic-authentication"
    sampled_requests_enabled   = true
  }
}
```

```
$ terraform output basic_auth_info | tr -d \" | cut -f 2 -d " " | base64 -d
gJ3eTjnIoWiS9ee4:FKRsxJ4Ngdg4bZ4d
```

##### 特定の PATH だけ Basic 認証

特定の PATH /tags\* にだけ Basic 認証を使用する例

![aws-waf-basic-auth-tags-paths](/images/aws-waf-basic-auth-tags-paths.png)

```hcl
resource "aws_wafv2_regex_pattern_set" "tags_paths" {
  provider = aws.virginia
  name     = "${var.service}-tags-paths"
  scope    = "CLOUDFRONT"

  regular_expression {
    regex_string = "tags.*"
  }
}


resource "aws_wafv2_web_acl" "cloudfront_basic_authentication" {
# 略
  rule {
    name     = "basic-auth"
    priority = 0

    action {
      block {
        custom_response {
          response_code = 401
          response_header {
            name  = "www-authenticate"
            value = "Basic"
          }
        }
      }
    }

    statement {
      and_statement {
        statement {
          regex_pattern_set_reference_statement {
            arn = aws_wafv2_regex_pattern_set.tags_paths.arn

            field_to_match {
              uri_path {
              }
            }

            text_transformation {
              priority = 1
              type     = "NONE"
            }
          }
        }

        statement {
          not_statement {
            statement {
              byte_match_statement {
                positional_constraint = "EXACTLY"
                search_string         = join(" ", ["Basic", base64encode(join(":", ["${local.s.basic_auth_user}", "${local.s.basic_auth_password}"]))])

                field_to_match {
                  single_header {
                    name = "authorization"
                  }
                }
                text_transformation {
                  priority = 0
                  type     = "NONE"
                }
              }
            }
          }
        }
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "cloudfront-basic-authentication-rule"
      sampled_requests_enabled   = true
    }
  }
}
```

### 終わりに

今回は、AWS WAF を実際使用した例を terraform のコードで紹介しました、実際、実用性があるものも多かったのではないでしょうか？ AWS WAF は、いろいろパラメータがあって設定するのが面白いですね。ルールを記述していると背徳感を味わえます。中小〜大規模な Architecture 設計など複雑な要件のものではなく、このような シンプルな props の記述は気が楽ですね、最近 terraform で作った AWS のリソースが言うことを聞いてくれるようになってきたので、割と楽しくなってきました。もっといろいろなものを IaC (コードで管理)できるようになりたいです。今後もこのような便利なクラウドサービスを使用して、Web サービスの品質や可用性に貢献していきたいです。
