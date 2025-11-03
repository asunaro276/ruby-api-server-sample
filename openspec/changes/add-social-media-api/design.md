# Social Media API 設計ドキュメント（サーバーレスアーキテクチャ）

## Context（コンテキスト）

XのようなSNSアプリケーションのバックエンドAPIを、AWS Lambda + API Gateway を使用したサーバーレスアーキテクチャで構築する。フロントエンドは別途作成される想定で、このプロジェクトではJSON APIのみを提供する。

### アーキテクチャ概要
```
[Client] → [API Gateway] → [Lambda Functions] → [RDS Aurora Serverless]
                                ↓
                            [S3 (画像)]
```

### 制約
- Ruby 3.4.7 を使用（Lambda ランタイム: ruby3.3 または カスタムランタイム）
- AWS Lambda の実行時間制限: 最大15分
- Lambda のメモリ制限: 128MB ～ 10,240MB
- API Gateway のタイムアウト: 29秒
- コールドスタート時の起動時間を考慮
- RESTful API 設計に従う

### ステークホルダー
- フロントエンド開発者（APIを使用する）
- エンドユーザー（SNSアプリを使用する）
- インフラエンジニア（AWS リソースの管理）

## Goals / Non-Goals（目標 / 非目標）

### 目標
- シンプルで理解しやすいREST API を提供
- JWT による安全な認証システムの実装
- 基本的なSNS機能（投稿、いいね、コメント）の提供
- スケーラブルなサーバーレスアーキテクチャの構築
- インフラコストの最適化（従量課金モデルの活用）
- 自動スケーリングによる高可用性の実現
- 適切なエラーハンドリングとバリデーション

### 非目標
- リアルタイム通知機能（将来の拡張として考慮、WebSocket API 使用）
- フォロー/フォロワー機能（フェーズ2として実装予定）
- ダイレクトメッセージ機能
- 複雑な検索機能（将来的に ElasticSearch 統合を検討）
- 管理者機能

## Decisions（決定事項）

### 1. サーバーレスアーキテクチャ: AWS Lambda + API Gateway

**理由:**
- **自動スケーリング**: トラフィックに応じて自動的にスケール
- **従量課金**: 使用した分だけ課金されるため、初期コストが低い
- **インフラ管理不要**: サーバーのメンテナンスやパッチ適用が不要
- **高可用性**: AWS のマネージドサービスによる高い可用性
- **開発速度**: インフラ管理の負担が少なく、アプリケーション開発に集中できる

**検討した代替案:**
- **EC2 + ALB**: 従来型、固定コストが高い、スケーリングが手動
- **ECS/Fargate**: コンテナベース、Lambda より柔軟だが管理コストが高い
- **Elastic Beanstalk**: PaaS、シンプルだがカスタマイズ性が低い

**実装詳細:**
- API Gateway: REST API（HTTP API も検討可能、コストが低い）
- Lambda 関数: Ruby 3.3 カスタムランタイム
- Lambda の設定:
  - メモリ: 1024MB（コールドスタート対策）
  - タイムアウト: 30秒（ほとんどのAPI は 5秒以内を目標）
  - 予約済み同時実行数: 最低5（コールドスタート対策）
  - Provisioned Concurrency: 本番環境で検討（コスト vs パフォーマンス）

