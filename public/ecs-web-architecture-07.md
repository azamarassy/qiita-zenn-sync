---
title: 【AWS】ECSを用いた簡易web構築⑦(CI/CDパイプライン構築(アプリケーション))
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

# 第7章 CI/CDパイプライン構築①（アプリケーション）

今回はアプリケーションコード更新時のCI/CDパイプライン構築です。
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

* アプリケーション変更時の **ビルド・デプロイの自動化**

---

### どんな構成か・何ができるか

本構成では、CI/CD を **3 つのワークフロー** に分離しています。

| ワークフロー               | 役割                      |
| -------------------- | ----------------------- |
| `application.yml`    | アプリケーションのビルド・デプロイ       |
| `pull_request.yml`   | Terraform Plan（変更内容の確認） |
| `infrastructure.yml` | Terraform Apply（インフラ更新） |



* **アプリ変更時**：ECS / S3 のみ更新（インフラ変更なし）
* **インフラPR作成時**：terraform plan実行
* **インフラMerge時**：terraform apply実行

![構成図](../images/ECS-web-architecture/ECS-DynamoDB-S3-CICD-git1.jpg)

---

### なぜこの構成にしたか（設計意図）

#### ワークフロー分離

* **実行時間とコストの最適化**

  * アプリの軽微な修正で Terraform Apply を走らせない
* **インフラの安定性向上**

  * 破壊的操作（apply）はレビュー後のみ実行
* **責務の明確化**

  * 「アプリケーションの変更」と「インフラの変更」を明確に分離

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

**実行条件の設定**

```yml
name: CI/CD Pipeline for Web Application

on:
  push:
    branches:
      - master
    paths:
      - 'backend/**' # backend ディレクトリ内の変更があった場合のみ実行
      - 'frontend/**' # frontend ディレクトリ内の変更があった場合のみ実行
```

アプリケーションのディレクトリ(backend, frontend)を変更し、masterブランチにmarge/pushした際に実行。

---
**環境変数の設定**

```yml
env:
  # ワークフロー全体で利用するSecretsをenvとして定義
  # Terraform状態管理用のS3バケット名とDynamoDBテーブル名
  TFSTATE_S3_BUCKET_NAME: ${{ secrets.TFSTATE_S3_BUCKET_NAME }}
  TFSTATE_DYNAMODB_TABLE_NAME: ${{ secrets.TFSTATE_DYNAMODB_TABLE_NAME }}
```
Github Secretsに保存したシークレットを環境変数に設定。

---
**OIDC認証**

```yml
jobs:
  update_application:
    name: Update Application (App-Only Deployment)
    runs-on: ubuntu-latest # 最新版のubuntuで実行

    permissions:
      id-token: write   # トークンを発行することを許可。OIDC に必須
      contents: read    # ソースコードの読み取り許可
      
    steps:
      - name: Checkout repository 
        uses: actions/checkout@v4 # GitHub上にあるコードを実行環境（仮想マシン）の中にコピー

      # --- OIDC を用いた AWS 認証 ---
      - name: Configure AWS credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/GitHubActionsDeployRole # GitHub Actionsが利用するIAMロールのARNを指定
          aws-region: ${{ env.AWS_REGION }}
```
aws-actions/configure-aws-credentials を使い、OIDC経由でAWSのIAMロールを引き受け。  
長期的なアクセスキーをGitHubに保存する必要がないため、万が一GitHubの設定が漏洩しても、被害を最小限に抑えられる。

---
**Terraformからの情報取得準備**

```yml
      - name: Set up Terraform
        uses: hashicorp/setup-terraform@v3  # 仮想環境で terraform コマンドを使えるようにする

      - name: Terraform Init (Stateファイルの読み込み)
        working-directory: ./terraform # 実行ディレクトリ指定
        run: |
          terraform init \
            -backend-config="bucket=${{ env.TFSTATE_S3_BUCKET_NAME }}" \
            -backend-config="dynamodb_table=${{ env.TFSTATE_DYNAMODB_TABLE_NAME }}" \
            -backend-config="key=dev/terraform.tfstate" \
            -backend-config="region=${{ env.AWS_REGION }}"
```

Terraformから情報を受け取るため、terraform initを実行。

