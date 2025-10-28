# Social Media API 機能の追加（サーバーレスアーキテクチャ）

## Why（なぜ）

XのようなSNSアプリケーションのバックエンドAPI を、AWS Lambda + API Gateway を使用したサーバーレスアーキテクチャで構築する。ユーザーが投稿を作成・共有し、他のユーザーと交流できる基本的なソーシャルメディア機能を提供する必要がある。

サーバーレスアーキテクチャを選択することで、以下のメリットを実現：
- **自動スケーリング**: トラフィック増加に自動対応
- **コスト最適化**: 使用量に応じた従量課金
- **インフラ管理不要**: サーバーのメンテナンスやパッチ適用が不要
- **高可用性**: AWS のマネージドサービスによる信頼性

## What Changes（何が変わるか）

### アーキテクチャ
- **サーバーレスアーキテクチャ**: AWS Lambda + API Gateway を使用
- **データベース**: Amazon Aurora Serverless v2 (PostgreSQL) を使用
- **画像ストレージ**: S3 + CloudFront CDN
- **インフラ管理**: Terraform + Serverless Framework

### 機能
- JWT認証によるユーザー認証システムの追加（新規登録、ログイン、ログアウト、ユーザー情報取得）
- 投稿機能の追加（CRUD操作、画像添付サポート）
- いいね機能の追加（投稿へのいいね/いいね解除）
- コメント機能の追加（投稿へのコメント追加、表示、削除）
- 画像アップロード機能の追加（S3 Presigned URL による直接アップロード）

### インフラストラクチャ
- Lambda 関数（各エンドポイントごと）
- API Gateway（REST API）
- Aurora Serverless v2 + RDS Proxy
- VPC、サブネット、セキュリティグループ
- S3 バケット + CloudFront
- Lambda Layers（共通ライブラリ）
- CloudWatch Logs + X-Ray（監視・トレーシング）

## Impact（影響）

### 影響を受ける仕様
- `user-auth`: ユーザー認証・認可機能（新規追加）
- `posts`: 投稿管理機能（新規追加）
- `likes`: いいね機能（新規追加）
- `comments`: コメント機能（新規追加）

### 影響を受けるコード
- **Lambda 関数**: 各エンドポイントごとの Lambda 関数の作成
  - lambda/auth/ (register, login, logout, me)
  - lambda/posts/ (index, show, create, update, destroy)
  - lambda/likes/ (index, create, destroy)
  - lambda/comments/ (index, create, destroy)
  - lambda/authorizers/ (jwt_authorizer)
- **Lambda Layers**: 共通コード（モデル、ヘルパー、Gems）
- **データベーススキーマ**: User, Post, Like, Comment テーブルの追加（PostgreSQL）
- **インフラストラクチャ**:
  - terraform/ ディレクトリの追加、AWS リソースの定義
  - serverless.yml の作成（Serverless Framework）
- **設定ファイル**:
  - .gitignore（terraform関連、環境変数）
  - 環境変数管理（AWS Secrets Manager / Parameter Store）

### 新しいAWSリソース
- **Lambda 関数**: 約15個（エンドポイントごと）
- **API Gateway**: REST API（dev, staging, prod ステージ）
- **Aurora Serverless v2**: PostgreSQL クラスター
- **RDS Proxy**: データベース接続プール
- **VPC**: プライベートサブネット、セキュリティグループ、NAT Gateway
- **S3**: 画像ストレージバケット
- **CloudFront**: CDN ディストリビューション
- **Lambda Layers**: 3つ（Gems, Models, Helpers）
- **CloudWatch Logs**: ログストリーム
- **Secrets Manager**: 認証情報の保存
- **IAM**: ロール・ポリシー（Lambda 実行ロール）

### 技術的な決定事項
- **アーキテクチャ**: AWS Lambda + API Gateway（サーバーレス）
- **データベース**: Amazon Aurora Serverless v2 (PostgreSQL)
- **認証方式**: JWT (JSON Web Token) + Lambda Authorizer
- **API形式**: RESTful JSON API
- **画像ストレージ**: S3 + CloudFront（Presigned URL による直接アップロード）
- **APIバージョニング**: /api/v1/ 形式
- **インフラ管理**: Terraform + Serverless Framework による Infrastructure as Code
- **Lambda フレームワーク**: Serverless Framework + Ruby カスタムランタイム
- **接続管理**: RDS Proxy による接続プール最適化
- **監視**: CloudWatch Logs + X-Ray トレーシング

### コスト見積もり（概算）
**開発環境（低トラフィック）:**
- Lambda: 月額 $5-10（100万リクエスト想定）
- API Gateway: 月額 $3-5
- Aurora Serverless v2: 月額 $20-30（最小ACU）
- S3 + CloudFront: 月額 $5-10
- RDS Proxy: 月額 $15-20
- **合計: 月額 約 $50-75**

**本番環境（中程度のトラフィック）:**
- Lambda: 月額 $50-100（1000万リクエスト想定）
- API Gateway: 月額 $30-50
- Aurora Serverless v2: 月額 $100-200
- S3 + CloudFront: 月額 $50-100
- RDS Proxy: 月額 $20-30
- **合計: 月額 約 $250-480**

**スケーリング:**
従量課金のため、トラフィック増加に応じて自動スケーリング。

### セキュリティ考慮事項
- Lambda は VPC 内に配置（RDS へのアクセス）
- RDS は private subnet に配置、パブリックアクセス不可
- JWT トークン検証による認証
- IAM ポリシーで最小権限の原則を適用
- S3 バケットはプライベート設定
- API Gateway に スロットリング、WAF を設定（オプション）
- Secrets Manager で認証情報を管理
