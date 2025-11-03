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

## 4. lambroll プロジェクトのセットアップ（ハイブリッドアプローチ）

- [ ] 4.1 lambda/ ディレクトリ構造を作成（機能グループごと）
  ```
  lambda/
  ├── auth/                   # 認証機能グループ
  │   ├── function.json
  │   ├── handler.rb
  │   └── controllers/
  │       ├── register.rb
  │       ├── login.rb
  │       ├── logout.rb
  │       └── me.rb
  ├── posts/                  # 投稿機能グループ
  │   ├── function.json
  │   ├── handler.rb
  │   └── controllers/
  │       ├── index.rb
  │       ├── show.rb
  │       ├── create.rb
  │       ├── update.rb
  │       ├── destroy.rb
  │       └── upload_url.rb
  ├── likes/                  # いいね機能グループ
  │   ├── function.json
  │   ├── handler.rb
  │   └── controllers/
  │       ├── index.rb
  │       ├── create.rb
  │       └── destroy.rb
  ├── comments/               # コメント機能グループ
  │   ├── function.json
  │   ├── handler.rb
  │   └── controllers/
  │       ├── index.rb
  │       ├── create.rb
  │       └── destroy.rb
  ├── authorizers/
  │   └── jwt/
  │       ├── function.json
  │       └── handler.rb
  └── layers/
      ├── gems/
      ├── models/
      └── helpers/
  ```
- [ ] 4.2 各機能グループに function.json を作成
  - [ ] lambda/auth/function.json（MemorySize: 512, Timeout: 15）
  - [ ] lambda/posts/function.json（MemorySize: 1024, Timeout: 30、S3権限必要）
  - [ ] lambda/likes/function.json（MemorySize: 512, Timeout: 15）
  - [ ] lambda/comments/function.json（MemorySize: 512, Timeout: 15）
  - [ ] lambda/authorizers/jwt/function.json（MemorySize: 256, Timeout: 10）
- [ ] 4.3 各機能グループに handler.rb を作成（API Gateway Proxyパターン）
  - [ ] ルーティングロジックを実装（httpMethod + path による分岐）
  - [ ] エラーハンドリングを実装
- [ ] 4.4 .gitignore に lambroll 関連ファイルを追加（*.zip, .lambroll/）
- [ ] 4.5 .lambroll.yml を作成（lambroll のグローバル設定、オプション）

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

## 6. テスト環境のセットアップ

- [ ] 6.1 RSpec をセットアップ
  - [ ] Gemfile に rspec, rspec-mocks, webmock を追加
  - [ ] spec/spec_helper.rb を作成
  - [ ] spec/support/ ディレクトリを作成（共通ヘルパー用）
- [ ] 6.2 テスト用の Lambda イベントモックを作成
  - [ ] spec/support/lambda_event_helper.rb（API Gateway Proxy イベントのモック）
- [ ] 6.3 テスト用のデータベース設定
  - [ ] spec/support/database_helper.rb（テスト用DB接続、トランザクション）
- [ ] 6.4 ローカルテスト環境の構築
  - [ ] Docker Compose で PostgreSQL コンテナを起動（ローカルテスト用）
  - [ ] テスト用の環境変数を設定（.env.test）

## 7. Lambda Authorizer の実装

- [ ] 7.1 Lambda Authorizer のテスト作成（spec/authorizers/jwt_spec.rb）
  - [ ] 有効なJWTトークンで認証成功のテスト
  - [ ] 無効なJWTトークンで認証失敗のテスト
  - [ ] トークンなしで認証失敗のテスト
  - [ ] IAM ポリシー生成のテスト
- [ ] 7.2 lambda/authorizers/jwt/function.json を作成（Lambda Authorizer 設定）
- [ ] 7.3 lambda/authorizers/jwt/handler.rb を実装
  - [ ] Authorization ヘッダーから JWT トークンを取得
  - [ ] JWT トークンを検証（Secrets Manager から JWT_SECRET を取得）
  - [ ] ユーザー情報を返す（IAM ポリシーと共に）
