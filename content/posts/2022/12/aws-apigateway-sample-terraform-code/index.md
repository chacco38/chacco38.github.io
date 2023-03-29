---
title: "TerraformでAmazon API Gatewayを構築するサンプルコードを書いてみた"
date: 2022-12-29T00:00:00+09:00
lastmod: null
tags: ["Terraform", "AWS", "Amazon API Gateway"]
draft: false
externalUrl: null
---

みなさん、こんにちは。今回はTerraformの入門ということでAmazon API Gatewayのサンプルコードを書いてみましたのでこちらを紹介していきたいと思います。

なお、サンプルコードを書いた際のTerraformおよびAWSプロバイダーのバージョンは次のとおりです。最新バージョンでは定義方法が異なっている可能性があるため、実際にコードを書く際は最新の「[Terraformドキュメント]」と「[AWSプロバイダードキュメント]」を確認しながら開発を進めていただければと思います。

[Terraformドキュメント]: https://developer.hashicorp.com/terraform/docs
[AWSプロバイダードキュメント]: https://registry.terraform.io/providers/hashicorp/aws/latest/docs

**例）versions.tf**

```tf:versions.tf
# Requirements
terraform {
  required_version = "~> 1.3.6"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 4.46.0"
    }
  }
}
```

<!-- omit in toc -->
## サンプルコードを書いてみた

今回は次のような構成のサンプルコードを書いてみました。なお、変数定義部分などの一部省略している点、ならびにステップごとの細かい説明などは省いていますのでご承知おきください。詳細については「[AWSプロバイダードキュメント]」をご参照ください。

