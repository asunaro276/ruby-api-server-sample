# Social Media API 実装タスク（サーバーレスアーキテクチャ）

## 1. 環境準備とツールのインストール
- [ ] 1.1 Terraform をインストール（terraform 1.5以上）
- [ ] 1.2 AWS CLI をインストールし、認証情報を設定
- [ ] 1.3 lambroll をインストール（`brew install fujiwara/tap/lambroll` または GitHub Releases からダウンロード）
- [ ] 1.4 Ruby 3.3 をインストール（ローカル開発用）
- [ ] 1.5 PostgreSQL クライアントをインストール（ローカル開発用）
- [ ] 1.6 Docker をインストール（Lambda Layer ビルド、Localstack でのローカルテスト用）

## 2. Terraform によるインフラストラクチャのセットアップ

### 2.1 Terraform プロジェクトの初期化
- [ ] 2.1.1 terraform/ ディレクトリを作成
- [ ] 2.1.2 .gitignore に terraform 関連ファイルを追加（*.tfstate, .terraform/, terraform.tfvars）
- [ ] 2.1.3 terraform/main.tf を作成（プロバイダー設定）
- [ ] 2.1.4 terraform/variables.tf を作成
- [ ] 2.1.5 terraform/outputs.tf を作成
- [ ] 2.1.6 terraform/terraform.tfvars を作成（.gitignore に追加済み）

### 2.2 VPC とネットワークの構築
- [ ] 2.2.1 terraform/modules/vpc/ モジュールを作成
- [ ] 2.2.2 VPC を定義（CIDR: 10.0.0.0/16）
- [ ] 2.2.3 パブリックサブネット × 2（Multi-AZ）を作成
- [ ] 2.2.4 プライベートサブネット × 2（Multi-AZ）を作成
- [ ] 2.2.5 インターネットゲートウェイを作成
- [ ] 2.2.6 NAT Gateway を作成（パブリックサブネット内）
- [ ] 2.2.7 ルートテーブルを設定
- [ ] 2.2.8 セキュリティグループを作成
  - [ ] Lambda 用セキュリティグループ
  - [ ] RDS 用セキュリティグループ（Lambda からのアクセスのみ許可）

### 2.3 Aurora Serverless v2 + RDS Proxy の構築
- [ ] 2.3.1 terraform/modules/rds/ モジュールを作成
- [ ] 2.3.2 Aurora Serverless v2 クラスターを定義
  - [ ] エンジン: PostgreSQL 15.x
  - [ ] 最小 ACU: 0.5
  - [ ] 最大 ACU: 4
  - [ ] Multi-AZ 構成
- [ ] 2.3.3 RDS サブネットグループを作成
- [ ] 2.3.4 データベースパスワードを Secrets Manager に保存
- [ ] 2.3.5 RDS Proxy を作成
  - [ ] 接続プール設定
  - [ ] Secrets Manager からのパスワード取得
- [ ] 2.3.6 RDS Proxy のエンドポイントを outputs に追加

### 2.4 S3 バケット + CloudFront の構築
- [ ] 2.4.1 terraform/modules/s3/ モジュールを作成
- [ ] 2.4.2 S3バケットを作成（画像ストレージ用）
  - [ ] バケット名（ユニーク性を考慮）
  - [ ] プライベートアクセス設定
  - [ ] パブリックアクセスブロックの有効化
  - [ ] バージョニングの有効化
  - [ ] 暗号化の有効化（SSE-S3）
  - [ ] CORS設定
- [ ] 2.4.3 ライフサイクルポリシーを設定（削除マーク付きオブジェクトを30日後に削除）
- [ ] 2.4.4 terraform/modules/cloudfront/ モジュールを作成
- [ ] 2.4.5 CloudFront ディストリビューションを作成
  - [ ] Origin Access Identity (OAI) を作成
  - [ ] S3 バケットポリシーを更新（OAI からのアクセスのみ許可）
  - [ ] キャッシュポリシーを設定