- [ ] 7.4 テストを実行して全てパスすることを確認
- [ ] 7.5 lambroll deploy --function authorizers/jwt でデプロイ
- [ ] 7.6 Terraform で API Gateway Authorizer リソースを作成し、Lambda Authorizer を接続

## 8. 認証機能のテスト作成

- [ ] 8.1 ユーザー登録のテスト作成（spec/auth/register_spec.rb）
  - [ ] 有効な情報でユーザー登録成功のテスト
  - [ ] email重複でユーザー登録失敗のテスト
  - [ ] username重複でユーザー登録失敗のテスト
  - [ ] バリデーションエラー（email形式不正）のテスト
  - [ ] バリデーションエラー（password短すぎる）のテスト
  - [ ] JWTトークンが返されることのテスト
- [ ] 8.2 ログインのテスト作成（spec/auth/login_spec.rb）
  - [ ] 正しいemail/passwordでログイン成功のテスト
  - [ ] 間違ったpasswordでログイン失敗のテスト
  - [ ] 存在しないemailでログイン失敗のテスト
  - [ ] JWTトークンが返されることのテスト
- [ ] 8.3 ログアウトのテスト作成（spec/auth/logout_spec.rb）
  - [ ] 認証済みユーザーのログアウト成功のテスト
  - [ ] 未認証ユーザーのログアウト失敗のテスト
- [ ] 8.4 ユーザー情報取得のテスト作成（spec/auth/me_spec.rb）
  - [ ] 認証済みユーザーの情報取得成功のテスト
  - [ ] 未認証ユーザーの情報取得失敗のテスト
  - [ ] 削除されたユーザーの情報取得失敗のテスト

## 9. 認証 Lambda 関数の実装（auth 機能グループ）

- [ ] 9.1 lambda/auth/handler.rb を実装（ルーティングロジック）
  - [ ] httpMethod と path によるルーティング
  - [ ] POST /api/v1/auth/register → Controllers::Register.call
  - [ ] POST /api/v1/auth/login → Controllers::Login.call
  - [ ] DELETE /api/v1/auth/logout → Controllers::Logout.call
  - [ ] GET /api/v1/auth/me → Controllers::Me.call
  - [ ] エラーハンドリング（404, 500）
- [ ] 9.2 lambda/auth/controllers/register.rb を実装
  - [ ] リクエストボディからユーザー情報を取得
  - [ ] バリデーション
  - [ ] User レコードを作成
  - [ ] JWT トークンを生成
  - [ ] レスポンスを返す（ステータスコード 201）
  - [ ] **テストを実行してパスすることを確認**
- [ ] 9.3 lambda/auth/controllers/login.rb を実装
  - [ ] email と password を取得
  - [ ] ユーザーを検証
  - [ ] JWT トークンを生成
  - [ ] レスポンスを返す（ステータスコード 200）
  - [ ] **テストを実行してパスすることを確認**
- [ ] 9.4 lambda/auth/controllers/logout.rb を実装
  - [ ] トークンブラックリスト機能（オプション、DynamoDB 使用）
  - [ ] レスポンスを返す（ステータスコード 200）
  - [ ] **テストを実行してパスすることを確認**
- [ ] 9.5 lambda/auth/controllers/me.rb を実装
  - [ ] Authorizer から user_id を取得（event['requestContext']['authorizer']['principalId']）
  - [ ] User レコードを取得
  - [ ] レスポンスを返す（ステータスコード 200）
  - [ ] **テストを実行してパスすることを確認**
- [ ] 9.6 全テストを実行して全てパスすることを確認
- [ ] 9.7 lambroll deploy --function auth でデプロイ
- [ ] 9.8 Terraform で API Gateway の /api/v1/auth/{proxy+} リソースを作成
  - [ ] ANY メソッドで Lambda Proxy統合
  - [ ] Authorizer を me エンドポイントのみ適用（リソースポリシーで制御）

## 10. 投稿機能のテスト作成

- [ ] 10.1 投稿一覧取得のテスト作成（spec/posts/index_spec.rb）
  - [ ] 投稿一覧を取得成功のテスト
  - [ ] ページネーションのテスト
  - [ ] 空の投稿一覧のテスト