| 項目 | 内容 | 役割 |
| --- | --- | --- |
| `run` | `terraform init` | 初期化コマンド（プラグインの取得など）を実行。 |
| `-backend-config` | `bucket` | 状態管理ファイル（tfstate）を保存する **S3バケット名** を指定。 |
| `-backend-config` | `dynamodb_table` | 同時実行を防ぐための **DynamoDBテーブル名** を指定。 |
| `-backend-config` | `key` | S3内での **tfstateファイルの保存パス** を指定。 |
| `-backend-config` | `region` | バックエンド（S3/DynamoDB）が存在する **リージョン** を指定。 |

---
**Terraformからの情報取得**

```yml
      - name: Get infrastructure info from Terraform State
        id: get_infra
        working-directory: ./terraform
        run: |
          ALB_DNS=$(terraform output -raw alb_dns_name)
          ECR_URL=$(terraform output -raw ecr_repository_url)
          ECS_CLUSTER=$(terraform output -raw ecs_cluster_name)
          ECS_SERVICE=$(terraform output -raw ecs_service_name)
          TASK_FAMILY=$(terraform output -raw ecs_task_definition_family)
          
          echo "ALB_DNS_NAME=$ALB_DNS" >> $GITHUB_OUTPUT
          echo "ECR_REPOSITORY_URL=$ECR_URL" >> $GITHUB_OUTPUT
          echo "ECS_CLUSTER_NAME=$ECS_CLUSTER" >> $GITHUB_OUTPUT
          echo "ECS_SERVICE_NAME=$ECS_SERVICE" >> $GITHUB_OUTPUT
          echo "TASK_DEFINITION_FAMILY=$TASK_FAMILY" >> $GITHUB_OUTPUT
```
Terraformが作成したリソースの情報（名前やURLなど）を取得し、後続のステップで使い回せるように保存。

| 行（コード） | 意味・役割 |
| --- | --- |
| **`id: get_infra`** | このステップに名前（ID）を付ける。後で他のステップから `steps.get_infra.outputs.XXX` という形で値を参照するために必要。 |
| **`(変数名)=$(terraform output ...)`** | Terraformの `output` 機能を使って、作成されたリソースの情報を取得し、一時的な変数に代入。 |
| **`echo "KEY=VALUE" >> $GITHUB_OUTPUT`** | 取得した値をGitHub Actions専用の保存先（$GITHUB_OUTPUT）に書き込む。これにより、**このステップが終わった後も他のステップからこの値が参照可能。** |


1. **`terraform output` の前提条件**
このコマンドが成功するためには、Terraformのコード側（`.tf`ファイル）で `output "alb_dns_name" { ... }` のように、**あらかじめ出力を定義しておく**必要があります。定義がないとエラーになります。
2. **`$GITHUB_OUTPUT` に書く理由**
通常、シェルスクリプト内の変数（例：`$ALB_DNS`）はそのステップが終わると消えます。後続のステップでこれらの値を使いたいため、GitHub Actionsの仕組みを使って値を出力しています。
3. **`-raw` オプションの重要性**
`-raw` を付けないと、出力値がダブルクォーテーション `"` で囲まれて取得されることがあります。URLなどに引用符が混ざると後続の処理で失敗するため、このオプションは必須です。

---
**ECRへログインし、コンテナに付けるタグを決定**

```yml
      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2
        
      - name: Set image version tag
        id: set_image_tag
        run: |
          IMAGE_TAG="v1.0.${{ github.run_number }}"
          echo "IMAGE_TAG=$IMAGE_TAG" >> $GITHUB_OUTPUT
```

| 行（コード） | 意味・役割 |
| --- | --- |
| **`uses: aws-actions/amazon-ecr-login@v2`** | 前のステップで取得した認証情報を使って、`docker login` を自動で実行。 |
| **`IMAGE_TAG="v1.0.${{ github.run_number }}"`** | タグ名設定。 |
| **`echo "IMAGE_TAG=$IMAGE_TAG" >> $GITHUB_OUTPUT`** | 組み立てたタグ名を、後続のステップから `steps.set_image_tag.outputs.IMAGE_TAG` として使えるように保存。 |

#### `github.run_number` を使う理由

どの時点のコードがデプロイされているか分からなくなるので、コンテナイメージに `latest` というタグだけを使い続けるのは、実務では推奨されません。
実行番号を含めることで、トラブルが起きた際に正常に動作していたバージョンに切り戻しが可能になります。