- [ ] 2.4.6 CloudFront URL を outputs に追加

### 2.5 IAM ロール・ポリシーの作成
- [ ] 2.5.1 terraform/modules/iam/ モジュールを作成
- [ ] 2.5.2 Lambda 実行ロールを作成
  - [ ] AWSLambdaBasicExecutionRole をアタッチ
  - [ ] AWSLambdaVPCAccessExecutionRole をアタッチ
- [ ] 2.5.3 Lambda 用の IAM ポリシーを作成（最小権限）
  - [ ] RDS Proxy への接続権限
  - [ ] S3 への読み書き権限（特定バケットのみ）
  - [ ] Secrets Manager への読み取り権限
  - [ ] CloudWatch Logs への書き込み権限
  - [ ] X-Ray への書き込み権限
- [ ] 2.5.4 Lambda 実行ロールに IAM ポリシーをアタッチ

### 2.6 Secrets Manager の設定
- [ ] 2.6.1 JWT_SECRET を Secrets Manager に保存
- [ ] 2.6.2 DB_PASSWORD を Secrets Manager に保存（既に作成済みなら確認）

### 2.7 Terraform の実行
- [ ] 2.7.1 terraform init を実行
- [ ] 2.7.2 terraform plan を実行して変更内容を確認
- [ ] 2.7.3 terraform apply を実行してリソースを作成
- [ ] 2.7.4 terraform output を実行して出力値を取得
  - [ ] VPC ID
  - [ ] サブネット IDs
  - [ ] セキュリティグループ IDs
  - [ ] RDS Proxy エンドポイント
  - [ ] S3 バケット名
  - [ ] CloudFront URL
  - [ ] Lambda 実行ロール ARN

## 3. データベーススキーマの作成

- [ ] 3.1 ローカルから RDS Proxy 経由で Aurora に接続
- [ ] 3.2 データベース migrations/ ディレクトリを作成
- [ ] 3.3 User テーブルの作成スクリプトを作成
  ```sql
  CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    password_digest VARCHAR(255) NOT NULL,
    username VARCHAR(50) UNIQUE NOT NULL,
    bio TEXT,
    avatar_url VARCHAR(500),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
  );
  CREATE INDEX idx_users_email ON users(email);
  CREATE INDEX idx_users_username ON users(username);
  ```
- [ ] 3.4 Post テーブルの作成スクリプトを作成
  ```sql
  CREATE TABLE posts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    content VARCHAR(280) NOT NULL,
    image_key VARCHAR(500),
    image_url VARCHAR(500),
    likes_count INTEGER DEFAULT 0,
    comments_count INTEGER DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
  );
  CREATE INDEX idx_posts_user_id ON posts(user_id);
  CREATE INDEX idx_posts_created_at ON posts(created_at DESC);
  ```
- [ ] 3.5 Like テーブルの作成スクリプトを作成
  ```sql
  CREATE TABLE likes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    post_id UUID NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE (user_id, post_id)
  );
  CREATE INDEX idx_likes_post_id ON likes(post_id, created_at DESC);
  ```
- [ ] 3.6 Comment テーブルの作成スクリプトを作成
  ```sql
  CREATE TABLE comments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    post_id UUID NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
    content VARCHAR(280) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
  );
  CREATE INDEX idx_comments_post_id ON comments(post_id, created_at);
  ```
- [ ] 3.7 マイグレーションを実行

## 4. lambroll プロジェクトのセットアップ

- [ ] 4.1 lambda/ ディレクトリ構造を作成
  ```
  lambda/
  ├── auth/
  │   ├── register/
  │   ├── login/
  │   ├── logout/
  │   └── me/
  ├── posts/
  │   ├── index/
  │   ├── show/
  │   ├── create/
  │   ├── update/
  │   └── destroy/
  ├── likes/
  │   ├── index/
  │   ├── create/
  │   └── destroy/
  ├── comments/
  │   ├── index/
  │   ├── create/
  │   └── destroy/
  ├── authorizers/
  │   └── jwt/
  └── layers/
      ├── gems/
      ├── models/
      └── helpers/
  ```