- [ ] 10.2 投稿詳細取得のテスト作成（spec/posts/show_spec.rb）
  - [ ] 投稿詳細を取得成功のテスト
  - [ ] 存在しない投稿IDで404エラーのテスト
  - [ ] is_likedフラグが正しく返されるテスト
- [ ] 10.3 投稿作成のテスト作成（spec/posts/create_spec.rb）
  - [ ] 有効な内容で投稿作成成功のテスト
  - [ ] 画像付き投稿作成成功のテスト
  - [ ] 空の内容で投稿作成失敗のテスト
  - [ ] 280文字超過で投稿作成失敗のテスト
  - [ ] 未認証で投稿作成失敗のテスト
- [ ] 10.4 投稿更新のテスト作成（spec/posts/update_spec.rb）
  - [ ] 自分の投稿の更新成功のテスト
  - [ ] 他人の投稿の更新失敗（403）のテスト
  - [ ] 未認証で投稿更新失敗のテスト
- [ ] 10.5 投稿削除のテスト作成（spec/posts/destroy_spec.rb）
  - [ ] 自分の投稿の削除成功のテスト
  - [ ] 他人の投稿の削除失敗（403）のテスト
  - [ ] S3から画像も削除されることのテスト
  - [ ] 未認証で投稿削除失敗のテスト
- [ ] 10.6 画像アップロードURLのテスト作成（spec/posts/upload_url_spec.rb）
  - [ ] Presigned URL生成成功のテスト
  - [ ] 未認証でPresigned URL生成失敗のテスト

## 11. 投稿 Lambda 関数の実装（posts 機能グループ）

- [ ] 11.1 lambda/posts/handler.rb を実装（ルーティングロジック）
  - [ ] httpMethod と path によるルーティング
  - [ ] GET /api/v1/posts → Controllers::Index.call
  - [ ] GET /api/v1/posts/:id → Controllers::Show.call
  - [ ] POST /api/v1/posts → Controllers::Create.call
  - [ ] PUT /api/v1/posts/:id → Controllers::Update.call
  - [ ] DELETE /api/v1/posts/:id → Controllers::Destroy.call
  - [ ] POST /api/v1/posts/upload-url → Controllers::UploadUrl.call
  - [ ] パスパラメータの抽出（正規表現で :id を取得）
- [ ] 11.2 lambda/posts/controllers/index.rb を実装
  - [ ] ページネーション処理（offset ベース）
  - [ ] Post 一覧を取得（N+1クエリ対策）
  - [ ] レスポンスを返す（ステータスコード 200）
  - [ ] **テストを実行してパスすることを確認**
- [ ] 11.3 lambda/posts/controllers/show.rb を実装
  - [ ] id パラメータから Post を取得
  - [ ] is_liked フラグを含める（認証済みユーザーの場合）
  - [ ] レスポンスを返す（ステータスコード 200）
  - [ ] **テストを実行してパスすることを確認**
- [ ] 11.4 lambda/posts/controllers/create.rb を実装
  - [ ] Authorizer から user_id を取得
  - [ ] リクエストボディから投稿内容を取得
  - [ ] 画像キーがある場合、CloudFront URL を生成
  - [ ] Post レコードを作成
  - [ ] レスポンスを返す（ステータスコード 201）
  - [ ] **テストを実行してパスすることを確認**
- [ ] 11.5 lambda/posts/controllers/update.rb を実装
  - [ ] 権限チェック（自分の投稿のみ）
  - [ ] Post レコードを更新
  - [ ] レスポンスを返す（ステータスコード 200）
  - [ ] **テストを実行してパスすることを確認**
- [ ] 11.6 lambda/posts/controllers/destroy.rb を実装
  - [ ] 権限チェック（自分の投稿のみ）
  - [ ] Post レコードを削除（cascade で likes, comments も削除）
  - [ ] S3 から画像を削除
  - [ ] レスポンスを返す（ステータスコード 200）
  - [ ] **テストを実行してパスすることを確認**