---
**コンテナをビルドし、ECRへ保存**
```yml
      - name: Build, tag, and push image to Amazon ECR
        env:
          ECR_REGISTRY_URL: ${{ steps.get_infra.outputs.ECR_REPOSITORY_URL }}
          VERSIONED_IMAGE_TAG: ${{ steps.set_image_tag.outputs.IMAGE_TAG }}
        run: |
          docker build -t $ECR_REGISTRY_URL:$VERSIONED_IMAGE_TAG ./backend
          docker tag $ECR_REGISTRY_URL:$VERSIONED_IMAGE_TAG $ECR_REGISTRY_URL:latest
          docker push $ECR_REGISTRY_URL:$VERSIONED_IMAGE_TAG
          docker push $ECR_REGISTRY_URL:latest
```

| 行（コード） | 意味・役割 |
| --- | --- |
| **`ECR_REGISTRY_URL`** | 前のステップで取得した「ECRリポジトリのURL」を代入。 |
| **`VERSIONED_IMAGE_TAG`** | 前のステップで作った「v1.0.x」といったバージョン番号を代入。 |
| **`docker build ... ./backend`** | **コンテナの作成**: `./backend` フォルダのプログラムを元に、バージョンタグ（v1.0.x）を付けてビルド。 |
| **`docker tag ... :latest`** | **最新版ラベルの付与**: 今作ったイメージに、別名として「latest」という名前も付ける（実体は1つで、名前が2つある状態）。 |
| **`docker push ... :v1.0.x`** | **送信（1）**: バージョン番号が付いたイメージをAWS（ECR）にアップロード。 |
| **`docker push ... :latest`** | **送信（2）**: latestタグが付いたイメージもアップロード。 |


#### 「バージョンタグ」と「latest」の両方を送る理由

* **バージョンタグ**: 「何かあった時に過去のバージョンに戻す」ために必要です。
* **latest**: デプロイ設定で「常に最新（latest）を使う」としている場合に、ここを更新するだけで勝手に中身が入れ替わるようにするためです。

---
**フロントエンドのビルド・デプロイ**

```yml
      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'

      - name: Install frontend dependencies
        working-directory: ./frontend
        run: npm install

      - name: Create .env.production
        working-directory: ./frontend
        run: |
          echo "REACT_APP_API_URL=http://${{ steps.get_infra.outputs.ALB_DNS_NAME }}" > .env.production

      - name: Build frontend
        working-directory: ./frontend
        run: npm run build

      - name: Sync static website to S3
        run: |
          aws s3 sync ./frontend/build s3://${{ secrets.WEB_BUCKET_NAME }} --delete
```
フロントエンドをビルドし、完成した静的ファイルをS3バケットへアップロードして公開。

| 項目 | 意味・役割 |
| --- | --- |
| **`uses: actions/setup-node@v4`** | **実行環境の準備**: 仮想マシンで Node.js を使えるようにする。 |
| **`run: npm install`** | **ライブラリ取得**: `package.json` に基づき、ビルドに必要なライブラリ（Reactなど）をインストール。 |
| **`run: echo "REACT_APP_... > .env.production`** | **接続先の注入**: 前のステップで取得した **ALBのDNS名（APIのURL）** を、フロントエンドの設定ファイルに書き込む。これにより、アプリがどこに通信すべきかを自動で反映。 |
| **`run: npm run build`** | **ビルド実行**: ソースコードを最適化（軽量化）し、ブラウザで読み取れる静的ファイル（HTML/CSS/JS）に変換。 |
| **`aws s3 sync ... --delete`** | **S3への同期**: 生成された `./frontend/build` 内のファイルをS3へアップロード。`--delete` を付けることで、古い不要なファイルを削除し、常に最新の状態に保つ。 |

---
**ECSタスク定義の取得**
```yml
      - name: Download current task definition
        run: |
          aws ecs describe-task-definition \
            --task-definition ${{ steps.get_infra.outputs.TASK_DEFINITION_FAMILY }} \
            --query taskDefinition > task-definition.json
```
現在AWS上で動いているタスク定義を手元にダウンロード。

| 行（コード） | 意味・役割 |
| --- | --- |
| **`aws ecs describe-task-definition`** | **AWSへの問い合わせ**: 指定したタスク定義の詳細情報を取得。 |
| **`--task-definition ${{ ... }}`** | **対象の指定**: 前のステップで取得した「タスク定義名（Family）」を指定して、どの設定を落とすか指定。 |
| **`--query taskDefinition`** | **情報の絞り込み**: AWSからの返答には不要なデータも含まれるため、「タスク定義の設定本体」だけを抽出。 |
| **`> task-definition.json`** | **ファイル保存**: 取得した内容を `task-definition.json` という名前のファイルとして保存。 |


#### ダウンロードが必要な理由