- [ ] 4.2 各Lambda関数ディレクトリに function.json テンプレートを作成
  ```json
  {
    "FunctionName": "social-media-api-{function-name}-${env}",
    "Runtime": "ruby3.3",
    "Role": "{{ tfstate `aws_iam_role.lambda_exec.arn` }}",
    "Handler": "index.handler",
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
    ]
  }
  ```
- [ ] 4.3 .gitignore に lambroll 関連ファイルを追加（*.zip, .lambroll/）
- [ ] 4.4 .lambroll.yml を作成（lambroll のグローバル設定、オプション）

## 5. Lambda Layers の作成

### 5.1 Gems Layer
- [ ] 5.1.1 lambda/layers/gems/ ディレクトリを作成
- [ ] 5.1.2 Gemfile を作成（pg, jwt, bcrypt, activerecord など）
- [ ] 5.1.3 build.sh を作成（Docker で Ruby 3.3 環境を構築し、gem をインストール）
- [ ] 5.1.4 Lambda Layer としてパッケージ化（ruby/gems/ ディレクトリ構造）
- [ ] 5.1.5 Terraform で Lambda Layer リソースを定義し、デプロイ

### 5.2 Models Layer
- [ ] 5.2.1 lambda/layers/models/ ディレクトリを作成
- [ ] 5.2.2 database.rb を作成（ActiveRecord 接続設定）
- [ ] 5.2.3 User モデルを作成
  - [ ] バリデーション（email, username, password）
  - [ ] アソシエーション（has_many :posts, :likes, :comments）
  - [ ] has_secure_password（bcrypt）
- [ ] 5.2.4 Post モデルを作成
  - [ ] バリデーション（content: 必須、最大280文字）
  - [ ] アソシエーション（belongs_to :user, has_many :likes, :comments）
- [ ] 5.2.5 Like モデルを作成
  - [ ] バリデーション（uniqueness: { scope: [:user_id, :post_id] }）
  - [ ] アソシエーション（belongs_to :user, :post）
- [ ] 5.2.6 Comment モデルを作成
  - [ ] バリデーション（content: 必須、最大280文字）
  - [ ] アソシエーション（belongs_to :user, :post）
- [ ] 5.2.7 Lambda Layer としてパッケージ化

### 5.3 Helpers Layer
- [ ] 5.3.1 lambda/layers/helpers/ ディレクトリを作成
- [ ] 5.3.2 jwt_helper.rb を作成（JWT エンコード・デコード）
- [ ] 5.3.3 response_helper.rb を作成（JSON レスポンスの統一フォーマット）
- [ ] 5.3.4 error_handler.rb を作成（エラーハンドリング）
- [ ] 5.3.5 Lambda Layer としてパッケージ化

## 6. Lambda Authorizer の実装

- [ ] 6.1 lambda/authorizers/jwt/ ディレクトリを作成
- [ ] 6.2 function.json を作成（Lambda Authorizer 設定）
- [ ] 6.3 index.rb を実装
  - [ ] Authorization ヘッダーから JWT トークンを取得
  - [ ] JWT トークンを検証（Secrets Manager から JWT_SECRET を取得）
  - [ ] ユーザー情報を返す（IAM ポリシーと共に）
- [ ] 6.4 lambroll deploy でデプロイ
- [ ] 6.5 Terraform で API Gateway Authorizer リソースを作成し、Lambda Authorizer を接続
- [ ] 6.6 テストを作成

## 7. 認証 Lambda 関数の実装

- [ ] 7.1 lambda/auth/register/ ディレクトリを作成
- [ ] 7.2 function.json と index.rb を作成（POST /api/v1/auth/register）
  - [ ] リクエストボディからユーザー情報を取得
  - [ ] バリデーション
  - [ ] User レコードを作成
  - [ ] JWT トークンを生成
  - [ ] レスポンスを返す（ステータスコード 201）