- [ ] 11.7 lambda/posts/controllers/upload_url.rb を実装
  - [ ] Authorizer から user_id を取得
  - [ ] S3 Presigned URL を生成（PutObject 権限、10分間有効）
  - [ ] レスポンスを返す（presigned_url, image_key）
  - [ ] **テストを実行してパスすることを確認**
- [ ] 11.8 全テストを実行して全てパスすることを確認
- [ ] 11.9 lambroll deploy --function posts でデプロイ
- [ ] 11.10 Terraform で API Gateway の /api/v1/posts/{proxy+} リソースを作成
  - [ ] ANY メソッドで Lambda Proxy統合
  - [ ] Authorizer を適用（create, update, destroy, upload-url に必要）

## 12. いいね機能のテスト作成

- [ ] 12.1 いいね一覧取得のテスト作成（spec/likes/index_spec.rb）
  - [ ] 投稿のいいね一覧を取得成功のテスト
  - [ ] いいねがゼロの投稿のテスト
  - [ ] ユーザー情報が正しく含まれるテスト
- [ ] 12.2 いいね追加のテスト作成（spec/likes/create_spec.rb）
  - [ ] 投稿にいいね追加成功のテスト
  - [ ] 既にいいねした投稿に再度いいね失敗（422）のテスト
  - [ ] 存在しない投稿にいいね失敗（404）のテスト
  - [ ] いいね数（likes_count）が増加することのテスト
  - [ ] 未認証でいいね失敗のテスト
- [ ] 12.3 いいね削除のテスト作成（spec/likes/destroy_spec.rb）
  - [ ] 自分のいいね削除成功のテスト
  - [ ] いいねしていない投稿のいいね削除失敗（404）のテスト
  - [ ] いいね数（likes_count）が減少することのテスト
  - [ ] 未認証でいいね削除失敗のテスト

## 13. いいね Lambda 関数の実装（likes 機能グループ）

- [ ] 13.1 lambda/likes/handler.rb を実装（ルーティングロジック）
  - [ ] httpMethod と path によるルーティング
  - [ ] GET /api/v1/posts/:post_id/likes → Controllers::Index.call
  - [ ] POST /api/v1/posts/:post_id/likes → Controllers::Create.call
  - [ ] DELETE /api/v1/posts/:post_id/likes → Controllers::Destroy.call
  - [ ] パスパラメータの抽出（:post_id）
- [ ] 13.2 lambda/likes/controllers/index.rb を実装
  - [ ] post_id から Like 一覧を取得
  - [ ] ユーザー情報を含める
  - [ ] レスポンスを返す（ステータスコード 200）
  - [ ] **テストを実行してパスすることを確認**
- [ ] 13.3 lambda/likes/controllers/create.rb を実装
  - [ ] Authorizer から user_id を取得
  - [ ] 重複チェック
  - [ ] Like レコードを作成
  - [ ] Post の likes_count を +1
  - [ ] レスポンスを返す（ステータスコード 201）
  - [ ] **テストを実行してパスすることを確認**
- [ ] 13.4 lambda/likes/controllers/destroy.rb を実装
  - [ ] Authorizer から user_id を取得
  - [ ] Like レコードを削除
  - [ ] Post の likes_count を -1
  - [ ] レスポンスを返す（ステータスコード 200）
  - [ ] **テストを実行してパスすることを確認**
- [ ] 13.5 全テストを実行して全てパスすることを確認
- [ ] 13.6 lambroll deploy --function likes でデプロイ
- [ ] 13.7 Terraform で API Gateway の /api/v1/posts/{post_id}/likes リソースを作成
  - [ ] Lambda Proxy統合
  - [ ] Authorizer を適用（create, destroy に必要）

## 14. コメント機能のテスト作成

- [ ] 14.1 コメント一覧取得のテスト作成（spec/comments/index_spec.rb）
  - [ ] 投稿のコメント一覧を取得成功のテスト（古い順）
  - [ ] ページネーションのテスト
  - [ ] コメントがゼロの投稿のテスト
  - [ ] ユーザー情報が正しく含まれるテスト
