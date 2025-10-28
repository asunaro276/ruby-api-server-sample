# ユーザー認証機能 仕様

## ADDED Requirements

### Requirement: ユーザー登録
システムは新規ユーザーの登録機能を提供するものとします（SHALL provide user registration）。

#### Scenario: 有効な情報でユーザー登録
- **WHEN** クライアントが `POST /api/v1/auth/register` に有効なユーザー情報（email, username, password）を送信
- **THEN** 新しいユーザーアカウントが作成され、JWTトークンとユーザー情報が返される
- **AND** ステータスコード 201 (Created) が返される

#### Scenario: 無効な情報でユーザー登録
- **WHEN** クライアントが無効なユーザー情報（空のemail、短すぎるパスワードなど）を送信
- **THEN** エラーメッセージとバリデーションエラーの詳細が返される
- **AND** ステータスコード 422 (Unprocessable Entity) が返される

#### Scenario: 重複するemailでユーザー登録
- **WHEN** クライアントが既に登録されているemailで登録を試みる
- **THEN** 重複エラーメッセージが返される
- **AND** ステータスコード 422 (Unprocessable Entity) が返される

### Requirement: ユーザーログイン
システムはユーザーのログイン機能を提供するものとします（SHALL provide user login）。

#### Scenario: 有効な認証情報でログイン
- **WHEN** クライアントが `POST /api/v1/auth/login` に有効な認証情報（email, password）を送信
- **THEN** JWTトークンとユーザー情報が返される
- **AND** ステータスコード 200 (OK) が返される
- **AND** トークンの有効期限は24時間とする

#### Scenario: 無効な認証情報でログイン
- **WHEN** クライアントが無効な認証情報（間違ったパスワードなど）を送信
- **THEN** 認証エラーメッセージが返される
- **AND** ステータスコード 401 (Unauthorized) が返される

### Requirement: 現在のユーザー情報取得
システムは認証済みユーザーの情報を取得する機能を提供するものとします（SHALL provide current user info retrieval）。

#### Scenario: 有効なトークンでユーザー情報取得
- **WHEN** クライアントが `GET /api/v1/auth/me` に有効なJWTトークンを含むリクエストを送信
- **THEN** 現在のユーザーの情報が返される
- **AND** ステータスコード 200 (OK) が返される

#### Scenario: 無効なトークンでユーザー情報取得
- **WHEN** クライアントが無効または期限切れのJWTトークンを含むリクエストを送信
- **THEN** 認証エラーメッセージが返される
- **AND** ステータスコード 401 (Unauthorized) が返される

#### Scenario: トークンなしでユーザー情報取得
- **WHEN** クライアントがJWTトークンなしでリクエストを送信
- **THEN** 認証エラーメッセージが返される
- **AND** ステータスコード 401 (Unauthorized) が返される

### Requirement: ユーザーログアウト
システムはユーザーのログアウト機能を提供するものとします（SHALL provide user logout）。

#### Scenario: 有効なトークンでログアウト
- **WHEN** クライアントが `DELETE /api/v1/auth/logout` に有効なJWTトークンを含むリクエストを送信
- **THEN** ログアウト成功メッセージが返される
- **AND** ステータスコード 200 (OK) が返される
- **AND** 該当トークンは無効化される（ブラックリスト機能は将来実装）

### Requirement: パスワードのバリデーション
システムはパスワードに対して適切なバリデーションを行うものとします（MUST validate passwords）。

#### Scenario: パスワードの最小文字数チェック
- **WHEN** ユーザーが8文字未満のパスワードで登録を試みる
- **THEN** バリデーションエラーが返される
- **AND** エラーメッセージに「パスワードは8文字以上である必要があります」と表示される

### Requirement: ユーザー名のバリデーション
システムはユーザー名に対して適切なバリデーションを行うものとします（MUST validate usernames）。

#### Scenario: ユーザー名の一意性チェック
- **WHEN** 既に存在するユーザー名で登録を試みる
- **THEN** バリデーションエラーが返される
- **AND** エラーメッセージに「このユーザー名は既に使用されています」と表示される

#### Scenario: ユーザー名の形式チェック
- **WHEN** ユーザーが3文字未満または20文字を超えるユーザー名で登録を試みる
- **THEN** バリデーションエラーが返される
- **AND** エラーメッセージに適切な文字数制限が表示される

### Requirement: メールアドレスのバリデーション
システムはメールアドレスに対して適切なバリデーションを行うものとします（MUST validate email addresses）。

#### Scenario: メールアドレスの形式チェック
- **WHEN** 無効な形式のメールアドレス（例: "invalid-email"）で登録を試みる
- **THEN** バリデーションエラーが返される
- **AND** エラーメッセージに「有効なメールアドレスを入力してください」と表示される