- [ ] 7.3 lambda/auth/login/ ディレクトリを作成
- [ ] 7.4 function.json と index.rb を作成（POST /api/v1/auth/login）
  - [ ] email と password を取得
  - [ ] ユーザーを検証
  - [ ] JWT トークンを生成
  - [ ] レスポンスを返す（ステータスコード 200）
- [ ] 7.5 lambda/auth/logout/ ディレクトリを作成
- [ ] 7.6 function.json と index.rb を作成（DELETE /api/v1/auth/logout）
  - [ ] トークンブラックリスト機能（オプション、DynamoDB 使用）
  - [ ] レスポンスを返す（ステータスコード 200）
- [ ] 7.7 lambda/auth/me/ ディレクトリを作成
- [ ] 7.8 function.json と index.rb を作成（GET /api/v1/auth/me）
  - [ ] Authorizer から user_id を取得
  - [ ] User レコードを取得
  - [ ] レスポンスを返す（ステータスコード 200）
- [ ] 7.9 lambroll deploy で各 Lambda 関数をデプロイ
- [ ] 7.10 Terraform で API Gateway のリソース、メソッド、統合を作成し、Lambda 関数を接続

## 8. 投稿 Lambda 関数の実装

- [ ] 8.1 lambda/posts/index/ に function.json と index.rb を作成（GET /api/v1/posts）
  - [ ] ページネーション処理（offset ベース）
  - [ ] Post 一覧を取得（N+1クエリ対策）
  - [ ] レスポンスを返す（ステータスコード 200）
- [ ] 8.2 lambda/posts/show/ に function.json と index.rb を作成（GET /api/v1/posts/:id）
  - [ ] id パラメータから Post を取得
  - [ ] is_liked フラグを含める（認証済みユーザーの場合）
  - [ ] レスポンスを返す（ステータスコード 200）
- [ ] 8.3 lambda/posts/create/ に function.json と index.rb を作成（POST /api/v1/posts）
  - [ ] Authorizer から user_id を取得
  - [ ] リクエストボディから投稿内容を取得
  - [ ] 画像キーがある場合、CloudFront URL を生成
  - [ ] Post レコードを作成
  - [ ] レスポンスを返す（ステータスコード 201）
- [ ] 8.4 lambda/posts/update/ に function.json と index.rb を作成（PUT /api/v1/posts/:id）
  - [ ] 権限チェック（自分の投稿のみ）
  - [ ] Post レコードを更新
  - [ ] レスポンスを返す（ステータスコード 200）
- [ ] 8.5 lambda/posts/destroy/ に function.json と index.rb を作成（DELETE /api/v1/posts/:id）
  - [ ] 権限チェック（自分の投稿のみ）
  - [ ] Post レコードを削除（cascade で likes, comments も削除）
  - [ ] S3 から画像を削除
  - [ ] レスポンスを返す（ステータスコード 200）
- [ ] 8.6 lambda/posts/upload_url/ に function.json と index.rb を作成（POST /api/v1/posts/upload-url）
  - [ ] Authorizer から user_id を取得
  - [ ] S3 Presigned URL を生成（PutObject 権限、10分間有効）
  - [ ] レスポンスを返す（presigned_url, image_key）
- [ ] 8.7 lambroll deploy で各 Lambda 関数をデプロイ
- [ ] 8.8 Terraform で API Gateway エンドポイントを設定（Authorizer を適用）

## 9. いいね Lambda 関数の実装

- [ ] 9.1 lambda/likes/index/ に function.json と index.rb を作成（GET /api/v1/posts/:post_id/likes）
  - [ ] post_id から Like 一覧を取得
  - [ ] ユーザー情報を含める
  - [ ] レスポンスを返す（ステータスコード 200）