**Lambda 関数の構成（ハイブリッドアプローチ - 機能グループごと）:**
```
lambda/
├── auth/                   # 認証機能グループ
│   ├── function.json       # lambroll設定
│   ├── handler.rb          # メインハンドラー（ルーティング）
│   └── controllers/        # 各エンドポイントのロジック
│       ├── register.rb     # POST /api/v1/auth/register
│       ├── login.rb        # POST /api/v1/auth/login
│       ├── logout.rb       # DELETE /api/v1/auth/logout
│       └── me.rb           # GET /api/v1/auth/me
├── posts/                  # 投稿機能グループ
│   ├── function.json
│   ├── handler.rb
│   └── controllers/
│       ├── index.rb        # GET /api/v1/posts
│       ├── show.rb         # GET /api/v1/posts/:id
│       ├── create.rb       # POST /api/v1/posts
│       ├── update.rb       # PUT /api/v1/posts/:id
│       ├── destroy.rb      # DELETE /api/v1/posts/:id
│       └── upload_url.rb   # POST /api/v1/posts/upload-url
├── likes/                  # いいね機能グループ
│   ├── function.json
│   ├── handler.rb
│   └── controllers/
│       ├── index.rb        # GET /api/v1/posts/:post_id/likes
│       ├── create.rb       # POST /api/v1/posts/:post_id/likes
│       └── destroy.rb      # DELETE /api/v1/posts/:post_id/likes
├── comments/               # コメント機能グループ
│   ├── function.json
│   ├── handler.rb
│   └── controllers/
│       ├── index.rb        # GET /api/v1/posts/:post_id/comments
│       ├── create.rb       # POST /api/v1/posts/:post_id/comments
│       └── destroy.rb      # DELETE /api/v1/comments/:id
├── authorizers/            # Lambda Authorizer
│   └── jwt/
│       ├── function.json
│       └── handler.rb
└── layers/                 # Lambda Layers（共通ライブラリ）
    ├── gems/               # Ruby gems（Sinatra等含む）
    │   ├── Gemfile
    │   ├── Gemfile.lock
    │   └── build.sh
    ├── models/             # ActiveRecord モデル
    │   └── ruby/lib/
    └── helpers/            # 共通ヘルパー
        └── ruby/lib/
```

**ハンドラーの実装例（API Gateway Proxy統合）:**
```ruby
# lambda/auth/handler.rb
require 'json'
require_relative 'controllers/register'
require_relative 'controllers/login'
require_relative 'controllers/logout'
require_relative 'controllers/me'

def handler(event:, context:)
  path = event['path']
  method = event['httpMethod']

  # ルーティング
  case [method, path]
  when ['POST', '/api/v1/auth/register']
    Controllers::Register.call(event)
  when ['POST', '/api/v1/auth/login']
    Controllers::Login.call(event)
  when ['DELETE', '/api/v1/auth/logout']
    Controllers::Logout.call(event)
  when ['GET', '/api/v1/auth/me']
    Controllers::Me.call(event)
  else
    {
      statusCode: 404,
      body: JSON.generate({ error: 'Not Found' })
    }
  end
rescue => e
  {
    statusCode: 500,
    body: JSON.generate({ error: e.message })
  }
end
```

**Lambda関数数の削減:**
- マイクロLambda: 15個 → ハイブリッド: **5個**（auth, posts, likes, comments, jwt_authorizer）
- 管理コストを削減しつつ、機能グループごとの独立性を維持

### 2. データベース: Amazon Aurora Serverless v2 (PostgreSQL)

**理由:**
- **サーバーレス**: 使用量に応じて自動スケーリング
- **コスト最適化**: アイドル時は最小容量まで縮小
- **PostgreSQL 互換**: Rails の ActiveRecord との親和性が高い
- **高可用性**: Multi-AZ 構成、自動バックアップ
- **パフォーマンス**: 従来の RDS より高速

**検討した代替案:**
- **RDS PostgreSQL**: 固定インスタンス、常時起動、コストが高い
- **DynamoDB**: NoSQL、スキーマレス、学習コストが高い、複雑なクエリに不向き
- **Aurora Serverless v1**: v2 より起動が遅い

**実装詳細:**
- エンジン: PostgreSQL 15.x
- 最小 ACU: 0.5（約 1GB RAM）
- 最大 ACU: 4（約 8GB RAM）
- VPC 内に配置、Lambda と同じ VPC
- セキュリティグループで Lambda からのアクセスのみ許可
- 接続管理: RDS Proxy を使用（接続プールの最適化）
- バックアップ: 自動バックアップ有効（7日間保持）

### 3. Lambda デプロイツール: lambroll

**理由:**
- **シンプル性**: Lambda関数のデプロイに特化した軽量ツール
- **Terraform連携**: Terraform state との連携が容易（tfstate template function）
- **柔軟な設定**: JSON または Jsonnet による設定ファイル
- **運用機能**: deploy, rollback, diff, logs など実用的なコマンド
- **学習コスト**: シンプルな設計で習得が容易

