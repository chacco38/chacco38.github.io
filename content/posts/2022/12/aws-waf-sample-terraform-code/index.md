---
title: "TerraformでAWS WAFを構築するサンプルコードを書いてみた"
date: 2022-12-13T00:00:00+09:00
lastmod: 2022-12-17T00:00:00+09:00
tags: ["Terraform", "AWS", "AWS WAF", "Amazon CloudFront", "Amazon API Gateway"]
draft: false
externalUrl: https://qiita.com/chacco38/items/20553b8967d77028a8c5
---

みなさん、こんにちは。今回はTerraformの入門ということでCloudFrontおよびAPI Gatewayに割り当てるAWS WAFのサンプルコードを書いてみましたのでこちらを紹介していきたいと思います。

なお、サンプルコードを書いた際のTerraformおよびAWSプロバイダーのバージョンは次のとおりです。最新バージョンでは定義方法が異なっている可能性があるため、実際にコードを書く際は最新の「[Terraformドキュメント]」と「[AWSプロバイダードキュメント]」を確認しながら開発を進めていただければと思います。

[Terraformドキュメント]: https://developer.hashicorp.com/terraform/docs
[AWSプロバイダードキュメント]: https://registry.terraform.io/providers/hashicorp/aws/latest/docs

```tf:versions.tf
# Requirements
terraform {
  required_version = "1.3.6"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "4.46.0"
    }
  }
}
```

<!-- omit in toc -->
## サンプルコードを書いてみた

今回は次のようなサンプルコードを書いてみました。なお、変数定義部分などの一部省略している点、ならびにステップごとの細かい説明などは省いていますのでご承知おきください。詳細については「[AWSプロバイダードキュメント]」をご参照ください。