- [ ] 9.2 lambda/likes/create/ に function.json と index.rb を作成（POST /api/v1/posts/:post_id/likes）
  - [ ] Authorizer から user_id を取得
  - [ ] 重複チェック
  - [ ] Like レコードを作成
  - [ ] Post の likes_count を +1
  - [ ] レスポンスを返す（ステータスコード 201）
- [ ] 9.3 lambda/likes/destroy/ に function.json と index.rb を作成（DELETE /api/v1/posts/:post_id/likes）
  - [ ] Authorizer から user_id を取得
  - [ ] Like レコードを削除
  - [ ] Post の likes_count を -1
  - [ ] レスポンスを返す（ステータスコード 200）
- [ ] 9.4 lambroll deploy で各 Lambda 関数をデプロイ
- [ ] 9.5 Terraform で API Gateway エンドポイントを設定（Authorizer を適用）

## 10. コメント Lambda 関数の実装

- [ ] 10.1 lambda/comments/index/ に function.json と index.rb を作成（GET /api/v1/posts/:post_id/comments）
  - [ ] post_id から Comment 一覧を取得（古い順）
  - [ ] ページネーション処理
  - [ ] ユーザー情報を含める
  - [ ] レスポンスを返す（ステータスコード 200）
- [ ] 10.2 lambda/comments/create/ に function.json と index.rb を作成（POST /api/v1/posts/:post_id/comments）
  - [ ] Authorizer から user_id を取得
  - [ ] バリデーション
  - [ ] Comment レコードを作成
  - [ ] Post の comments_count を +1
  - [ ] レスポンスを返す（ステータスコード 201）
- [ ] 10.3 lambda/comments/destroy/ に function.json と index.rb を作成（DELETE /api/v1/comments/:id）
  - [ ] 権限チェック（自分のコメントのみ）
  - [ ] Comment レコードを削除
  - [ ] Post の comments_count を -1
  - [ ] レスポンスを返す（ステータスコード 200）
- [ ] 10.4 lambroll deploy で各 Lambda 関数をデプロイ
- [ ] 10.5 Terraform で API Gateway エンドポイントを設定（Authorizer を適用）

## 11. エラーハンドリングとバリデーション

- [ ] 11.1 共通エラーハンドラーを Helpers Layer に追加
- [ ] 11.2 カスタムエラーレスポンスを定義（JSON 形式）
- [ ] 11.3 各 Lambda 関数にエラーハンドリングを追加
  - [ ] 404: Not Found
  - [ ] 401: Unauthorized
  - [ ] 403: Forbidden
  - [ ] 422: Unprocessable Entity
  - [ ] 500: Internal Server Error
- [ ] 11.4 バリデーションエラーメッセージのフォーマット統一

## 12. API Gateway の設定

- [ ] 12.1 Terraform で CORS を有効化（Gateway Response の設定）
- [ ] 12.2 API キーを設定（Terraform、オプション）
- [ ] 12.3 スロットリング設定を追加（Terraform、レート制限）
  - [ ] リクエストレート: 100 req/sec
  - [ ] バーストレート: 200 req
- [ ] 12.4 ステージを作成（Terraform、dev, staging, prod）
- [ ] 12.5 カスタムドメイン設定（Terraform、オプション、Route 53 + ACM 証明書）
- [ ] 12.6 デプロイメントを作成し、ステージに紐付け

## 13. CloudWatch Logs + X-Ray の設定

- [ ] 13.1 CloudWatch Logs を有効化（Lambda 作成時に自動的に作成される）
- [ ] 13.2 X-Ray トレーシングを有効化
  - [ ] Lambda の function.json に `"TracingConfig": {"Mode": "Active"}` を追加
  - [ ] Terraform で API Gateway のトレーシングを有効化
  - [ ] IAM ロールに X-Ray 書き込み権限を追加