ECSのデプロイ（`aws-actions/amazon-ecs-deploy-task-definition`）を行う際、**ベースとなるJSONファイル**が手元に必要だからです。
現在の設定ファイルをダウンロードし、次のステップでコンテナイメージのタグだけを最新に書き換えるのが一般的なデプロイ方法です。


---
**タスク定義の編集**
```yml
      - name: Update task definition with new image
        id: update_task_def
        env:
          ECR_REGISTRY_URL: ${{ steps.get_infra.outputs.ECR_REPOSITORY_URL }}
          VERSIONED_IMAGE_TAG: ${{ steps.set_image_tag.outputs.IMAGE_TAG }}
        run: |
          cat task-definition.json | \
            jq --arg IMAGE "$ECR_REGISTRY_URL:$VERSIONED_IMAGE_TAG" \
            '.containerDefinitions[0].image = $IMAGE' | \
            jq 'del(.taskDefinitionArn, .revision, .status, .requiresAttributes, .compatibilities, .registeredAt, .registeredBy)' \
            > new-task-definition.json
          
          cat new-task-definition.json
```
ダウンロードした古い設定ファイルのイメージURLを、今回ビルドした最新のものに書き換え、不要な項目を削除。

| 行（コード） | 意味・役割 |
| --- | --- |
| **`env:`** | 書き換えに使う「新しいECRのURL」と「新タグ（v1.0.x）」を環境変数として用意。 |
| **`cat task-definition.json`** | 前のステップで保存したJSONファイルを読み込む。 |
| **`jq --arg IMAGE "..."`** | **中身の書き換え**: `jq` に新しいイメージ名を渡し、JSON内の `image` という項目を最新のものに書き換える。 |
| **`jq 'del(.taskDefinitionArn, ...)'`** | **不要な項目の削除**: AWSから落としたJSONには「ID」や「登録日時」などの**自動付与される項目**が含まれるので削除。 |
| **`> new-task-definition.json`** | **保存**: 書き換えが終わったデプロイ用設定ファイルを新しく保存。 |


#### 1. `jq 'del(...)'` が必要な理由

AWS CLIで取得したJSONには、その時の「リビジョン番号（何回目の更新か）」や「ステータス」などが含まれています。
これを削除せずにそのまま `register-task-definition` （登録）しようとすると、**「その項目はユーザーが指定できるものではありません」というエラー**で弾かれます。これは、CLIやGitHub Actionsでデプロイする際の必須テクニックです。

#### 2. `.containerDefinitions[0]` の前提

このコードは「コンテナが1つだけ」の構成を想定しています。


#### 3. 代替手段としての公式アクション

この `jq` の一連の作業を代行してくれる `aws-actions/amazon-ecs-render-task-definition` という公式アクションも存在します。

---
**新しいタスク定義の登録**
```yml
      - name: Register new task definition
        id: register_task_def
        run: |
          TASK_DEF_ARN=$(aws ecs register-task-definition \
            --cli-input-json file://new-task-definition.json \
            --query 'taskDefinition.taskDefinitionArn' \
            --output text)
          echo "TASK_DEFINITION_ARN=$TASK_DEF_ARN" >> $GITHUB_OUTPUT
          echo "新しいTask Definition: $TASK_DEF_ARN"
```
新しく作った設定ファイルをAWSに登録し、発行されたARNを記録。

| 行（コード） | 意味・役割 |
| --- | --- |
| **`TASK_DEF_ARN=$(...)`** | コマンドの実行結果（新しいタスク定義のID）を変数 `TASK_DEF_ARN` に代入。 |
| **`aws ecs register-task-definition`** | **AWSへ登録**: 「新しいタスク定義を登録するコマンド。 |
| **`--cli-input-json file://...`** | **ファイルの指定**: 前のステップで作成した `new-task-definition.json` を読み込ませる。 |
| **`--query '...' --output text`** | **結果の抽出**: 登録完了後にAWSから返ってくる情報の中から、**「新しく発行されたARN（リビジョン番号付きのID）」**だけを抜き出す。 |
| **`echo "...=$TASK_DEF_ARN" >> $GITHUB_OUTPUT`** | **保存**: 新しいARNを、「サービスの更新」ステップで使えるように保存。 |

このステップで行っているのは新しいタスク定義の登録のみ。

---
**ECSサービスの更新（デプロイ実行）**
```yml
      - name: Update ECS service with new task definition
        run: |
          aws ecs update-service \
            --cluster ${{ steps.get_infra.outputs.ECS_CLUSTER_NAME }} \
            --service ${{ steps.get_infra.outputs.ECS_SERVICE_NAME }} \
            --task-definition ${{ steps.register_task_def.outputs.TASK_DEFINITION_ARN }} \
            --force-new-deployment
```