- [ ] 14.2 コメント追加のテスト作成（spec/comments/create_spec.rb）
  - [ ] 投稿にコメント追加成功のテスト
  - [ ] 空のコメント内容で失敗（422）のテスト
  - [ ] 280文字超過で失敗（422）のテスト
  - [ ] 存在しない投稿にコメント失敗（404）のテスト
  - [ ] コメント数（comments_count）が増加することのテスト
  - [ ] 未認証でコメント追加失敗のテスト
- [ ] 14.3 コメント削除のテスト作成（spec/comments/destroy_spec.rb）
  - [ ] 自分のコメント削除成功のテスト
  - [ ] 他人のコメント削除失敗（403）のテスト
  - [ ] コメント数（comments_count）が減少することのテスト
  - [ ] 未認証でコメント削除失敗のテスト
  - [ ] 存在しないコメント削除失敗（404）のテスト

## 15. コメント Lambda 関数の実装（comments 機能グループ）

- [ ] 15.1 lambda/comments/handler.rb を実装（ルーティングロジック）
  - [ ] httpMethod と path によるルーティング
  - [ ] GET /api/v1/posts/:post_id/comments → Controllers::Index.call
  - [ ] POST /api/v1/posts/:post_id/comments → Controllers::Create.call
  - [ ] DELETE /api/v1/comments/:id → Controllers::Destroy.call
  - [ ] パスパラメータの抽出（:post_id, :id）
- [ ] 15.2 lambda/comments/controllers/index.rb を実装
  - [ ] post_id から Comment 一覧を取得（古い順）
  - [ ] ページネーション処理
  - [ ] ユーザー情報を含める
  - [ ] レスポンスを返す（ステータスコード 200）
  - [ ] **テストを実行してパスすることを確認**
- [ ] 15.3 lambda/comments/controllers/create.rb を実装
  - [ ] Authorizer から user_id を取得
  - [ ] バリデーション
  - [ ] Comment レコードを作成
  - [ ] Post の comments_count を +1
  - [ ] レスポンスを返す（ステータスコード 201）
  - [ ] **テストを実行してパスすることを確認**
- [ ] 15.4 lambda/comments/controllers/destroy.rb を実装
  - [ ] 権限チェック（自分のコメントのみ）
  - [ ] Comment レコードを削除
  - [ ] Post の comments_count を -1
  - [ ] レスポンスを返す（ステータスコード 200）
  - [ ] **テストを実行してパスすることを確認**
- [ ] 15.5 全テストを実行して全てパスすることを確認
- [ ] 15.6 lambroll deploy --function comments でデプロイ
- [ ] 15.7 Terraform で API Gateway のリソースを作成
  - [ ] /api/v1/posts/{post_id}/comments（Lambda Proxy統合）
  - [ ] /api/v1/comments/{id}（Lambda Proxy統合）
  - [ ] Authorizer を適用（create, destroy に必要）

## 16. エラーハンドリングとバリデーション

- [ ] 16.1 共通エラーハンドラーを Helpers Layer に追加
- [ ] 16.2 カスタムエラーレスポンスを定義（JSON 形式）
- [ ] 16.3 各 Lambda 関数にエラーハンドリングを追加
  - [ ] 404: Not Found
  - [ ] 401: Unauthorized
  - [ ] 403: Forbidden
  - [ ] 422: Unprocessable Entity
  - [ ] 500: Internal Server Error
- [ ] 16.4 バリデーションエラーメッセージのフォーマット統一

## 17. API Gateway の設定

- [ ] 17.1 Terraform で CORS を有効化（Gateway Response の設定）
- [ ] 17.2 API キーを設定（Terraform、オプション）
- [ ] 17.3 スロットリング設定を追加（Terraform、レート制限）
  - [ ] リクエストレート: 100 req/sec
  - [ ] バーストレート: 200 req
- [ ] 17.4 ステージを作成（Terraform、dev, staging, prod）
- [ ] 17.5 カスタムドメイン設定（Terraform、オプション、Route 53 + ACM 証明書）
- [ ] 17.6 デプロイメントを作成し、ステージに紐付け

## 18. CloudWatch Logs + X-Ray の設定