- [例1. リージョナルREST APIゲートウェイの作成](#例1-リージョナルrest-apiゲートウェイの作成)
- [例2. プライベートREST APIゲートウェイの作成](#例2-プライベートrest-apiゲートウェイの作成)

ちなみに、すべてのサンプルコードに共通してプロバイダー定義は次のようにしています。

**例）providers.tf**

```tf:providers.tf
# デフォルトのプロバイダー設定
provider "aws" {
  region = var.aws_region
  default_tags {
    tags = {
      Owner     = "matt"
      Terraform = "true"
    }
  }
}
```

### 例1. リージョナルREST APIゲートウェイの作成

Cognitoオーソライザを使って認証を行うリージョナルREST APIゲートウェイの例で、バックエンドのサービスとの接続にはVPCリンクを経由するようにしています。今回はOpenAPI定義もTerraformでデプロイさせていますが、ここはTerraform管理下から切り離しても良い部分かもしれません。

**例）main.tf**

```tf:main.tf
# リージョナルREST APIゲートウェイの定義
resource "aws_api_gateway_rest_api" "regional" {
  name = var.regional_api_gateway_name

  body = templatefile("${path.module}/regional_openapi.yaml", {
    vpc_link          = aws_api_gateway_vpc_link.reginal.id,
    cognito_user_pool = aws_cognito_user_pool.this.arn,
    api_gateway_role  = aws_iam_role.regional_api_gateway.arn,
    backend_dns_name  = aws_lb.backend.dns_name,
  })

  endpoint_configuration {
    types = ["REGIONAL"]
  }

  tags = {
    Name = var.regional_api_gateway_name
  }
}

# リージョナルREST APIのデプロイ定義
resource "aws_api_gateway_deployment" "regional" {
  rest_api_id = aws_api_gateway_rest_api.regional.id

  triggers = {
    redeployment = sha1(templatefile("${path.module}/regional_openapi.yaml", {
      vpc_link          = aws_api_gateway_vpc_link.reginal.id,
      cognito_user_pool = aws_cognito_user_pool.this.arn,
      api_gateway_role  = aws_iam_role.regional_api_gateway.arn,
      backend_dns_name  = aws_lb.backend.dns_name,
    }))
  }

  lifecycle {
    create_before_destroy = true
  }
}

# リージョナルREST APIのステージ定義
resource "aws_api_gateway_stage" "regional" {
  deployment_id = aws_api_gateway_deployment.regional.id
  rest_api_id   = aws_api_gateway_rest_api.regional.id
  stage_name    = var.regional_api_gateway_stage_name

  access_log_settings {
    destination_arn = aws_cloudwatch_log_group.regional_apigateway_log.arn
    format          = "{\"requestId\":\"$context.requestId\", \"extendedRequestId\":\"$context.extendedRequestId\", \"ip\": \"$context.identity.sourceIp\", \"caller\":\"$context.identity.caller\", \"user\":\"$context.identity.user\", \"requestTime\":\"$context.requestTime\", \"httpMethod\":\"$context.httpMethod\", \"resourcePath\":\"$context.resourcePath\", \"status\":\"$context.status\", \"protocol\":\"$context.protocol\", \"responseLength\":\"$context.responseLength\"}"
  }
}

# リージョナルREST APIのメソッド設定定義
resource "aws_api_gateway_method_settings" "regional" {
  rest_api_id = aws_api_gateway_rest_api.regional.id
  stage_name  = aws_api_gateway_stage.regional.stage_name
  method_path = "*/*"

  settings {
    metrics_enabled    = true
    logging_level      = "INFO"
    data_trace_enabled = true
  }
}

# リージョナルREST APIのバックエンドサービス接続用VPCリンク定義
resource "aws_api_gateway_vpc_link" "regional" {
  name        = var.vpc_link_name
  target_arns = [aws_lb.backend.arn]

  tags = {
    Name = var.vpc_link_name
  }
}

# モニタリング用IAMロールの接続
resource "aws_api_gateway_account" "this" {
  cloudwatch_role_arn = aws_iam_role.cloudwatch.arn
}

# モニタリング用IAMロールの定義
resource "aws_iam_role" "cloudwatch" {
  name = var.iam_cloudwatch_role_name

  assume_role_policy = jsonencode({
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Principal": {
          "Service": "apigateway.amazonaws.com"
        },
        "Action": "sts:AssumeRole"
      }
    ]
  })

  tags = {
    Name = var.iam_cloudwatch_role_name
  }

  lifecycle {
    create_before_destroy = true
  }
}

# モニタリング用IAMロールへのポリシー割り当て
resource "aws_iam_role_policy_attachment" "cloudwatch" {
  role       = aws_iam_role.cloudwatch.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonAPIGatewayPushToCloudWatchLogs"
}

# リージョナルREST API用IAMロールの定義
resource "aws_iam_role" "regional_api_gateway" {
  name = var.iam_regional_api_gateway_role_name

  assume_role_policy = jsonencode({
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Principal": {
          "Service": "apigateway.amazonaws.com"
        },
        "Action": "sts:AssumeRole"
      }
    ]
  })

  tags = {
    Name = var.iam_regional_api_gateway_role_name
  }

  lifecycle {
    create_before_destroy = true
  }
}

# リージョナルREST API用IAMポリシーの定義
resource "aws_iam_policy" "regional_api_gateway" {
  name = var.iam_regional_api_gateway_policy_name
  path = "/"

  policy = jsonencode({
    "Version": "2012-10-17",
    "Statement": [
      {
        "Action": [
          "apigateway:*"
        ],
        "Effect": "Allow",
        "Resource": [
          "arn:aws:apigateway:*::/restapis/*/authorizers"
        ],
        "Condition": {
          "ArnLike": {
            "apigateway:CognitoUserPoolProviderArn": [
                "${aws_cognito_user_pool.this.arn}"
            ]
          }
        }
      }
    ]
  })

  tags = {
    Name = var.iam_regional_api_gateway_policy_name
  }

  lifecycle {
    create_before_destroy = true
  }
}

# リージョナルREST API用IAMロールへのポリシー割り当て
resource "aws_iam_role_policy_attachment" "regional_api_gateway" {
  policy_arn = aws_iam_policy.regional_api_gateway.arn
  role       = aws_iam_role.regional_api_gateway.name
}
```

### 例2. プライベートREST APIゲートウェイの作成

特定のVPCエンドポイント経由のアクセスのみを許可するプライベートREST APIゲートウェイの例です。今回はOpenAPI定義もTerraformでデプロイさせていますが、ここはTerraform管理下から切り離しても良い部分かもしれません。

**例）main.tf**

```tf:main.tf
# プライベートREST APIゲートウェイの定義
resource "aws_api_gateway_rest_api" "private" {
  name = var.private_api_gateway_name

  body = templatefile("${path.module}/private_openapi.yaml", {
    api_gateway_role  = aws_iam_role.private_api_gateway.arn
  })

  endpoint_configuration {
    types            = ["PRIVATE"]
    vpc_endpoint_ids = [aws_vpc_endpoint.execute_api.id]
  }

  tags = {
    Name = var.private_api_gateway_name
  }
}

# プライベートREST APIゲートウェイのポリシー定義 (特定のVPCエンドポイントのみ許可)
resource "aws_api_gateway_rest_api_policy" "private" {
  rest_api_id = aws_api_gateway_rest_api.private.id

  policy = jsonencode({
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Principal": "*",
        "Action": "execute-api:Invoke",
        "Resource": "${aws_api_gateway_rest_api.private.execution_arn}/*"
      },
      {
        "Effect": "Deny",
        "Principal": "*",
        "Action": "execute-api:Invoke",
        "Resource": "${aws_api_gateway_rest_api.private.execution_arn}/*",
        "Condition": {
          "StringNotEquals": {
            "aws:SourceVpce": "${aws_vpc_endpoint.execute_api.id}"
          }
        }
      }
    ]
  })
}

# プライベートREST APIのデプロイ定義
resource "aws_api_gateway_deployment" "private" {
  rest_api_id = aws_api_gateway_rest_api.private.id

  triggers = {
    redeployment = sha1(templatefile("${path.module}/private_openapi.yaml", {
      api_gateway_role  = aws_iam_role.private_api_gateway.arn
    }))
  }

  lifecycle {
    create_before_destroy = true
  }
}

# プライベートREST APIのステージ定義
resource "aws_api_gateway_stage" "private" {
  deployment_id = aws_api_gateway_deployment.private.id
  rest_api_id   = aws_api_gateway_rest_api.private.id
  stage_name    = var.private_api_gateway_stage_name

  access_log_settings {
    destination_arn = aws_cloudwatch_log_group.private_apigateway_log.arn
    format          = "{\"requestId\":\"$context.requestId\", \"extendedRequestId\":\"$context.extendedRequestId\", \"ip\": \"$context.identity.sourceIp\", \"caller\":\"$context.identity.caller\", \"user\":\"$context.identity.user\", \"requestTime\":\"$context.requestTime\", \"httpMethod\":\"$context.httpMethod\", \"resourcePath\":\"$context.resourcePath\", \"status\":\"$context.status\", \"protocol\":\"$context.protocol\", \"responseLength\":\"$context.responseLength\"}"
  }
}

# プライベートREST APIのメソッド設定定義
resource "aws_api_gateway_method_settings" "private" {
  rest_api_id = aws_api_gateway_rest_api.private.id
  stage_name  = aws_api_gateway_stage.private.stage_name
  method_path = "*/*"

  settings {
    metrics_enabled    = true
    logging_level      = "INFO"
    data_trace_enabled = true
  }
}

# モニタリング用IAMロールの接続
resource "aws_api_gateway_account" "this" {
  cloudwatch_role_arn = aws_iam_role.cloudwatch.arn
}

# モニタリング用IAMロールの定義
resource "aws_iam_role" "cloudwatch" {
  name = var.iam_cloudwatch_role_name

  assume_role_policy = jsonencode({
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Principal": {
          "Service": "apigateway.amazonaws.com"
        },
        "Action": "sts:AssumeRole"
      }
    ]
  })

  tags = {
    Name = var.iam_cloudwatch_role_name
  }

  lifecycle {
    create_before_destroy = true
  }
}

# モニタリング用IAMロールへのポリシー割り当て
resource "aws_iam_role_policy_attachment" "cloudwatch" {
  role       = aws_iam_role.cloudwatch.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonAPIGatewayPushToCloudWatchLogs"
}

# プライベートREST API用IAMロールの定義
resource "aws_iam_role" "private_api_gateway" {
  name = var.iam_private_api_gateway_role_name

  assume_role_policy = jsonencode({
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Principal": {
          "Service": "apigateway.amazonaws.com"
        },
        "Action": "sts:AssumeRole"
      }
    ]
  })

  tags = {
    Name = var.iam_private_api_gateway_role_name
  }

  lifecycle {
    create_before_destroy = true
  }
}

# プライベートREST API用IAMポリシーの定義
resource "aws_iam_policy" "private_api_gateway" {
  name = var.iam_private_api_gateway_policy_name
  path = "/"

  policy = jsonencode({
    # 割愛
  })

  tags = {
    Name = var.iam_private_api_gateway_policy_name
  }

  lifecycle {
    create_before_destroy = true
  }
}

# プライベートREST API用IAMロールへのポリシー割り当て
resource "aws_iam_role_policy_attachment" "private_api_gateway" {
  policy_arn = aws_iam_policy.private_api_gateway.arn
  role       = aws_iam_role.private_api_gateway.name
}
```

<!-- omit in toc -->
## 終わりに

今回はTerraformの入門ということで、Amazon API Gatewayのサンプルコードをいくつかご紹介してきましたがいかがだったでしょうか。こんな記事でも誰かの役に立っていただけるのであれば幸いです。

なお、今回ご紹介したコードはあくまでサンプルであり、動作を保証するものではございません。そのまま使用したことによって発生したトラブルなどについては一切責任を負うことはできませんのでご注意ください。

---

- AWS は、米国その他の諸国における Amazon.com, Inc. またはその関連会社の商標です。
- Terraform は、HashiCorp, Inc. の米国およびその他の国における商標または登録商標です。
- その他、記載されている会社名および商品・製品・サービス名は、各社の商標または登録商標です。
