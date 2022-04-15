---
title: "最近、業務で使用した AWS WAF の設定、IP, Request, Path 制限など。 [Terraform]"
emoji: "☁"
type: "tech"
topics: ["terraform", "WAF", "AWS"]
published: false
---

## WAF

WAF の一般的な FW (FireWall) との違いは、FW は、通信における送信元情報と送信先情報(IP アドレスやポート番号等)を基にアクセスを制限するのに対し、WAF は、 FW で制限できないウェブアプリケーションへの通信内容を検査することができ、さらに細かなアプリケーションへの脆弱性攻撃に対しての事前対策として細かな制限やルールを定義することが有効です。  
詳しくは、 [IPA Web Application Firewall(WAF)読本](https://www.ipa.go.jp/security/vuln/waf.html)　を呼んでみましょう。
ここで言う一般的な FW は、AWS では、[Security Group](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-security-groups.html) のイメージです。
[AWS WAF (V2)](https://docs.aws.amazon.com/ja_jp/waf/latest/APIReference/Welcome.html) については、Classmethod さんの記事が良かったので読んでみてください。  
[AWS WAF を完全に理解する WAF の基礎から v2 の変更点まで](https://dev.classmethod.jp/articles/fully-understood-aws-waf-v2/)  
記事では、自分が業務で実際に使用した AWS WAF の設定で Terraform で記述したものを載せていこうと思います。

### Frontend 特定の IP からのみアクセスを許可する

Web ACL(Access Contrll List) に対して作成した WAF の ACL をアタッチします。

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

### API 特定のメールアドレスを含むリクエストを拒否する。

```hcl
resource "aws_wafv2_web_acl" "mail_limit" {
  name  = "${var.service}-body-limit-waf"
  scope = "REGIONAL"

  default_action {
    allow {}
  }

  rule {
    name     = "${var.service}-mail-limit"
    priority = 1

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

### API /v1/admin/\* 以下の IP 制限

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
    priority = 1

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

### Rate limit

```hcl
## ip rate limit
resource "aws_wafv2_regex_pattern_set" "rate_limit" {
  name  = "${var.service}-limit-api-path-set"
  scope = "REGIONAL"

  regular_expression {
    regex_string = "/v1/rate/limit/.*"
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
    priority = 1

    action {
      block {}
    }

    statement {
      rate_based_statement {
        limit              = 100
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