- [ ] 18.1 CloudWatch Logs を有効化（Lambda 作成時に自動的に作成される）
- [ ] 18.2 X-Ray トレーシングを有効化
  - [ ] Lambda の function.json に `"TracingConfig": {"Mode": "Active"}` を追加
  - [ ] Terraform で API Gateway のトレーシングを有効化
  - [ ] IAM ロールに X-Ray 書き込み権限を追加
- [ ] 18.3 Terraform で CloudWatch Alarms を作成
  - [ ] Lambda エラー率 > 5%
  - [ ] API Gateway 4xx エラー率 > 10%
  - [ ] API Gateway 5xx エラー率 > 1%
  - [ ] Lambda Duration > 5秒
- [ ] 18.4 Terraform で CloudWatch ダッシュボードを作成

## 19. 統合テストとE2Eテスト

- [ ] 19.1 Localstack で統合テスト（オプション）
  - [ ] Localstack で Lambda, API Gateway, S3 を起動
  - [ ] 統合テストを実行
- [ ] 19.2 E2E テスト（Postman/Newman）
  - [ ] Postman Collection を作成
  - [ ] 全エンドポイントのE2Eテストを作成
  - [ ] Newman で CI/CD パイプラインに統合

## 20. Seedデータの作成

- [ ] 20.1 seeds/ ディレクトリを作成
- [ ] 20.2 seed.rb を作成（サンプルユーザー、投稿、いいね、コメント）
- [ ] 20.3 seed.rb を実行してデータを投入

## 21. デプロイとドキュメント

- [ ] 21.1 Terraform で開発環境のインフラをデプロイ（`terraform apply`）
- [ ] 21.2 lambroll で全Lambda関数をデプロイ（`lambroll deploy --all`）
- [ ] 21.3 API Gateway のエンドポイント URL を取得（Terraform output）
- [ ] 21.4 README.md を更新
  - [ ] アーキテクチャ図を追加
  - [ ] API エンドポイント一覧を記載
  - [ ] 認証方法（JWT）の使い方を記載
  - [ ] Terraform によるインフラセットアップ手順を記載
  - [ ] lambroll によるLambdaデプロイ手順を記載
  - [ ] 環境変数の設定方法を記載
  - [ ] ローカル開発・テストのセットアップ手順を記載（TDDワークフロー）
- [ ] 21.5 OpenAPI (Swagger) 仕様を生成
  - [ ] API Gateway から OpenAPI 仕様をエクスポート（AWS CLI または Terraform）
  - [ ] Swagger UI で表示できるように設定

## 22. 最終確認とテスト

- [ ] 22.1 全ての Lambda 関数の動作確認
- [ ] 22.2 API Gateway のエンドポイント確認
- [ ] 22.3 全テストスイートの実行（RSpec）
- [ ] 22.4 E2E テストの実行（Postman/Newman）
- [ ] 22.5 CloudWatch Logs でログを確認
- [ ] 22.6 X-Ray でトレースを確認
- [ ] 22.7 CloudWatch Alarms が正しく設定されているか確認
- [ ] 22.8 S3 への画像アップロードの動作確認
- [ ] 22.9 CloudFront 経由での画像配信確認
- [ ] 22.10 Aurora Serverless v2 のスケーリング確認
- [ ] 22.11 Lambda のコールドスタート時間を計測
- [ ] 22.12 セキュリティチェック
  - [ ] IAM ポリシーの最小権限確認
  - [ ] VPC 設定の確認
  - [ ] S3 バケットのパブリックアクセスブロック確認
  - [ ] Secrets Manager の認証情報確認

## 23. 本番環境へのデプロイ

- [ ] 23.1 本番環境用の terraform workspace を作成
- [ ] 23.2 terraform apply --workspace=prod を実行（本番インフラのデプロイ）
- [ ] 23.3 本番環境用の環境変数を設定（env=prod）
- [ ] 23.4 lambroll deploy --all を実行（本番環境のLambda関数をデプロイ）
- [ ] 23.5 本番環境の動作確認
- [ ] 23.6 監視とアラートの設定を確認
- [ ] 23.7 バックアップの設定を確認（RDS 自動バックアップ）
