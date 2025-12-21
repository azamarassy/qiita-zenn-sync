---
title: 【AWS】ECSを用いた簡易web構築⑧(CI/CDパイプライン構築(インフラ))
tags:
  - 'AWS'
  - 'CI/CD'
  - 'Github Actions'
  - 'Terraform'
private: true
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

# 第8章 CI/CDパイプライン構築②（インフラ）

ECSを用いた簡易web構築シリーズの最終回です。  
  
今回はインフラ更新時のCI/CDパイプライン構築です。  
下記理由からGithub Actionsを用います。

* 無料利用可能
* YAML定義がシンプルで学習しやすい
* サーバー管理不要
* GitHubとの高い親和性

## 📚 目次

* [1. 構成の説明](#1-構成の説明)
* [2. 前提条件](#2-前提条件)
* [3. workflows構成](#3-workflows構成)
* [4. 振り返り](#4-振り返り)
* [5. 参照](#5-参照)
* [6. コード全体](#6-コード全体)


---

## 1. 構成の説明

### 何をしたいか

* インフラ変更時の **ビルド・デプロイの自動化**

---

### どんな構成か・何ができるか

インフラに関して、CI/CD を **2 つのワークフロー** に分離しています。

| ワークフロー               | 役割                      |
| -------------------- | ----------------------- |
| `pull_request.yml`   | Terraform Plan（変更内容の確認） |
| `infrastructure.yml` | Terraform Apply（インフラ更新） |

* **インフラPR作成時**：terraform plan実行
* **インフラMerge時**：terraform apply実行

![構成図](../images/ECS-web-architecture/ECS-DynamoDB-S3-CICD-git1.jpg)

---

### なぜこの構成にしたか（設計意図）

インフラ更新時は、
**PR作成 → レビュー → Merge** の流れを前提とし、

* **PR作成時**：検証用ワークフロー（Terraform plan）
* **Merge時**：本番反映用ワークフロー（Terraform apply）

を実行する設計としています。

これにより、レビュー・承認後に Merge されたタイミングでのみ、本番反映用のワークフローを実行し、意図しないインフラ変更を防ぐ設計としています。

---

## 2. 前提条件

以下が利用可能であることを前提とする。

* GitHub リポジトリ
* GitHub Actions が有効
* AWS アカウント
* Terraform（インフラは構築済み）
* IAM OIDC 設定済みの GitHub Actions 用ロール

※以下はGithub Actions上のLinuxで実行するため不要。
* Docker
* Node.js 18.x（フロントエンド用）

---

## 3. workflows構成

### フォルダ構成（関連部分）

```
.
├── backend/
├── frontend/
├── terraform/
└── .github/workflows/
    ├── application.yml
    ├── pull_request.yml
    └── infrastructure.yml

```

---

### CI/CD で参照している Terraform Outputs

* `alb_dns_name`
* `ecr_repository_url`
* `ecs_cluster_name`
* `ecs_service_name`
* `ecs_task_definition_family`

---

**実行条件の設計**

```yml
    paths-ignore:
      - 'backend/**' # backend ディレクトリ以外の変更があった場合のみ実行
      - 'frontend/**' # frontend ディレクトリ以外の変更があった場合のみ実行
```
アプリケーション以外のディレクトリに変更があった場合に実行します。

---

**terraform apply**

```yml
      - name: Terraform Apply
        working-directory: ./terraform
        env:
          TF_VAR_aws_region: ${{ env.AWS_REGION }}
          TF_VAR_aws_account_id: ${{ secrets.AWS_ACCOUNT_ID }}
          TF_VAR_alb_ingress_source_ip: ${{ secrets.ALB_INGRESS_SOURCE_IP }}
          TF_VAR_s3_web_bucket_access_ip: ${{ secrets.S3_WEB_BUCKET_ACCESS_IP }}
          TF_VAR_web_bucket_name: ${{ secrets.WEB_BUCKET_NAME }}
          TF_VAR_storage_bucket_name: ${{ secrets.STORAGE_BUCKET_NAME }}
        run: |
          terraform apply -auto-approve
```
アプリケーションCI/CDと同様にOICD認証、terraform initを実行後、terraform applyを実行します。
公開したくない情報を変数にして引数として渡して実行しています。

```yml
      - name: Terraform Plan
        id: plan
        working-directory: ./terraform
        run: |
          terraform plan
```
PR作成時はOICD認証、terraform initを実行後、terraform planを実行します。

---

## 4. 振り返り

以上で本シリーズは終了です。
ここまで閲覧くださりありがとうございます。
初めてのAWS環境構築記事でしたが、皆様の参考になっていれば幸いです。

次回はS3 + CloudFront 構成について記事を書きたいと考えています。

以下は本シリーズの記事へのリンクです。

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

* [Github Actions公式ドキュメント](https://docs.github.com/ja/actions)

---

## 6. コード全体

```yml
name: CI/CD Pipeline for Infrastructure

on:
  push:
    branches:
      - master
    paths-ignore:
      - 'backend/**' # backend ディレクトリ以外の変更があった場合のみ実行
      - 'frontend/**' # frontend ディレクトリ以外の変更があった場合のみ実行

env:
  # ワークフロー全体で利用するSecretsをenvとして定義
  AWS_REGION: ${{ secrets.AWS_REGION }}
  ECR_REPOSITORY_NAME: ${{ secrets.ECR_REPOSITORY_NAME }}
  # Terraform状態管理用のS3バケット名とDynamoDBテーブル名
  TFSTATE_S3_BUCKET_NAME: ${{ secrets.TFSTATE_S3_BUCKET_NAME }}
  TFSTATE_DYNAMODB_TABLE_NAME: ${{ secrets.TFSTATE_DYNAMODB_TABLE_NAME }}

jobs:
  deploy_infrastructure:
    name: Update All Infrastructure
    runs-on: ubuntu-latest

    permissions:
      id-token: write   # ← OIDC に必須
      contents: read     

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      # --- OIDC を用いた AWS 認証 ---
      - name: Configure AWS credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/GitHubActionsDeployRole
          aws-region: ${{ env.AWS_REGION }}

      - name: Set up Terraform
        uses: hashicorp/setup-terraform@v3

      - name: Terraform Init
        working-directory: ./terraform
        run: |
          terraform init \
            -backend-config="bucket=${{ env.TFSTATE_S3_BUCKET_NAME }}" \
            -backend-config="dynamodb_table=${{ env.TFSTATE_DYNAMODB_TABLE_NAME }}" \
            -backend-config="key=dev/terraform.tfstate" \
            -backend-config="region=${{ env.AWS_REGION }}"

      - name: Terraform Apply
        working-directory: ./terraform
        env:
          TF_VAR_aws_region: ${{ env.AWS_REGION }}
          TF_VAR_aws_account_id: ${{ secrets.AWS_ACCOUNT_ID }}
          TF_VAR_alb_ingress_source_ip: ${{ secrets.ALB_INGRESS_SOURCE_IP }}
          TF_VAR_s3_web_bucket_access_ip: ${{ secrets.S3_WEB_BUCKET_ACCESS_IP }}
          TF_VAR_web_bucket_name: ${{ secrets.WEB_BUCKET_NAME }}
          TF_VAR_storage_bucket_name: ${{ secrets.STORAGE_BUCKET_NAME }}
        run: |
          terraform apply -auto-approve
```

pull_request.yml
```yml
name: Terraform Plan

on:
  pull_request:
    paths:
      - 'terraform/**'
    # branches: の行は削除し、masterへのPRだけでなく、全てのリポジトリ変更に対してPlanが実行されるようにします。

env:
  AWS_REGION: ${{ secrets.AWS_REGION }}
  TFSTATE_S3_BUCKET_NAME: ${{ secrets.TFSTATE_S3_BUCKET_NAME }}
  TFSTATE_DYNAMODB_TABLE_NAME: ${{ secrets.TFSTATE_DYNAMODB_TABLE_NAME }}

jobs:
  plan: # ジョブ名をより具体的に変更しました
    name: Terraform Plan
    runs-on: ubuntu-latest
    permissions:
      contents: read # リポジトリのチェックアウトのため
      pull-requests: write # PRコメント投稿のため (推奨)
      id-token: write   # ← OIDC に必須

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      # --- OIDC を用いた AWS 認証 ---
      - name: Configure AWS credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/GitHubActionsDeployRole
          aws-region: ${{ env.AWS_REGION }}

      - name: Set up Terraform
        uses: hashicorp/setup-terraform@v3

      - name: Terraform Init
        id: init
        working-directory: ./terraform
        run: |
          terraform init \
            -backend-config="bucket=${{ env.TFSTATE_S3_BUCKET_NAME }}" \
            -backend-config="dynamodb_table=${{ env.TFSTATE_DYNAMODB_TABLE_NAME }}" \
            -backend-config="key=dev/terraform.tfstate" \
            -backend-config="region=${{ env.AWS_REGION }}"

      - name: Terraform Plan
        id: plan # IDを追加して、後続のステップでplanの結果を利用できるようにします
        working-directory: ./terraform
        # Plan結果をGitHub Actionsの出力に保存するため、パイプを使って出力を整形
        run: |
          terraform plan -no-color | tee plan_output.txt

      - name: Add Plan to Pull Request Comment
        uses: actions/github-script@v6 # PRコメント投稿用の汎用アクション
        if: github.event_name == 'pull_request'
        with:
          script: |
            const fs = require('fs');
            const planOutput = fs.readFileSync('terraform/plan_output.txt', 'utf8');
            const output = `### Terraform Plan Result\n\n\`\`\`hcl\n${planOutput}\n\`\`\``
            
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: output
            })
```