新しいタスク定義を使って、実際に動いているコンテナを最新版に入れ替え。


| 行（コード） | 意味・役割 |
| --- | --- |
| **`aws ecs update-service`** | **デプロイ実行**: 稼働中の「サービス」の設定を更新するコマンド。 |
| **`--cluster ${{ ... }}`** | **場所の指定**: サービスが動いている「クラスター名」を指定。 |
| **`--service ${{ ... }}`** | **対象の指定**: 更新したい具体的な「サービス名」を指定。 |
| **`--task-definition ${{ ... }}`** | **新設定の適用**: 一つ前のステップで登録した、**最新のリビジョン番号付きのタスク定義（ARN）**を指定。 |
| **`--force-new-deployment`** | **強制更新**: 「設定内容が同じでも、強制的に新しいコンテナを作り直して入れ替える」という指示。 |

#### 1. `--force-new-deployment` が必要な理由

通常、ECSはタスク定義が変わらないとコンテナを入れ替えません。
しかし、例えばイメージタグに `latest` を使っている場合、AWSから見ると「設定（タグ名）は変わっていない」と判断され、中身が更新されません。このオプションを付けることで、**確実に最新のイメージでコンテナを再起動させる**ことができます。

#### 2. ローリングアップデートの挙動

このコマンドを投げると、ECSは「古いコンテナをいきなり消す」のではなく、**「新しいコンテナが正常に起動したのを確認してから、古い方を消す」**というローリングアップデートを開始します。


---

#### デプロイ完了を待機

```yml

      - name: Wait for service stability
        run: |
          aws ecs wait services-stable \
            --cluster ${{ steps.get_infra.outputs.ECS_CLUSTER_NAME }} \
            --services ${{ steps.get_infra.outputs.ECS_SERVICE_NAME }}
```
aws ecs wait services-stable を実行することで、新しいタスクが正常に起動し、古いタスクとの入れ替えが完了するまでワークフローを待機させます。  
これにより、デプロイ指示の成否だけでなく、実行環境（コンテナ）が正しく立ち上がったことまでをCI/CDパイプライン上で保証できます。

---

## 4. 振り返り

### 改善できるところ

* アプリケーションPR作成時のテスト用workflow追加
* npm install実行時のキャッシュの活用


次回はインフラのCI/CDパイプライン構築です。
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
name: CI/CD Pipeline for Web Application

on:
  push:
    branches:
      - master
    paths:
      - 'backend/**' # backend ディレクトリ内の変更があった場合のみ実行
      - 'frontend/**' # frontend ディレクトリ内の変更があった場合のみ実行

env:
  # ワークフロー全体で利用するSecretsをenvとして定義
  AWS_REGION: ${{ secrets.AWS_REGION }}
  ECR_REPOSITORY_NAME: ${{ secrets.ECR_REPOSITORY_NAME }}
  # Terraform状態管理用のS3バケット名とDynamoDBテーブル名
  TFSTATE_S3_BUCKET_NAME: ${{ secrets.TFSTATE_S3_BUCKET_NAME }}
  TFSTATE_DYNAMODB_TABLE_NAME: ${{ secrets.TFSTATE_DYNAMODB_TABLE_NAME }}

