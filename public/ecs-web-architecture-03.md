---
title: 【AWS】ECSを用いた簡易web構築③(VPC・ネットワーク構成)
tags:
  - 'AWS'
  - 'VPC'
  - 'Terraform'
private: true
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

# 第3章：VPC・ネットワーク構成

本章では、本シリーズで利用する **VPC およびネットワーク関連リソース** について解説します。
後続章で構築する ECS / ALB / 各種 AWS サービスが安全かつ安定して動作するための **基盤部分** となる章です。


## 📚 目次

* [1. 構成の説明](#1-構成の説明)
* [2. 作成方法](#2-作成方法)
* [3. Terraform構成](#3-terraform構成)
* [4. 振り返り](#4-振り返り)
* [5. 参照](#5-参照)
* [6. コード全体](#6-コード全体)

---

## 1. 構成の説明

### 何をしたいか

* ECS（Fargate）で動作するバックエンドを **プライベートネットワーク内に配置**
* インターネット通信を最小限に抑えつつ、必要な AWS サービスへアクセス可能にする
* フロントエンド（S3）や外部ユーザーからのアクセス経路を明確に分離する

---

### どんな構成か・どんなことができるか

本章で構築するネットワーク構成は以下の通りです。

* VPC
* Public / Private サブネット（複数 AZ）
* Internet Gateway（Public 用）
* Gateway VPC Endpoint（S3 / DynamoDB）
* Interface VPC Endpoint（ECS / ECR / CloudWatch Logs）
* ALB / ECS / VPC Endpoint 用の Security Group

![構成図](../images/ECS-web-architecture/ECS-DynamoDB-S3-CICD-arc.jpg)

---

### この構成を採用した意図

* **コスト削減**

  * NAT Gateway を使用せず、VPC Endpoint を活用
* **セキュリティ向上**

  * ECS タスクはインターネット非接続
  * 通信は VPC 内、または AWS 内部ネットワークに限定


---

## 2. 作成方法

### 前提条件

* AWS アカウント
* Terraform 実行環境
* 基本的な VPC / サブネットの理解

---

### 構築順序（概要）

1. VPC 作成
2. Public / Private サブネット作成
3. Internet Gateway 作成
4. Route Table 作成・関連付け
5. VPC Endpoint（Gateway / Interface）作成
6. Security Group 作成

---

### 主要な設定ポイント

#### サブネット設計

* **Public サブネット**

  * ALB 配置用
  * Internet Gateway 経由で外部通信可能
* **Private サブネット**

  * ECS タスク配置用
  * インターネットルートなし

---

#### NAT Gateway を使わない構成

* Private サブネットの `0.0.0.0/0` ルートは未設定
* S3 / DynamoDB への通信は **Gateway VPC Endpoint** を使用
* ECS / ECR / Logs は **Interface VPC Endpoint** を使用

これにより、

* インターネット通信の削減
* NAT Gateway コストの削減

を実現しています。

---

#### セキュリティグループ設計

* ALB：特定 IP からの HTTP のみ許可(学習用途で自分のIPのみに制限)
* ECS：ALB からの通信のみ許可
* VPC Endpoint：VPC 内からの HTTPS のみ許可

通信経路を **明示的に制御** する設計としています。

---

## 3. Terraform構成

### フォルダ構成（例）

```text
terraform/
└── modules/
    ├── vpc/
    │   ├── main.tf
    │   ├── variables.tf
    │   └── outputs.tf
    ├── vpc_endpoint/
    │   ├── main.tf
    │   ├── variables.tf
    │   └── outputs.tf
    └── sg/
        ├── main.tf
        ├── variables.tf
        └── outputs.tf
```

---

### 主要なリソースと解説

#### VPC

```hcl
resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr
  enable_dns_support   = var.enable_dns_support
  enable_dns_hostnames = var.enable_dns_hostnames
}
```

* DNS 機能を有効化し、ECS / VPC Endpoint での名前解決を可能に ※詳細は後述。

---

#### サブネット
```hcl
resource "aws_subnet" "public" {
  for_each          = var.public_subnets
  vpc_id            = aws_vpc.main.id
  cidr_block        = each.value.cidr
  availability_zone = each.value.az
}

resource "aws_subnet" "private" {
  for_each          = var.private_subnets
  vpc_id            = aws_vpc.main.id
  cidr_block        = each.value.cidr
  availability_zone = each.value.az
}
```

* `for_each` を使用し、AZ ごとに柔軟に定義
* Public / Private を明確に分離

---

#### Route Table

* Public：Internet Gateway へのデフォルトルートあり
```hcl
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id
}

resource "aws_route" "public_igw" { #ルートテーブル作成
  route_table_id         = aws_route_table.public.id
  destination_cidr_block = "0.0.0.0/0" #VPC内部（10.0.0.0/16など）以外の通信先の指定。  
  gateway_id             = aws_internet_gateway.main.id #通信先にインターネットゲートウェイを指定。
}

resource "aws_route_table_association" "public" { #ルートテーブルをサブネットに紐づけ
  for_each       = aws_subnet.public
  subnet_id      = each.value.id #紐付け対象のサブネットID。
  route_table_id = aws_route_table.public.id #紐づけ対象のルートテーブルのID。

}
```

* Private：インターネットルートなし
```hcl
resource "aws_route_table" "private" {
  vpc_id = aws_vpc.main.id
}

resource "aws_route_table_association" "private" {
  for_each       = aws_subnet.private
  subnet_id      = each.value.id
  route_table_id = aws_route_table.private.id
}
```
---

#### VPC Endpoint

* **Gateway**

  * S3
  * DynamoDB

```hcl
resource "aws_vpc_endpoint" "s3_gateway" {
  vpc_id            = aws_vpc.main.id #vpcを指定
  service_name      = "com.amazonaws.${var.region}.s3" #サービスを指定
  vpc_endpoint_type = "Gateway" #ゲートウェイ型を指定
  route_table_ids   = [aws_route_table.private.id] #ルートテーブルを指定
}
```

* **Interface**

  * ECS
  * ECR（API / DKR）
  * CloudWatch Logs

```hcl
  resource "aws_vpc_endpoint" "ecr_api" {
  vpc_id              = var.vpc_id #vpcを指定
  service_name        = "com.amazonaws.${var.region}.ecr.api" #サービスを指定
  vpc_endpoint_type   = "Interface" #インターフェース型を指定
  subnet_ids          = var.private_subnet_ids #サブネットを指定
  security_group_ids  = [var.vpce_sg_id] #セキュリティグループを指定
  private_dns_enabled = true
}
```
**private_dns_enabled = trueについて**  
ECRからイメージを取得しようとする際、通常は https://api.ecr.ap-northeast-1.amazonaws.com というURLにアクセスします。  
true（有効）にすると、VPC内部で「ECRのURL」を名前解決した際、パブリックIPではなくVPCエンドポイントのプライベートIPアドレスを回答してくれるようになります。  
これにより、ECSなどの設定を一切変えずにPrivate サブネットから AWS サービスへ **インターネットを経由せず通信可能** になります。  
この private_dns_enabled = true が機能するためには、VPC自体の設定で以下の2つが true になっている必要があります。
```hcl
enable_dns_support = true
enable_dns_hostnames = true
```

---

#### Security Group
| 対象リソース | 許可ポート | 送信元（Source） | 役割・目的 |
| --- | --- | --- | --- |
| **ALB** | `HTTP (80)` | 特定のIPアドレス`(var.alb_ingress_cidr)` | 学習用途のため自分のIPからのアクセスのみ許可 |
| **ECSタスク** | `TCP (3000)` | **ALBのセキュリティグループ** | ALBを経由した通信のみをアプリケーションに許可 |
| **VPCエンドポイント** | `HTTPS (443)` | VPCのCIDR`(var.vpc_cidr)` | VPC内部（ECS等）からのAWSサービスへの通信を許可 |

* ALB / ECS / VPC Endpoint ごとに分離
* ポート・通信元を最小限に制限

---

## 4. 振り返り

はじめはNAT Gatewayを用いて構成していましたが、VPCエンドポイントで代替できることに気づき、安価に今回の環境を構築できるようになりました。

コンテナイメージを作成する際に外部との通信が必要なのでNAT Gatewayが必要と思い込んでいましたが、イメージをビルド・プッシュするのはローカルやGitHub Actions内の仮想サーバーなので、実行環境であるECSが直接外部と通信する必要はありませんでした。

※アプリが実行中にインターネット上の外部APIを呼ぶ必要がある場合はNAT Gatewayが必要。

---

### 次にやること

次章では、
[バックエンド（Node.js）を Docker 化し、ECS で実行する準備]()について解説します。

* **[①構成の説明]()**  
* **[②S3(静的Webサイトホスティング)/フロントエンド]()**  
* **[③VPC関連]()**  
* **[④Docker/バックエンド]()**  
* **[⑤ECS + ALB]()**  
* **[⑥DynamoDB・S3連携]()**  
* **[⑦CI/CD(アプリケーション)]()** 
* **[⑧CI/CD(インフラ)]()**

---

## 5. 参照

* [AWS公式ドキュメント（VPC）](https://docs.aws.amazon.com/ja_jp/vpc/?icmpid=docs_homepage_featuredsvcs)
* [Terraform AWS Provider ドキュメント](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)

---

## 6. コード全体
※ファイル数が多いためvariables, outputsは省略  
terraform/modules/vpc/main.tf
```hcl
############################
# VPC
############################
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_support   = var.enable_dns_support
  enable_dns_hostnames = var.enable_dns_hostnames

  tags = {
    Name        = var.vpc_name
    Environment = var.environment
  }
}

############################
# Subnet
############################
resource "aws_subnet" "public" {
  for_each          = var.public_subnets
  vpc_id            = aws_vpc.main.id
  cidr_block        = each.value.cidr
  availability_zone = each.value.az

  tags = {
    Name        = each.key
    Environment = var.environment
  }
}

resource "aws_subnet" "private" {
  for_each          = var.private_subnets
  vpc_id            = aws_vpc.main.id
  cidr_block        = each.value.cidr
  availability_zone = each.value.az

  tags = {
    Name        = each.key
    Environment = var.environment
  }
}

############################
# Internet Gateway（Public 用）
############################
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name        = var.igw_name
    Environment = var.environment
  }
}

############################
# Route Table（Public）
############################
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name        = var.public_route_table_name
    Environment = var.environment
  }
}

resource "aws_route" "public_igw" {
  route_table_id         = aws_route_table.public.id
  destination_cidr_block = "0.0.0.0/0"
  gateway_id             = aws_internet_gateway.main.id
}

resource "aws_route_table_association" "public" {
  for_each       = aws_subnet.public
  subnet_id      = each.value.id
  route_table_id = aws_route_table.public.id
}

############################
# Route Table（Private）
# ※ インターネットルートなし
############################
resource "aws_route_table" "private" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name        = var.private_route_table_name
    Environment = var.environment
  }
}

resource "aws_route_table_association" "private" {
  for_each       = aws_subnet.private
  subnet_id      = each.value.id
  route_table_id = aws_route_table.private.id
}

############################
# Gateway Endpoint（S3）
############################
resource "aws_vpc_endpoint" "s3_gateway" {
  vpc_id            = aws_vpc.main.id
  service_name      = "com.amazonaws.${var.region}.s3"
  vpc_endpoint_type = "Gateway"
  route_table_ids   = [aws_route_table.private.id]

  tags = {
    Name        = var.s3_gateway_endpoint_name
    Environment = var.environment
  }
}

############################
# Gateway Endpoint（DynamoDB）
############################
resource "aws_vpc_endpoint" "dynamodb_gateway" {
  vpc_id            = aws_vpc.main.id
  service_name      = "com.amazonaws.${var.region}.dynamodb"
  vpc_endpoint_type = "Gateway"
  route_table_ids   = [aws_route_table.private.id]

  tags = {
    Name        = "${var.environment}-net-dynamodb-gateway-01"
    Environment = var.environment
  }
}
```
terraform/modules/vpc_endpoint/main.tf
```hcl
############################################################
# Interface Endpoint
############################################################

resource "aws_vpc_endpoint" "logs" {
  vpc_id              = var.vpc_id
  service_name        = "com.amazonaws.${var.region}.logs"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = var.private_subnet_ids
  security_group_ids  = [var.vpce_sg_id]
  private_dns_enabled = true
}

resource "aws_vpc_endpoint" "ecr_api" {
  vpc_id              = var.vpc_id
  service_name        = "com.amazonaws.${var.region}.ecr.api"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = var.private_subnet_ids
  security_group_ids  = [var.vpce_sg_id]
  private_dns_enabled = true
}

resource "aws_vpc_endpoint" "ecr_dkr" {
  vpc_id              = var.vpc_id
  service_name        = "com.amazonaws.${var.region}.ecr.dkr"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = var.private_subnet_ids
  security_group_ids  = [var.vpce_sg_id]
  private_dns_enabled = true
}

resource "aws_vpc_endpoint" "ecs" {
  vpc_id              = var.vpc_id
  service_name        = "com.amazonaws.${var.region}.ecs"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = var.private_subnet_ids
  security_group_ids  = [var.vpce_sg_id]
  private_dns_enabled = true
}

```
terraform/modules/sg/main.tf
```hcl
############################
# ALB Security Group
############################
resource "aws_security_group" "alb_sg" {
  vpc_id      = var.vpc_id
  name        = var.alb_sg_name
  description = "Security group for ALB"

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = [var.alb_ingress_cidr]
    description = "Allow HTTP access from specific IP"
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name        = var.alb_sg_name
    Environment = var.environment
  }
}

############################
# ECS Security Group
############################
resource "aws_security_group" "ecs_sg" {
  vpc_id      = var.vpc_id
  name        = var.ecs_sg_name
  description = "Security group for ECS tasks"

  ingress {
    from_port       = 3000
    to_port         = 3000
    protocol        = "tcp"
    security_groups = [aws_security_group.alb_sg.id]
    description     = "Allow app access from ALB"
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name        = var.ecs_sg_name
    Environment = var.environment
  }
}

############################
# VPC Endpoint 用 Security Group
############################
resource "aws_security_group" "vpce_sg" {
  vpc_id      = var.vpc_id
  name        = "${var.environment}-vpce-sg"
  description = "Security group for VPC Interface Endpoints"

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = [var.vpc_cidr]
    description = "Allow HTTPS from VPC"
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name        = "${var.environment}-vpce-sg"
    Environment = var.environment
  }
}

```