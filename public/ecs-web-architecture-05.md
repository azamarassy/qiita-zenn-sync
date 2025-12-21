---
title: 【AWS】ECSを用いた簡易web構築⑤(ECS・ALB)
tags:
  - 'AWS'
  - 'ECS'
  - 'Terraform'
private: true
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

# 第5章：ECS / ALB（バックエンドの実行と公開）

本章では、前章で Docker 化したバックエンドアプリケーションを
**ECS（Fargate）上で実行し、ALB（Application Load Balancer）経由で外部公開する構成** について解説します。

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

* Docker イメージ化したバックエンド API を ECS で実行
* 外部から HTTP リクエストを受け付けられるようにする
* ECS タスクを直接公開せず、**ALB を経由してアクセス** させる
* DynamoDB / S3 へのアクセスは ECS タスクロール経由で実施

---

### どんな構成か・何ができるか

本章で構築する構成は以下の通りです。

* **ECR**
  バックエンドの Docker イメージを格納

* **ECS（Fargate）**

  * ECS クラスター
  * タスク定義
  * ECS サービス

* **ALB**

  * インターネット向け ALB
  * リスナー（HTTP）
  * ターゲットグループ（ECS タスク）

ALB が受け取ったリクエストは ECS サービスに紐づくタスクへ転送され、
バックエンド API がレスポンスを返却します。

![構成図](../images/ECS-web-architecture/ECS-DynamoDB-S3-CICD-arc.jpg)

---

## 2. 作成方法（概要）

### 前提条件

* AWS アカウント
* Terraform 実行環境
* VPC / サブネット / セキュリティグループが作成済み
  ※ 本シリーズの前章で構築済みである前提

---

### 構築順序（概要）

1. ECR リポジトリ作成
2. ECS クラスター作成
3. タスク定義作成
4. ALB・ターゲットグループ作成
5. ECS サービス作成（ALB と紐付け）

---

### 主な設定方針・注意点

* **ネットワークモードは `awsvpc`**

  * Fargate を使う場合に必須
* **ECS タスクはプライベートサブネット**

  * 外部公開は ALB のみ
* **IAM ロールで AWS リソースにアクセス**

  * アクセスキーは使用しない

---

## 3. Terraform 構成

### フォルダ構成

```text
terraform/
└── modules/
    ├── alb/
    │   ├── main.tf
    │   ├── variables.tf
    │   └── outputs.tf
    ├── ecs/
    │   ├── main.tf
    │   ├── variables.tf
    │   └── outputs.tf
    └── ecr/
        ├── main.tf
        ├── variables.tf
        └── outputs.tf
```


---

### 主要リソースと役割

#### ECR リポジトリ

```hcl
resource "aws_ecr_repository" "api_ecr" {
  name = var.ecr_name
  image_scanning_configuration {
    scan_on_push = true
  }
}
```

* バックエンドの Docker イメージ格納先
* push 時のイメージスキャンを有効化し、中身に脆弱性がないか自動で検査
---

#### ECS クラスター

```hcl
resource "aws_ecs_cluster" "main" {
  name = var.ecs_cluster_name
}
```

---

#### ECS タスク定義

```hcl
resource "aws_ecs_task_definition" "api_task" {
  requires_compatibilities = ["FARGATE"] #FARGATEを指定
  network_mode             = "awsvpc"
}
```

* タスクの設計が、Fargateの制約や規格を守っていることを保証・制限する設定
* コンテナ1つ1つに、VPC内の固有のIPアドレスを直接割り当て

---

#### ECS サービス

```hcl
resource "aws_ecs_service" "api_service" {
  desired_count = var.desired_task_count #常に起動させるコンテナの数

  launch_type     = "FARGATE" #Fargateでコンテナを作成
  platform_version = "LATEST" #Fargateを常に最新バージョンで使用

    network_configuration { # コンテナの稼働区画の設定
    subnets         = var.private_subnet_ids # サブネットを指定
    security_groups = [var.ecs_security_group_id] # セキュリティグループを指定
    assign_public_ip = false # コンテナにパブリックIPを割り当てない
  }

  load_balancer {
    target_group_arn = var.alb_target_group_arn # ターゲットグループを指定
    container_name   = var.container_name 
    container_port   = var.container_port # コンテナの何番ポートにリクエストを送るか指定
  }
}
```

* タスクを常時起動
* ALB のターゲットグループと連携

---

#### ALB / ターゲットグループ / リスナー

```hcl
resource "aws_lb" "api_alb" {
  name               = var.alb_name
  internal           = false #ロードバランサーをインターネットに公開(パブリックIP割り当て)
  load_balancer_type = "application" #ALBを使用
  security_groups    = [var.alb_security_group_id]
  subnets            = var.public_subnet_ids
}
```

```hcl
resource "aws_lb_listener" "http_listener" {
  load_balancer_arn = aws_lb.api_alb.arn #ALBを指定

#インターネットから指定のプロトコルで来た指定のポートへの通信を受信  
  port              = var.listener_port 
  protocol          = var.listener_protocol

  default_action { #デフォルトの動作
    type             = "forward" #届いた通信をそのまま転送する
    target_group_arn = aws_lb_target_group.api_tg.arn #転送先の指定
  }
}
```

```hcl
resource "aws_lb_target_group" "api_tg" {
  name        = var.target_group_name
 #ALBからコンテナに届ける際のポート番号とプロトコル
  port        = var.target_group_port
  protocol    = var.target_group_protocol


  vpc_id      = var.vpc_id #VPCを指定
  target_type = var.target_group_type #Fargateを使う場合は必ずip

  health_check {
    path                = var.health_check_path #対象のURL
    protocol            = var.health_check_protocol
    matcher             = var.health_check_success_codes #成功とみなすHTTPステータスコード(200)
    interval            = var.health_check_interval #何秒ごとに実施するか
    timeout             = var.health_check_timeout
    healthy_threshold   = var.health_check_healthy_threshold #何回連続で成功したら、状態を切り替えるか
    unhealthy_threshold = var.health_check_unhealthy_threshold #何回連続で失敗したら、状態を切り替えるか
  }
}

```