jobs:
  update_application:
    name: Update Application (App-Only Deployment)
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

      # --- 1. インフラ情報取得 ---
      - name: Set up Terraform
        uses: hashicorp/setup-terraform@v3

      - name: Terraform Init (Stateファイルの読み込み)
        working-directory: ./terraform
        run: |
          terraform init \
            -backend-config="bucket=${{ env.TFSTATE_S3_BUCKET_NAME }}" \
            -backend-config="dynamodb_table=${{ env.TFSTATE_DYNAMODB_TABLE_NAME }}" \
            -backend-config="key=dev/terraform.tfstate" \
            -backend-config="region=${{ env.AWS_REGION }}"

      - name: Get infrastructure info from Terraform State
        id: get_infra
        working-directory: ./terraform
        run: |
          ALB_DNS=$(terraform output -raw alb_dns_name)
          ECR_URL=$(terraform output -raw ecr_repository_url)
          ECS_CLUSTER=$(terraform output -raw ecs_cluster_name)
          ECS_SERVICE=$(terraform output -raw ecs_service_name)
          TASK_FAMILY=$(terraform output -raw ecs_task_definition_family)
          
          echo "ALB_DNS_NAME=$ALB_DNS" >> $GITHUB_OUTPUT
          echo "ECR_REPOSITORY_URL=$ECR_URL" >> $GITHUB_OUTPUT
          echo "ECS_CLUSTER_NAME=$ECS_CLUSTER" >> $GITHUB_OUTPUT
          echo "ECS_SERVICE_NAME=$ECS_SERVICE" >> $GITHUB_OUTPUT
          echo "TASK_DEFINITION_FAMILY=$TASK_FAMILY" >> $GITHUB_OUTPUT
          
      # --- 2. Docker Build & Push ---
      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2
        
      - name: Set image version tag
        id: set_image_tag
        run: |
          IMAGE_TAG="v1.0.${{ github.run_number }}"
          echo "IMAGE_TAG=$IMAGE_TAG" >> $GITHUB_OUTPUT

      - name: Build, tag, and push image to Amazon ECR
        env:
          ECR_REGISTRY_URL: ${{ steps.get_infra.outputs.ECR_REPOSITORY_URL }}
          VERSIONED_IMAGE_TAG: ${{ steps.set_image_tag.outputs.IMAGE_TAG }}
        run: |
          docker build -t $ECR_REGISTRY_URL:$VERSIONED_IMAGE_TAG ./backend
          docker tag $ECR_REGISTRY_URL:$VERSIONED_IMAGE_TAG $ECR_REGISTRY_URL:latest
          docker push $ECR_REGISTRY_URL:$VERSIONED_IMAGE_TAG
          docker push $ECR_REGISTRY_URL:latest
          
      # --- 3. Frontend Build & S3 Sync ---
      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'

      - name: Install frontend dependencies
        working-directory: ./frontend
        run: npm install

      - name: Create .env.production
        working-directory: ./frontend
        run: |
          echo "REACT_APP_API_URL=http://${{ steps.get_infra.outputs.ALB_DNS_NAME }}" > .env.production
          cat .env.production

      - name: Build frontend
        working-directory: ./frontend
        run: npm run build

      - name: Sync static website to S3
        run: |
          aws s3 sync ./frontend/build s3://${{ secrets.WEB_BUCKET_NAME }} --delete

      # --- 4. ECS Task Definition の更新とデプロイ ---

      # 現在のTask Definitionを取得
      - name: Download current task definition
        run: |
          aws ecs describe-task-definition \
            --task-definition ${{ steps.get_infra.outputs.TASK_DEFINITION_FAMILY }} \
            --query taskDefinition > task-definition.json

      # 新しいイメージタグに更新
      - name: Update task definition with new image
        id: update_task_def
        env:
          ECR_REGISTRY_URL: ${{ steps.get_infra.outputs.ECR_REPOSITORY_URL }}
          VERSIONED_IMAGE_TAG: ${{ steps.set_image_tag.outputs.IMAGE_TAG }}
        run: |
          cat task-definition.json | \
            jq --arg IMAGE "$ECR_REGISTRY_URL:$VERSIONED_IMAGE_TAG" \
            '.containerDefinitions[0].image = $IMAGE' | \
            jq 'del(.taskDefinitionArn, .revision, .status, .requiresAttributes, .compatibilities, .registeredAt, .registeredBy)' \
            > new-task-definition.json
          

      # 新しいTask Definitionを登録
      - name: Register new task definition
        id: register_task_def
        run: |
          TASK_DEF_ARN=$(aws ecs register-task-definition \
            --cli-input-json file://new-task-definition.json \
            --query 'taskDefinition.taskDefinitionArn' \
            --output text)
          echo "TASK_DEFINITION_ARN=$TASK_DEF_ARN" >> $GITHUB_OUTPUT

      # ECSサービスを新しいTask Definitionで更新
      - name: Update ECS service with new task definition
        run: |
          aws ecs update-service \
            --cluster ${{ steps.get_infra.outputs.ECS_CLUSTER_NAME }} \
            --service ${{ steps.get_infra.outputs.ECS_SERVICE_NAME }} \
            --task-definition ${{ steps.register_task_def.outputs.TASK_DEFINITION_ARN }} \
            --force-new-deployment

      # デプロイ完了を待機（オプション）
      - name: Wait for service stability
        run: |
          aws ecs wait services-stable \
            --cluster ${{ steps.get_infra.outputs.ECS_CLUSTER_NAME }} \
            --services ${{ steps.get_infra.outputs.ECS_SERVICE_NAME }}
```