**選択: lambroll + Terraform**
- Terraform でインフラ（VPC, RDS, S3, IAM など）を管理
- lambroll で Lambda 関数のデプロイと運用を管理
- Terraform の出力を lambroll の設定で参照可能（tfstate連携）
- 関心の分離: インフラとアプリケーションの責務が明確

**検討した代替案:**
- **Serverless Framework**: フルスタックフレームワーク、多機能だが複雑
- **AWS SAM**: AWS 公式、CloudFormation ベース、冗長
- **Jets**: Rails ライク、学習コストは低いがカスタマイズ性が低い

**実装詳細:**

各機能グループごとにディレクトリと設定ファイルを配置：

```json
// lambda/auth/function.json
{
  "FunctionName": "social-media-api-auth-${env}",
  "Runtime": "ruby3.3",
  "Role": "{{ tfstate `aws_iam_role.lambda_exec.arn` }}",
  "Handler": "handler.handler",
  "MemorySize": 512,
  "Timeout": 15,
  "Environment": {
    "Variables": {
      "DB_HOST": "{{ tfstate `aws_rds_proxy.main.endpoint` }}",
      "DB_NAME": "{{ env `DB_NAME` }}",
      "JWT_SECRET_ARN": "{{ tfstate `aws_secretsmanager_secret.jwt_secret.arn` }}",
      "STAGE": "${env}"
    }
  },
  "VpcConfig": {
    "SubnetIds": "{{ tfstate `aws_subnet.private[*].id` | to_json }}",
    "SecurityGroupIds": "{{ tfstate `[aws_security_group.lambda.id]` | to_json }}"
  },
  "Layers": [
    "{{ tfstate `aws_lambda_layer_version.gems.arn` }}",
    "{{ tfstate `aws_lambda_layer_version.models.arn` }}",
    "{{ tfstate `aws_lambda_layer_version.helpers.arn` }}"
  ],
  "TracingConfig": {
    "Mode": "Active"
  }
}
```

```json
// lambda/posts/function.json
{
  "FunctionName": "social-media-api-posts-${env}",
  "Runtime": "ruby3.3",
  "Role": "{{ tfstate `aws_iam_role.lambda_exec.arn` }}",
  "Handler": "handler.handler",
  "MemorySize": 1024,
  "Timeout": 30,
  "Environment": {
    "Variables": {
      "DB_HOST": "{{ tfstate `aws_rds_proxy.main.endpoint` }}",
      "DB_NAME": "{{ env `DB_NAME` }}",
      "S3_BUCKET": "{{ tfstate `aws_s3_bucket.images.id` }}",
      "CLOUDFRONT_DOMAIN": "{{ tfstate `aws_cloudfront_distribution.main.domain_name` }}",
      "STAGE": "${env}"
    }
  },
  "VpcConfig": {
    "SubnetIds": "{{ tfstate `aws_subnet.private[*].id` | to_json }}",
    "SecurityGroupIds": "{{ tfstate `[aws_security_group.lambda.id]` | to_json }}"
  },
  "Layers": [
    "{{ tfstate `aws_lambda_layer_version.gems.arn` }}",
    "{{ tfstate `aws_lambda_layer_version.models.arn` }}",
    "{{ tfstate `aws_lambda_layer_version.helpers.arn` }}"
  ],
  "TracingConfig": {
    "Mode": "Active"
  }
}
```

**lambroll コマンド:**
```bash
# 機能グループごとのデプロイ
lambroll deploy --function auth
lambroll deploy --function posts

# 全関数のデプロイ
lambroll deploy --all

# デプロイ前の差分確認
lambroll diff --function auth

# ロールバック
lambroll rollback --function auth

# ログの確認
lambroll logs --function auth --follow

# Lambda関数の実行（API Gateway Proxy形式）
lambroll invoke --function auth --payload '{
  "httpMethod": "POST",
  "path": "/api/v1/auth/login",
  "body": "{\"email\":\"test@example.com\",\"password\":\"password123\"}"
}'
```

### 4. 認証システム: JWT (JSON Web Token)

**理由:**
- ステートレスな認証が可能で、Lambda に適している
- モバイルアプリやSPAからの利用に適している
- スケーラビリティが高い（セッションストアが不要）
- Lambda の stateless な性質と相性が良い