- [ ] 13.3 Terraform で CloudWatch Alarms を作成
  - [ ] Lambda エラー率 > 5%
  - [ ] API Gateway 4xx エラー率 > 10%
  - [ ] API Gateway 5xx エラー率 > 1%
  - [ ] Lambda Duration > 5秒
- [ ] 13.4 Terraform で CloudWatch ダッシュボードを作成

## 14. テストの作成

- [ ] 14.1 RSpec をセットアップ
- [ ] 14.2 ユーザー認証のテスト
  - [ ] register のユニットテスト
  - [ ] login のユニットテスト
  - [ ] バリデーションのテスト
- [ ] 14.3 投稿機能のテスト
  - [ ] CRUD操作のテスト
  - [ ] 権限チェックのテスト
  - [ ] 画像アップロードのテスト
- [ ] 14.4 いいね機能のテスト
  - [ ] いいね追加・削除のテスト
  - [ ] カウンター更新のテスト
- [ ] 14.5 コメント機能のテスト
  - [ ] コメント追加・削除のテスト
  - [ ] カウンター更新のテスト
- [ ] 14.6 Localstack で統合テスト（オプション）
- [ ] 14.7 E2E テスト（Postman/Newman）

## 15. Seedデータの作成

- [ ] 15.1 seeds/ ディレクトリを作成
- [ ] 15.2 seed.rb を作成（サンプルユーザー、投稿、いいね、コメント）
- [ ] 15.3 seed.rb を実行してデータを投入

## 16. デプロイとドキュメント

- [ ] 16.1 Terraform で開発環境のインフラをデプロイ（`terraform apply`）
- [ ] 16.2 lambroll で全Lambda関数をデプロイ（`lambroll deploy --all`）
- [ ] 16.3 API Gateway のエンドポイント URL を取得（Terraform output）
- [ ] 16.4 README.md を更新
  - [ ] アーキテクチャ図を追加
  - [ ] API エンドポイント一覧を記載
  - [ ] 認証方法（JWT）の使い方を記載
  - [ ] Terraform によるインフラセットアップ手順を記載
  - [ ] lambroll によるLambdaデプロイ手順を記載
  - [ ] 環境変数の設定方法を記載
  - [ ] ローカル開発のセットアップ手順を記載
- [ ] 16.5 OpenAPI (Swagger) 仕様を生成
  - [ ] API Gateway から OpenAPI 仕様をエクスポート（AWS CLI または Terraform）
  - [ ] Swagger UI で表示できるように設定

## 17. 最終確認とテスト

- [ ] 17.1 全ての Lambda 関数の動作確認
- [ ] 17.2 API Gateway のエンドポイント確認
- [ ] 17.3 Postman で API テストを実行
- [ ] 17.4 CloudWatch Logs でログを確認
- [ ] 17.5 X-Ray でトレースを確認
- [ ] 17.6 CloudWatch Alarms が正しく設定されているか確認
- [ ] 17.7 S3 への画像アップロードの動作確認
- [ ] 17.8 CloudFront 経由での画像配信確認
- [ ] 17.9 Aurora Serverless v2 のスケーリング確認
- [ ] 17.10 Lambda のコールドスタート時間を計測
- [ ] 17.11 セキュリティチェック
  - [ ] IAM ポリシーの最小権限確認
  - [ ] VPC 設定の確認
  - [ ] S3 バケットのパブリックアクセスブロック確認
  - [ ] Secrets Manager の認証情報確認

## 18. 本番環境へのデプロイ

- [ ] 18.1 本番環境用の terraform workspace を作成
- [ ] 18.2 terraform apply --workspace=prod を実行（本番インフラのデプロイ）
- [ ] 18.3 本番環境用の環境変数を設定（env=prod）
- [ ] 18.4 lambroll deploy --all を実行（本番環境のLambda関数をデプロイ）
- [ ] 18.5 本番環境の動作確認
- [ ] 18.6 監視とアラートの設定を確認
- [ ] 18.7 バックアップの設定を確認（RDS 自動バックアップ）