* ALB が外部リクエストを受信
* ヘルスチェック結果に応じて ECS タスクへ転送
* ECS タスクは IP ターゲットとして登録

---

## 4. 振り返り

### 改善できるところ

* HTTPS（ACM）対応
* Auto Scaling の導入

---

### 次にやること

次章では、
**ECS 上のバックエンドから DynamoDB / S3 のデータを取得し、
フロントエンド（ブラウザ）に表示する流れ** を整理します。

* **[①構成の説明]()**  
* **[②S3(静的Webサイトホスティング)/フロントエンド]()**  
* **[③VPC関連]()**  
* **[④Docker/バックエンド]()**  
* **[⑤ECS + ALB]()**  
* **[⑥DynamoDB・S3連携]()**  
* **[⑦CI/CD(アプリケーション)]()** 
* **[⑧CI/CD(インフラ)]()**

---

## 5. 参考リンク

* [ECS公式ドキュメント](https://docs.aws.amazon.com/ja_jp/ecs/)
* [ELB公式ドキュメント](https://docs.aws.amazon.com/ja_jp/elasticloadbalancing/)

---

## 6. コード全体

variables.tf, output.tfは省略

modules/ecr/main.tf

```hcl
resource "aws_ecr_repository" "api_ecr" {
  name = var.ecr_name
  image_scanning_configuration {
    scan_on_push = true  # イメージpush時にスキャン
  }
  force_delete = true
  image_tag_mutability = "MUTABLE"  # タグ変更可否 (MUTABLE または IMMUTABLE)
}
```
modules/ecs/main.tf
```hcl
resource "aws_ecs_cluster" "main" {
  name = var.ecs_cluster_name

  setting {
    name  = "containerInsights"
    value = "enabled" # Container Insightsを有効化
  }

  tags = {
    Name        = var.ecs_cluster_name
    Environment = var.environment
  }
}

resource "aws_ecs_task_definition" "api_task" {
  family                   = var.task_definition_family
  cpu                      = var.task_cpu
  memory                   = var.task_memory
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  execution_role_arn       = var.ecs_task_execution_role_arn
  task_role_arn            = var.ecs_task_role_arn

  container_definitions = jsonencode([
    {
      name        = var.container_name,
      image       = "${var.ecr_repository_url}:${var.container_image_tag}", # ECRリポジトリのURLを使用
      essential   = true,
      portMappings = [
        {
          containerPort = var.container_port,
          hostPort      = var.container_port,
          protocol      = "tcp"
        }
      ],
      environment = [
        {
          name  = "DYNAMODB_TABLE_NAME",
          value = var.dynamodb_table_name
        },
        {
          name  = "AWS_REGION",
          value = var.region
        },
        {
          name = "S3_BUCKET",
          value = var.s3_storage_bucket_name
        },
      ],
      logConfiguration = {
        logDriver = "awslogs",
        options = {
          "awslogs-group"         = "/ecs/dev-api-ecs-task-01", # CloudWatch Logsグループ名
          "awslogs-region"        = var.region,
          "awslogs-stream-prefix" = "ecs"
        }
      }
    }
  ])

  tags = {
    Name        = var.task_definition_family
    Environment = var.environment
  }
}

resource "aws_cloudwatch_log_group" "ecs_log_group" {
  name              = "/ecs/${var.task_definition_family}"
  retention_in_days = 1 # ログ保持期間（任意）

  tags = {
    Name        = "/ecs/${var.task_definition_family}"
    Environment = var.environment
  }
}


resource "aws_ecs_service" "api_service" {
  name            = var.ecs_service_name
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.api_task.arn
  desired_count   = var.desired_task_count
  launch_type     = "FARGATE"
  platform_version = "LATEST"

  network_configuration {
    subnets         = var.private_subnet_ids
    security_groups = [var.ecs_security_group_id]
    assign_public_ip = false
  }

  load_balancer {
    target_group_arn = var.alb_target_group_arn
    container_name   = var.container_name
    container_port   = var.container_port
  }

  tags = {
    Name        = var.ecs_service_name
    Environment = var.environment
  }
}

```

modules/alb/main.tf
```hcl
resource "aws_lb_target_group" "api_tg" {
  name        = var.target_group_name
  port        = var.target_group_port
  protocol    = var.target_group_protocol
  vpc_id      = var.vpc_id
  target_type = var.target_group_type

  health_check {
    path                = var.health_check_path
    protocol            = var.health_check_protocol
    matcher             = var.health_check_success_codes
    interval            = var.health_check_interval
    timeout             = var.health_check_timeout
    healthy_threshold   = var.health_check_healthy_threshold
    unhealthy_threshold = var.health_check_unhealthy_threshold
  }

  tags = {
    Name        = var.target_group_name
    Environment = var.environment
  }
}

resource "aws_lb" "api_alb" {
  name               = var.alb_name
  internal           = false
  load_balancer_type = "application"
  security_groups    = [var.alb_security_group_id]
  subnets            = var.public_subnet_ids

  tags = {
    Name        = var.alb_name
    Environment = var.environment
  }
}

resource "aws_lb_listener" "http_listener" {
  load_balancer_arn = aws_lb.api_alb.arn
  port              = var.listener_port
  protocol          = var.listener_protocol

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.api_tg.arn
  }
}
```