**実装詳細:**
- `jwt` gem を使用
- トークンの有効期限: 24時間
- リフレッシュトークン: DynamoDB に保存（オプション）
- パスワードハッシュ化: bcrypt
- トークン検証: API Gateway の Lambda Authorizer を使用

**Lambda Authorizer の実装:**
```ruby
# lambda/authorizers/jwt_authorizer.rb
def handler(event:, context:)
  token = event['authorizationToken']
  # JWT トークンを検証
  # ユーザー情報を返す
end
```

### 5. 画像ストレージ: AWS S3 + CloudFront

**理由:**
- スケーラブルで信頼性が高い
- Lambda から直接アクセス可能
- CloudFront CDN によるグローバルな高速配信
- Presigned URL による直接アップロード（Lambda の負荷軽減）

**実装詳細:**
- S3バケット: プライベート設定
- CloudFront: S3 へのアクセスを制限（OAI: Origin Access Identity）
- アップロード方式:
  1. クライアントが Lambda に画像アップロード用の Presigned URL をリクエスト
  2. Lambda が S3 Presigned URL を生成して返す
  3. クライアントが Presigned URL を使用して直接 S3 にアップロード
  4. アップロード完了後、Lambda に投稿作成リクエスト（画像キーを含む）
- 画像形式: JPEG, PNG, GIF をサポート
- 画像サイズ制限: 10MB
- ライフサイクルポリシー: 削除された投稿の画像を30日後に自動削除

### 6. インフラストラクチャ管理: Terraform

**理由:**
- Infrastructure as Code (IaC) により、インフラの構成を再現可能かつバージョン管理可能にする
- AWS リソース（Lambda, API Gateway, RDS, S3, IAM など）の一貫した管理が可能
- チーム間でのインフラ構成の共有と理解が容易
- 環境（開発、ステージング、本番）ごとのリソース管理が効率的
- Serverless Framework と併用可能

**検討した代替案:**
- **AWS Console 手動設定**: 簡単だが、再現性がなく、ドキュメント化が困難
- **AWS CloudFormation**: AWS ネイティブだが、YAML/JSON の記述が冗長
- **AWS CDK**: プログラマブルだが、学習コストが高い

**実装詳細:**
- Terraform 1.5以上を使用
- 以下のリソースをTerraformで管理:
  - **VPC とネットワーク**: VPC, Subnets, Security Groups, NAT Gateway
  - **RDS Aurora Serverless v2**: データベースクラスター、サブネットグループ
  - **RDS Proxy**: 接続プール管理
  - **S3バケット**: 画像ストレージ用
  - **CloudFront**: CDN 配信
  - **IAM ロール・ポリシー**: Lambda 実行ロール、最小権限の原則
  - **CloudWatch Logs**: ログ保存
  - **Lambda Layers**: 共通ライブラリ
  - **API Gateway**: REST API、ステージ、リソース、メソッド
- Terraform state はリモートバックエンド（S3 + DynamoDB）で管理
- 環境ごとに Terraform workspace を使用
- Terraform の出力値を lambroll の function.json で参照（tfstate template function）

**ディレクトリ構造:**
```
terraform/
├── main.tf                    # メインの設定
├── variables.tf               # 変数定義
├── outputs.tf                 # 出力値
├── terraform.tfvars          # 変数の値（gitignore）
└── modules/
    ├── vpc/                   # VPC、サブネット、セキュリティグループ
    ├── rds/                   # Aurora Serverless v2
    ├── s3/                    # S3バケット
    ├── cloudfront/            # CloudFront 配信
    ├── iam/                   # IAM ロール・ポリシー
    └── lambda-layers/         # Lambda Layers
```

**セキュリティ考慮事項:**
- Lambda は VPC 内に配置（RDS へのアクセス）
- RDS は private subnet に配置、パブリックアクセス不可
- S3バケットはプライベートに設定
- パブリックアクセスブロックを有効化
- バージョニングとログを有効化
- 暗号化（SSE-S3またはSSE-KMS）を有効化
- IAM ポリシーで最小権限の原則を適用
- API Gateway に API キー、スロットリング、WAF を設定

### 7. データベース設計