- [例1. 汎用的なCloudFront向け定義](#例1-汎用的なcloudfront向け定義)
- [例2. 汎用的なAPI Gateway向け定義](#例2-汎用的なapi-gateway向け定義)
- [例3. IPホワイトリストを用いたアクセス制御](#例3-ipホワイトリストを用いたアクセス制御)
- [例4. カスタムヘッダーを用いたアクセス制御](#例4-カスタムヘッダーを用いたアクセス制御)
- [例5. CloudWatch Logsを活用したロギング設定](#例5-cloudwatch-logsを活用したロギング設定)
- [例6. S3バケットを活用したロギング設定](#例6-s3バケットを活用したロギング設定)

ちなみに、すべてのサンプルコードに共通してプロバイダー定義は次のようにしています。

```tf:providers.tf
# デフォルトのプロバイダー設定
provider "aws" {
  region = var.aws_region
  default_tags {
    tags = {
      terraform = "true"
    }
  }
}

# CloudFront向けのプロバイダー設定
provider "aws" {
  alias  = "us_east_1"
  region = "us-east-1"
  default_tags {
    tags = {
      terraform = "true"
    }
  }
}
```

### 例1. 汎用的なCloudFront向け定義

次のAWSマネージドルールを組み合わせてそれっぽい汎用的なCloudFront向けのWAF設定を記載してみました。Linux OS以降はアプリケーションの実装により適宜変えるイメージです。

- [Amazon IP 評価リストマネージドルールグループ](https://docs.aws.amazon.com/ja_jp/waf/latest/developerguide/aws-managed-rule-groups-ip-rep.html#aws-managed-rule-groups-ip-rep-amazon)
- [匿名 IP リストマネージドルールグループ](https://docs.aws.amazon.com/ja_jp/waf/latest/developerguide/aws-managed-rule-groups-ip-rep.html#aws-managed-rule-groups-ip-rep-anonymous)
- [コアルールセット (CRS) マネージドルールグループ](https://docs.aws.amazon.com/ja_jp/waf/latest/developerguide/aws-managed-rule-groups-baseline.html#aws-managed-rule-groups-baseline-crs)
- [既知の不正な入力マネージドルールグループ](https://docs.aws.amazon.com/ja_jp/waf/latest/developerguide/aws-managed-rule-groups-baseline.html#aws-managed-rule-groups-baseline-known-bad-inputs)
- [Linux オペレーティングシステムマネージドルールグループ](https://docs.aws.amazon.com/ja_jp/waf/latest/developerguide/aws-managed-rule-groups-use-case.html#aws-managed-rule-groups-use-case-linux-os)
- [POSIX オペレーティングシステムマネージドルールグループ](https://docs.aws.amazon.com/ja_jp/waf/latest/developerguide/aws-managed-rule-groups-use-case.html#aws-managed-rule-groups-use-case-posix-os)
- [SQL データベースマネージドルールグループ](https://docs.aws.amazon.com/ja_jp/waf/latest/developerguide/aws-managed-rule-groups-use-case.html#aws-managed-rule-groups-use-case-sql-db)

{{< alert >}}
- CloudFrontリソースは`us-east-1`リージョンに配置されるため、WAFリソースも同様に`us-east-1`へ作成する必要があるのでご留意ください。
- 優先度についてはAWSマネジメントコンソールでポチポチした際に付与される優先度と同じ順序にしています。
- CloudFrontへのWAF割り当てはaws_wafv2_web_acl_associationではなく、aws_cloudfront_distributionにて実施するため今回は省略しています。
- 8KB以上のリクエストボディを扱えるように`SizeRestrictions_BODY`ルールを、他のクラウドプロバイダーからのアクセスを誤ってブロックしないように`HostingProviderIPList`ルールを除外しています。
{{< /alert >}}

```tf:例1のリソース定義
# WAFのWeb ACL設定
resource "aws_wafv2_web_acl" "cloudfront" {
  provider = aws.us_east_1

  name  = var.aws_gloal_wafv2_name
  scope = "CLOUDFRONT"

  default_action {
    allow {}
  }

  rule {
    name     = "AWS-AWSManagedRulesAmazonIpReputationList"
    priority = 0

    override_action {
      none {}
    }

    statement {
      managed_rule_group_statement {
        name        = "AWSManagedRulesAmazonIpReputationList"
        vendor_name = "AWS"
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "AWS-AWSManagedRulesAmazonIpReputationList"
      sampled_requests_enabled   = true
    }
  }

  rule {
    name     = "AWS-AWSManagedRulesAnonymousIpList"
    priority = 1

    override_action {
      none {}
    }

    statement {
      managed_rule_group_statement {
        name        = "AWSManagedRulesAnonymousIpList"
        vendor_name = "AWS"

        excluded_rule {
          name = "HostingProviderIPList"
        }
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "AWS-AWSManagedRulesAnonymousIpList"
      sampled_requests_enabled   = true
    }
  }

  rule {
    name     = "AWS-AWSManagedRulesCommonRuleSet"
    priority = 2

    override_action {
      none {}
    }

    statement {
      managed_rule_group_statement {
        name        = "AWSManagedRulesCommonRuleSet"
        vendor_name = "AWS"

        excluded_rule {
          name = "SizeRestrictions_BODY"
        }
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "AWS-AWSManagedRulesCommonRuleSet"
      sampled_requests_enabled   = true
    }
  }

  rule {
    name     = "AWS-AWSManagedRulesKnownBadInputsRuleSet"
    priority = 3

    override_action {
      none {}
    }

    statement {
      managed_rule_group_statement {
        name        = "AWSManagedRulesKnownBadInputsRuleSet"
        vendor_name = "AWS"
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "AWS-AWSManagedRulesKnownBadInputsRuleSet"
      sampled_requests_enabled   = true
    }
  }

  rule {
    name     = "AWS-AWSManagedRulesLinuxRuleSet"
    priority = 4

    override_action {
      none {}
    }

    statement {
      managed_rule_group_statement {
        name        = "AWSManagedRulesLinuxRuleSet"
        vendor_name = "AWS"
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "AWS-AWSManagedRulesLinuxRuleSet"
      sampled_requests_enabled   = true
    }
  }

  rule {
    name     = "AWS-AWSManagedRulesUnixRuleSet"
    priority = 5

    override_action {
      none {}
    }

    statement {
      managed_rule_group_statement {
        name        = "AWSManagedRulesUnixRuleSet"
        vendor_name = "AWS"
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "AWS-AWSManagedRulesUnixRuleSet"
      sampled_requests_enabled   = true
    }
  }

  rule {
    name     = "AWS-AWSManagedRulesSQLiRuleSet"
    priority = 6

    override_action {
      none {}
    }

    statement {
      managed_rule_group_statement {
        name        = "AWSManagedRulesSQLiRuleSet"
        vendor_name = "AWS"
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "AWS-AWSManagedRulesSQLiRuleSet"
      sampled_requests_enabled   = true
    }
  }

  visibility_config {
    cloudwatch_metrics_enabled = true
    metric_name                = var.aws_gloal_wafv2_name
    sampled_requests_enabled   = true
  }
}
```

### 例2. 汎用的なAPI Gateway向け定義

「[例1. 汎用的なCloudFront向け定義](#例1-汎用的なcloudfront向け定義)」と同等内容のそれっぽい汎用的なAPI Gateway向けのWAF定義です。CloudFront向けとの差異はスコープが`REGIONAL`になっている点とデフォルトのプロバイダー設定を利用している点の2点です。

```tf:例2のリソース定義
# WAFのWeb ACL設定
resource "aws_wafv2_web_acl" "apigw" {
  name  = var.aws_apigw_wafv2_name
  scope = "REGIONAL"

  default_action {
    allow {}
  }

  rule {
    name     = "AWS-AWSManagedRulesAmazonIpReputationList"
    priority = 0

    override_action {
      none {}
    }

    statement {
      managed_rule_group_statement {
        name        = "AWSManagedRulesAmazonIpReputationList"
        vendor_name = "AWS"
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "AWS-AWSManagedRulesAmazonIpReputationList"
      sampled_requests_enabled   = true
    }
  }

  rule {
    name     = "AWS-AWSManagedRulesAnonymousIpList"
    priority = 1

    override_action {
      none {}
    }

    statement {
      managed_rule_group_statement {
        name        = "AWSManagedRulesAnonymousIpList"
        vendor_name = "AWS"

        excluded_rule {
          name = "HostingProviderIPList"
        }
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "AWS-AWSManagedRulesAnonymousIpList"
      sampled_requests_enabled   = true
    }
  }

  rule {
    name     = "AWS-AWSManagedRulesCommonRuleSet"
    priority = 2

    override_action {
      none {}
    }

    statement {
      managed_rule_group_statement {
        name        = "AWSManagedRulesCommonRuleSet"
        vendor_name = "AWS"

        excluded_rule {
          name = "SizeRestrictions_BODY"
        }
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "AWS-AWSManagedRulesCommonRuleSet"
      sampled_requests_enabled   = true
    }
  }

  rule {
    name     = "AWS-AWSManagedRulesKnownBadInputsRuleSet"
    priority = 3

    override_action {
      none {}
    }

    statement {
      managed_rule_group_statement {
        name        = "AWSManagedRulesKnownBadInputsRuleSet"
        vendor_name = "AWS"
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "AWS-AWSManagedRulesKnownBadInputsRuleSet"
      sampled_requests_enabled   = true
    }
  }

  rule {
    name     = "AWS-AWSManagedRulesLinuxRuleSet"
    priority = 4

    override_action {
      none {}
    }

    statement {
      managed_rule_group_statement {
        name        = "AWSManagedRulesLinuxRuleSet"
        vendor_name = "AWS"
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "AWS-AWSManagedRulesLinuxRuleSet"
      sampled_requests_enabled   = true
    }
  }

  rule {
    name     = "AWS-AWSManagedRulesUnixRuleSet"
    priority = 5

    override_action {
      none {}
    }

    statement {
      managed_rule_group_statement {
        name        = "AWSManagedRulesUnixRuleSet"
        vendor_name = "AWS"
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "AWS-AWSManagedRulesUnixRuleSet"
      sampled_requests_enabled   = true
    }
  }

  rule {
    name     = "AWS-AWSManagedRulesSQLiRuleSet"
    priority = 6

    override_action {
      none {}
    }

    statement {
      managed_rule_group_statement {
        name        = "AWSManagedRulesSQLiRuleSet"
        vendor_name = "AWS"
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "AWS-AWSManagedRulesSQLiRuleSet"
      sampled_requests_enabled   = true
    }
  }

  visibility_config {
    cloudwatch_metrics_enabled = true
    metric_name                = var.aws_apigw_wafv2_name
    sampled_requests_enabled   = true
  }
}

# API GatewayへのWAFの割り当て
resource "aws_wafv2_web_acl_association" "apigw" {
  resource_arn = var.aws_apigw_arn  # or aws_api_gateway_stage.*.arn
  web_acl_arn  = aws_wafv2_web_acl.wafv2_web_acl_regional.arn
}
```

### 例3. IPホワイトリストを用いたアクセス制御

許可したIPアドレスからのみアクセスを許可するホワイトリストベースのアクセス制御を用いた例です。デフォルトアクションはブロックにして、ホワイトリストに合致した場合のみ許可するようにしています。

```tf:例3のリソース定義
# WAFのWeb ACL設定
resource "aws_wafv2_web_acl" "cloudfront" {
  provider = aws.us_east_1

  name  = var.aws_cloudfront_wafv2_name
  scope = "CLOUDFRONT"

  default_action {
    block {}
  }

  rule {
    name     = "AllowedIPAddressList"
    priority = 0

    action {
      allow {}
    }

    statement {
      ip_set_reference_statement {
        arn = aws_wafv2_ip_set.cloudfront.arn
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = false
      metric_name                = "AllowedIPAddressList"
      sampled_requests_enabled   = true
    }
  }

  visibility_config {
    cloudwatch_metrics_enabled = true
    metric_name                = var.aws_cloudfront_wafv2_name
    sampled_requests_enabled   = true
  }
}

# WAFのIP sets設定
resource "aws_wafv2_ip_set" "cloudfront" {
  provider = aws.us_east_1

  name               = var.aws_wafv2_ipset_name
  description        = var.aws_wafv2_ipset_description
  addresses          = var.aws_wafv2_ipset_addresses
  scope              = "CLOUDFRONT"
  ip_address_version = "IPV4"
}
```

### 例4. カスタムヘッダーを用いたアクセス制御

CloudFrontなどで付与した特定のカスタムヘッダーが含まれていた場合にのみアクセスを許可するといったケースの例です。なお、カスタムヘッダー名と設定値はAWS Systems Manager Parameter Storeへ事前に格納しておき、Terraform実行時にdataブロックを用いて値を取得してくるようにしています。

```tf:例4のリソース定義
# WAFのWeb ACL設定
resource "aws_wafv2_web_acl" "apigw" {
  name  = var.aws_apigw_wafv2_name
  scope = "REGIONAL"

  default_action {
    block {}
  }

  rule {
    name     = "AllowedCustomHeader"
    priority = 0

    action {
      allow {}
    }

    statement {
      byte_match_statement {
        field_to_match {
          single_header {
            name = data.aws_ssm_parameter.custom_header_name.value
          }
        }

        positional_constraint = "CONTAINS"
        search_string         = data.aws_ssm_parameter.custom_header_value.value
        text_transformation {
          priority = 0
          type     = "NONE"
        }
      }
    }

  visibility_config {
    cloudwatch_metrics_enabled = true
    metric_name                = var.aws_apigw_wafv2_name
    sampled_requests_enabled   = true
  }
}

# SSMパラメータストアからのデータ取得
data "aws_ssm_parameter" "custom_header_name" {
  name = var.aws_ssmparam_custom_header_name
}

data "aws_ssm_parameter" "custom_header_value" {
  name = var.aws_ssmparam_custom_header_value
}
```

### 例5. CloudWatch Logsを活用したロギング設定

WAFログをCloudWatch Logsへ出力する際の例です。WAFログのロググループ名は"aws-waf-logs-*"とする必要があること、CloudFront向けWAFについては`us-east-1`にすることを忘れがちなのでご注意ください。

https://docs.aws.amazon.com/ja_jp/waf/latest/developerguide/logging-cw-logs.html#logging-cw-logs-naming

```tf:例5のリソース定義(CloudFrontの場合)
# CloudFront向けWAFログ設定
resource "aws_wafv2_web_acl_logging_configuration" "cloudfront" {
 provider = aws.us_east_1

 log_destination_configs = [aws_cloudwatch_log_group.cloudfront_waf.arn]
 resource_arn            = aws_wafv2_web_acl.cloudfront.arn
}

# CloudFront向けWAFログ用のロググループ設定
resource "aws_cloudwatch_log_group" "cloudfront_waf" {
  provider = aws.us_east_1

  name = "aws-waf-logs-${var.aws_cloudfront_wafv2_name}"
}
```

```tf:例5のリソース定義(APIGatewayの場合)
# API Gateway向けWAFログ設定
resource "aws_wafv2_web_acl_logging_configuration" "apigw" {
 log_destination_configs = [aws_cloudwatch_log_group.apigw_waf.arn]
 resource_arn            = aws_wafv2_web_acl.apigw.arn
}

# API Gateway向けWAFログ用のロググループ設定
resource "aws_cloudwatch_log_group" "apigw_waf" {
  name = "aws-waf-logs-${var.aws_apigw_wafv2_name}"
}
```

### 例6. S3バケットを活用したロギング設定

WAFログをS3バケットへ出力する際の例です。WAFログを格納するS3バケット名は"aws-waf-logs-*"とする必要があることを忘れがちなのでご注意ください。また、CloudFront向けWAFのログ用S3バケットは、他のリソースとは異なりデフォルト側のプロバイダー設定を利用するのでこちらもご注意ください。

https://docs.aws.amazon.com/ja_jp/waf/latest/developerguide/logging-s3.html#logging-s3-naming

```tf:例6のリソース定義(CloudFrontの場合)
# CloudFront向けWAFログ設定
resource "aws_wafv2_web_acl_logging_configuration" "cloudfront" {
 provider = aws.us_east_1

 log_destination_configs = [aws_s3_bucket.cloudfront_waf_log.arn]
 resource_arn            = aws_wafv2_web_acl.cloudfront.arn
}

# CloudFront向けWAFログ用のログバケット設定（詳細は割愛）
resource "aws_s3_bucket" "cloudfront_waf_log" {
  bucket = "aws-waf-logs-${var.aws_cloudfront_wafv2_name}"

  # 以下省略
}
```

```tf:例6のリソース定義(APIGatewayの場合)
# API Gateway向けWAFログ設定
resource "aws_wafv2_web_acl_logging_configuration" "apigw" {
 log_destination_configs = [aws_s3_bucket.apigw_waf_log.arn]
 resource_arn            = aws_wafv2_web_acl.apigw.arn
}

# API Gateway向けWAFログ用のログバケット設定（詳細は割愛）
resource "aws_s3_bucket" "apigw_waf_log" {
  bucket = "aws-waf-logs-${var.aws_apigw_wafv2_name}"

  # 以下省略
}
```

<!-- omit in toc -->
## 終わりに

今回はTerraformの入門ということで、CloudFrontおよびAPI Gatewayに割り当てるAWS WAFのサンプルコードをいくつかご紹介してきましたがいかがだったでしょうか。こんな記事でも誰かの役に立っていただけるのであれば幸いです。

なお、今回ご紹介したコードはあくまでサンプルであり、動作を保証するものではございません。そのまま使用したことによって発生したトラブルなどについては一切責任を負うことはできませんのでご注意ください。

---

- AWS は、米国その他の諸国における Amazon.com, Inc. またはその関連会社の商標です。
- Terraform は、HashiCorp, Inc. の米国およびその他の国における商標または登録商標です。
- その他、記載されている会社名および商品・製品・サービス名は、各社の商標または登録商標です。

