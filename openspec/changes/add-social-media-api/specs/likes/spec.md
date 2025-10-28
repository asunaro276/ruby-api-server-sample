# いいね機能 仕様

## ADDED Requirements

### Requirement: いいねの追加
システムは認証済みユーザーが投稿にいいねを追加する機能を提供するものとします（SHALL provide like creation）。

#### Scenario: 投稿にいいねを追加
- **WHEN** 認証済みユーザーが `POST /api/v1/posts/:post_id/likes` にリクエストを送信
- **THEN** 投稿にいいねが追加される
- **AND** いいね情報が返される
- **AND** 投稿のいいね数が1増加する
- **AND** ステータスコード 201 (Created) が返される

#### Scenario: 既にいいねした投稿に再度いいね
- **WHEN** 認証済みユーザーが既にいいねした投稿に再度いいねを試みる
- **THEN** エラーメッセージが返される
- **AND** エラーメッセージに「既にいいねしています」と表示される
- **AND** ステータスコード 422 (Unprocessable Entity) が返される

#### Scenario: 存在しない投稿にいいね
- **WHEN** 認証済みユーザーが存在しない投稿にいいねを試みる
- **THEN** エラーメッセージが返される
- **AND** ステータスコード 404 (Not Found) が返される

#### Scenario: 認証なしでいいねを追加
- **WHEN** 未認証のユーザーがいいねを試みる
- **THEN** 認証エラーメッセージが返される
- **AND** ステータスコード 401 (Unauthorized) が返される

### Requirement: いいねの削除
システムは認証済みユーザーが投稿のいいねを削除する機能を提供するものとします（SHALL provide like deletion）。

#### Scenario: 自分のいいねを削除
- **WHEN** 認証済みユーザーが `DELETE /api/v1/posts/:post_id/likes` にリクエストを送信
- **THEN** いいねが削除される
- **AND** 削除成功メッセージが返される
- **AND** 投稿のいいね数が1減少する
- **AND** ステータスコード 200 (OK) が返される

#### Scenario: いいねしていない投稿のいいねを削除
- **WHEN** 認証済みユーザーがいいねしていない投稿のいいねを削除しようとする
- **THEN** エラーメッセージが返される
- **AND** エラーメッセージに「いいねしていません」と表示される
- **AND** ステータスコード 404 (Not Found) が返される

#### Scenario: 認証なしでいいねを削除
- **WHEN** 未認証のユーザーがいいねの削除を試みる
- **THEN** 認証エラーメッセージが返される
- **AND** ステータスコード 401 (Unauthorized) が返される

### Requirement: いいね一覧の取得
システムは投稿のいいね一覧を取得する機能を提供するものとします（SHALL provide likes listing）。

#### Scenario: 投稿のいいね一覧を取得
- **WHEN** クライアントが `GET /api/v1/posts/:post_id/likes` にリクエストを送信
- **THEN** 投稿のいいね一覧が返される
- **AND** 各いいねにはユーザー情報（username, avatar_url）が含まれる
- **AND** ステータスコード 200 (OK) が返される

#### Scenario: いいねがゼロの投稿のいいね一覧を取得
- **WHEN** クライアントがいいねがない投稿のいいね一覧を取得
- **THEN** 空の配列が返される
- **AND** ステータスコード 200 (OK) が返される

### Requirement: いいね状態の確認
システムは認証済みユーザーが特定の投稿にいいねしているかを確認する機能を提供するものとします（SHALL provide like status check）。

#### Scenario: 自分がいいねした投稿を確認
- **WHEN** 認証済みユーザーが投稿詳細を取得
- **THEN** レスポンスに現在のユーザーがいいねしているかどうかのフラグ（is_liked）が含まれる
- **AND** いいねしている場合は `is_liked: true` が返される

#### Scenario: 自分がいいねしていない投稿を確認
- **WHEN** 認証済みユーザーが投稿詳細を取得
- **THEN** レスポンスに `is_liked: false` が含まれる

### Requirement: いいね数のカウント
システムは投稿のいいね数を正確にカウントする機能を提供するものとします（MUST accurately count likes）。

#### Scenario: いいね追加時にカウント更新
- **WHEN** ユーザーが投稿にいいねを追加
- **THEN** 投稿の `likes_count` が1増加する
- **AND** カウンターキャッシュが更新される

#### Scenario: いいね削除時にカウント更新
- **WHEN** ユーザーがいいねを削除
- **THEN** 投稿の `likes_count` が1減少する
- **AND** カウンターキャッシュが更新される