**User テーブル:**
- id (UUID, Primary Key)
- email (VARCHAR(255), UNIQUE, NOT NULL, INDEX)
- password_digest (VARCHAR(255), NOT NULL)
- username (VARCHAR(50), UNIQUE, NOT NULL, INDEX)
- bio (TEXT)
- avatar_url (VARCHAR(500))
- created_at (TIMESTAMP)
- updated_at (TIMESTAMP)

**Post テーブル:**
- id (UUID, Primary Key)
- user_id (UUID, Foreign Key, NOT NULL, INDEX)
- content (VARCHAR(280), NOT NULL)
- image_key (VARCHAR(500))  # S3 オブジェクトキー
- image_url (VARCHAR(500))  # CloudFront URL
- likes_count (INTEGER, DEFAULT 0)
- comments_count (INTEGER, DEFAULT 0)
- created_at (TIMESTAMP, INDEX)
- updated_at (TIMESTAMP)

**Like テーブル:**
- id (UUID, Primary Key)
- user_id (UUID, Foreign Key, NOT NULL)
- post_id (UUID, Foreign Key, NOT NULL)
- created_at (TIMESTAMP)
- UNIQUE INDEX (user_id, post_id)
- INDEX (post_id, created_at)

**Comment テーブル:**
- id (UUID, Primary Key)
- user_id (UUID, Foreign Key, NOT NULL)
- post_id (UUID, Foreign Key, NOT NULL)
- content (VARCHAR(280), NOT NULL)
- created_at (TIMESTAMP)
- updated_at (TIMESTAMP)
- INDEX (post_id, created_at)

### 8. API エンドポイント設計

すべてのエンドポイントは API Gateway 経由でアクセスされ、個別の Lambda 関数にルーティングされる。

**API Gateway ステージ:**
- `dev`: 開発環境
- `staging`: ステージング環境
- `prod`: 本番環境

**エンドポイント形式:** `https://{api-id}.execute-api.{region}.amazonaws.com/{stage}/api/v1/...`

**認証関連:**
- POST /api/v1/auth/register - ユーザー登録
- POST /api/v1/auth/login - ログイン
- DELETE /api/v1/auth/logout - ログアウト（トークンの無効化）
- GET /api/v1/auth/me - 現在のユーザー情報取得（認証必須）

**投稿関連:**
- GET /api/v1/posts - 投稿一覧取得（ページネーション）
- POST /api/v1/posts - 投稿作成（認証必須）
- GET /api/v1/posts/:id - 投稿詳細取得
- PUT /api/v1/posts/:id - 投稿更新（認証必須、権限チェック）
- DELETE /api/v1/posts/:id - 投稿削除（認証必須、権限チェック）
- POST /api/v1/posts/upload-url - 画像アップロード用 Presigned URL 取得（認証必須）

**いいね関連:**
- POST /api/v1/posts/:post_id/likes - いいねを追加（認証必須）
- DELETE /api/v1/posts/:post_id/likes - いいねを削除（認証必須）
- GET /api/v1/posts/:post_id/likes - いいね一覧取得

**コメント関連:**
- GET /api/v1/posts/:post_id/comments - コメント一覧取得
- POST /api/v1/posts/:post_id/comments - コメント作成（認証必須）
- DELETE /api/v1/comments/:id - コメント削除（認証必須、権限チェック）

### 9. Lambda 共通ライブラリ (Lambda Layers)

Lambda Layers を使用して共通コードとライブラリを共有し、デプロイサイズとコールドスタートを最適化。

**Layer 構成:**
1. **Gems Layer**: Ruby gems (pg, jwt, bcrypt など)
2. **Models Layer**: ActiveRecord モデル
3. **Helpers Layer**: 共通ヘルパー関数（JWT 検証、エラーハンドリングなど）

**共通コード:**
```ruby
# layers/shared/lib/database.rb
require 'pg'
require 'active_record'

module Database
  def self.connect
    ActiveRecord::Base.establish_connection(
      adapter: 'postgresql',
      host: ENV['DB_HOST'],
      username: ENV['DB_USERNAME'],
      password: ENV['DB_PASSWORD'],
      database: ENV['DB_NAME']
    )
  end
end

# layers/shared/lib/jwt_helper.rb
module JwtHelper
  def self.encode(payload)
    JWT.encode(payload, ENV['JWT_SECRET'], 'HS256')
  end

  def self.decode(token)
    JWT.decode(token, ENV['JWT_SECRET'], true, { algorithm: 'HS256' })
  end
end
```

### 10. 環境変数管理

Lambda 関数の環境変数は AWS Systems Manager Parameter Store または AWS Secrets Manager で管理。

**環境変数:**
- `DB_HOST`: RDS Proxy エンドポイント
- `DB_USERNAME`: データベースユーザー名
- `DB_PASSWORD`: データベースパスワード（Secrets Manager から取得）
- `DB_NAME`: データベース名
- `JWT_SECRET`: JWT トークンの署名シークレット（Secrets Manager）
- `S3_BUCKET`: 画像ストレージバケット名
- `CLOUDFRONT_DOMAIN`: CloudFront ディストリビューション URL
- `STAGE`: dev/staging/prod

## Risks / Trade-offs（リスク / トレードオフ）

### リスク1: コールドスタート時の遅延

**説明:**
Lambda 関数が初回起動時（コールドスタート）に数秒かかる可能性がある。

**緩和策:**
- Provisioned Concurrency を使用（本番環境で検討、コスト増加）
- Lambda のメモリを増やす（1024MB 以上推奨）
- Lambda Layers でコードサイズを最小化
- 頻繁にアクセスされるエンドポイントは定期的に Ping（CloudWatch Events）
- Ruby のロード時間を最小化（必要なライブラリのみ require）

### リスク2: Lambda の実行時間制限（最大15分）

**説明:**
長時間かかる処理は Lambda では実行できない。

**緩和策:**
- ほとんどの API リクエストは 5秒以内を目標に設計
- 画像処理などの重い処理は S3 + Lambda (非同期) で処理
- タイムアウトが予想される処理は SQS + Lambda で非同期処理化

### リスク3: VPC Lambda のコールドスタート

**説明:**
VPC 内の Lambda は ENI (Elastic Network Interface) の作成に時間がかかり、コールドスタートがさらに遅くなる。

**緩和策:**
- Hyperplane ENI（AWS の最適化）を使用（自動的に適用される）
- RDS Proxy を使用して接続プールを管理
- 可能な限り VPC 外の Lambda を使用（例: 認証関連）

### リスク4: データベース接続の枯渇

**説明:**
Lambda が同時に大量起動すると、RDS への接続数が枯渇する可能性がある。

**緩和策:**
- RDS Proxy を必ず使用（接続プールを管理）
- Lambda の同時実行数を制限（Reserved Concurrency）
- Aurora Serverless v2 の ACU を適切に設定

### リスク5: JWT トークンの無効化が困難

**緩和策:**
- 短い有効期限（24時間）を設定
- トークンブラックリストを DynamoDB に保存（オプション）

### リスク6: API Gateway のタイムアウト（29秒）

**説明:**
API Gateway は 29秒でタイムアウトする。

**緩和策:**
- すべての API リクエストは 25秒以内に完了するように設計
- 長時間処理は非同期化（SQS + Lambda）

### トレードオフ1: コスト vs パフォーマンス

**トレードオフ:**
- **Provisioned Concurrency**: コールドスタートを回避できるが、コストが高い
- **オンデマンド**: コストは低いが、コールドスタートが発生する

**推奨:**
- 開発環境: オンデマンド
- 本番環境: ピーク時のみ Provisioned Concurrency

### トレードオフ2: Lambda 関数の粒度

**選択: ハイブリッドアプローチ（機能グループごとに Lambda 関数を分割）**

**Lambda関数構成:**
- auth_function: 認証関連（4エンドポイント）
- posts_function: 投稿関連（6エンドポイント）
- likes_function: いいね関連（3エンドポイント）
- comments_function: コメント関連（3エンドポイント）
- jwt_authorizer: JWT認証（1関数）
- **合計: 5個の Lambda 関数**

**メリット（マイクロLambdaとモノリシックLambdaの良いとこ取り）:**
- **管理コスト削減**: 15個→5個に削減、デプロイが簡素化
- **機能グループごとの独立性**: 認証機能と投稿機能は独立してスケール
- **適度なコードサイズ**: 各関数 10-20MB 程度、コールドスタート 1-2秒
- **柔軟な設定**: posts_function だけメモリ/タイムアウトを大きく設定可能
- **障害の局所化**: 投稿機能のバグが認証機能に影響しない
- **共通コード共有**: 機能グループ内ではコードを直接共有、Lambda Layers も併用
- **ローカル開発**: 機能グループごとに開発・テスト可能

**デメリット:**
- **エンドポイント間の独立性低下**: 同じ関数内のエンドポイントは一緒にデプロイ
- **ルーティング実装**: 各関数内でルーティングロジックが必要（API Gateway Proxyパターン）
- **コールドスタート**: マイクロLambdaより若干遅い（機能グループ分のコードをロード）

**検討した他の選択肢:**
- **マイクロLambda（1エンドポイント = 1関数）**: 15個、管理コストが高い
- **モノリシックLambda（1関数）**: コールドスタート遅い、障害影響範囲大

### トレードオフ3: Aurora Serverless v2 vs RDS

**選択: Aurora Serverless v2**

**理由:**
- 自動スケーリング
- コスト最適化

**デメリット:**
- 最小容量でも課金される
- コールドスタート時の接続遅延（RDS Proxy で緩和）

## Migration Plan（移行計画）

既存のシステムがないため、新規実装のみ。

### フェーズ1: インフラストラクチャのセットアップ
1. Terraform で VPC、サブネット、セキュリティグループを構築
2. Aurora Serverless v2 をセットアップ
3. RDS Proxy を設定
4. S3 バケットと CloudFront を構築
5. Lambda Layers を作成（Gems, Models, Helpers）

### フェーズ2: 認証機能の実装
1. データベーススキーマ（User テーブル）を作成
2. User モデルを実装
3. Lambda 関数（register, login, logout, me）を実装
4. Lambda Authorizer を実装
5. API Gateway にエンドポイントを追加
6. テストを作成

### フェーズ3: 投稿機能の実装
1. Post テーブルを作成
2. Post モデルを実装
3. Lambda 関数（posts CRUD）を実装
4. 画像アップロード（Presigned URL）を実装
5. API Gateway にエンドポイントを追加
6. テストを作成

### フェーズ4: いいね・コメント機能の実装
1. Like, Comment テーブルを作成
2. Like, Comment モデルを実装
3. Lambda 関数（likes, comments）を実装
4. カウンターキャッシュの実装
5. API Gateway にエンドポイントを追加
6. テストを作成

### フェーズ5: 監視とログの設定
1. CloudWatch Logs を設定
2. X-Ray トレーシングを有効化
3. CloudWatch Alarms を設定（エラー率、レイテンシ）
4. ダッシュボードを作成

### ロールバック計画
- Terraform による Infrastructure as Code でロールバックが容易
- データベースマイグレーションのロールバックが可能
- Lambda のバージョニングとエイリアスによるロールバック
- API Gateway のステージごとのデプロイ

## Open Questions（未解決の質問）

1. **Provisioned Concurrency を使用するか？**
   - 本番環境でのみ使用を検討
   - コスト vs パフォーマンスのトレードオフ

2. **ページネーションの方式は？**
   - Offset ベース vs Cursor ベース
   - Aurora Serverless v2 のパフォーマンスを考慮して決定

3. **CORS設定はどうするか？**
   - API Gateway で CORS を有効化
   - 開発環境では全てのオリジンを許可
   - 本番環境では特定のオリジンのみを許可

4. **APIドキュメントの生成方法は？**
   - OpenAPI (Swagger) 仕様を作成
   - API Gateway から OpenAPI 仕様をエクスポート

5. **テストフレームワークは？**
   - RSpec でユニットテスト
   - Localstack でローカル統合テスト
   - Postman/Newman で E2E テスト

6. **モニタリングとアラートは？**
   - CloudWatch Metrics + Alarms
   - X-Ray トレーシング
   - ログ分析（CloudWatch Logs Insights）

7. **災害復旧（DR）計画は？**
   - RDS の自動バックアップ（7日間）
   - S3 のバージョニング
   - クロスリージョンレプリケーション（将来検討）
