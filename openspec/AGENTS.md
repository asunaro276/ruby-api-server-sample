# OpenSpec ガイドライン

OpenSpecを使用した仕様駆動開発のためのAIコーディングアシスタント向けガイドラインです。

## TL;DR クイックチェックリスト

- 既存の作業を検索: `openspec spec list --long`、`openspec list`（全文検索には`rg`のみを使用）
- スコープを決定: 新機能 vs 既存機能の変更
- ユニークな`change-id`を選択: kebab-case、動詞主導（`add-`、`update-`、`remove-`、`refactor-`）
- スキャフォールド: `proposal.md`、`tasks.md`、`design.md`（必要な場合のみ）、および影響を受ける機能ごとのデルタ仕様
- デルタを記述: `## ADDED|MODIFIED|REMOVED|RENAMED Requirements`を使用し、各要件に少なくとも1つの`#### Scenario:`を含める
- 検証: `openspec validate [change-id] --strict`を実行し、問題を修正
- 承認を要求: 提案が承認されるまで実装を開始しない

## 3段階ワークフロー

### ステージ1: 変更の作成
以下の場合に提案を作成します:
- 機能や機能性を追加する
- 破壊的変更を行う（API、スキーマ）
- アーキテクチャやパターンを変更する
- パフォーマンスを最適化する（動作を変更）
- セキュリティパターンを更新する

トリガー（例）:
- 「変更提案を作成するのを手伝って」
- 「変更を計画するのを手伝って」
- 「提案を作成するのを手伝って」
- 「仕様提案を作成したい」
- 「仕様を作成したい」

緩い一致のガイダンス:
- 次のいずれかを含む: `proposal`、`change`、`spec`
- 次のいずれかと共に: `create`、`plan`、`make`、`start`、`help`

提案をスキップする場合:
- バグ修正（意図された動作を復元）
- タイポ、フォーマット、コメント
- 依存関係の更新（破壊的でない）
- 設定の変更
- 既存の動作のテスト

**ワークフロー**
1. `openspec/project.md`、`openspec list`、および`openspec list --specs`を確認して、現在のコンテキストを理解します。
2. ユニークな動詞主導の`change-id`を選択し、`openspec/changes/<id>/`配下に`proposal.md`、`tasks.md`、オプションの`design.md`、および仕様デルタをスキャフォールディングします。
3. `## ADDED|MODIFIED|REMOVED Requirements`を使用し、各要件に少なくとも1つの`#### Scenario:`を含めて仕様デルタをドラフトします。
4. `openspec validate <id> --strict`を実行し、提案を共有する前にすべての問題を解決します。

### ステージ2: 変更の実装
これらのステップをTODOとして追跡し、1つずつ完了させます。
1. **proposal.mdを読む** - 何が構築されているかを理解する
2. **design.mdを読む**（存在する場合） - 技術的決定を確認する
3. **tasks.mdを読む** - 実装チェックリストを取得する
4. **タスクを順番に実装** - 順番に完了する
5. **完了を確認** - ステータスを更新する前に、`tasks.md`のすべての項目が完了していることを確認する
6. **チェックリストを更新** - すべての作業が完了した後、すべてのタスクを`- [x]`に設定してリストが現実を反映するようにする
7. **承認ゲート** - 提案がレビューされ承認されるまで実装を開始しない

### ステージ3: 変更のアーカイブ
デプロイ後、別のPRを作成して:
- `changes/[name]/` → `changes/archive/YYYY-MM-DD-[name]/`に移動
- 機能が変更された場合は`specs/`を更新
- ツール専用の変更には`openspec archive <change-id> --skip-specs --yes`を使用（常に変更IDを明示的に渡す）
- `openspec validate --strict`を実行して、アーカイブされた変更がチェックに合格することを確認

## すべてのタスクの前に

**コンテキストチェックリスト:**
- [ ] `specs/[capability]/spec.md`で関連する仕様を読む
- [ ] `changes/`で保留中の変更を確認して競合を確認
- [ ] `openspec/project.md`で規約を読む
- [ ] `openspec list`を実行してアクティブな変更を確認
- [ ] `openspec list --specs`を実行して既存の機能を確認

**仕様を作成する前に:**
- 機能が既に存在するかどうかを常に確認
- 重複を作成するよりも既存の仕様を変更することを優先
- `openspec show [spec]`を使用して現在の状態を確認
- リクエストが曖昧な場合は、スキャフォールディング前に1〜2つの明確化質問をする

### 検索ガイダンス
- 仕様を列挙: `openspec spec list --long`（スクリプト用に`--json`）
- 変更を列挙: `openspec list`（または`openspec change list --json` - 非推奨だが利用可能）
- 詳細を表示:
  - 仕様: `openspec show <spec-id> --type spec`（フィルタには`--json`を使用）
  - 変更: `openspec show <change-id> --json --deltas-only`
- 全文検索（ripgrepを使用）: `rg -n "Requirement:|Scenario:" openspec/specs`

## クイックスタート

### CLIコマンド

```bash
# 必須コマンド
openspec list                  # アクティブな変更をリスト
openspec list --specs          # 仕様をリスト
openspec show [item]           # 変更または仕様を表示
openspec validate [item]       # 変更または仕様を検証
openspec archive <change-id> [--yes|-y]   # デプロイ後にアーカイブ（非対話的実行には--yesを追加）

# プロジェクト管理
openspec init [path]           # OpenSpecを初期化
openspec update [path]         # 指示ファイルを更新

# 対話モード
openspec show                  # 選択を促す
openspec validate              # 一括検証モード

# デバッグ
openspec show [change] --json --deltas-only
openspec validate [change] --strict
```

### コマンドフラグ

- `--json` - 機械可読出力
- `--type change|spec` - アイテムを明確化
- `--strict` - 包括的な検証
- `--no-interactive` - プロンプトを無効化
- `--skip-specs` - 仕様更新なしでアーカイブ
- `--yes`/`-y` - 確認プロンプトをスキップ（非対話的アーカイブ）

## ディレクトリ構造

```
openspec/
├── project.md              # プロジェクト規約
├── specs/                  # 現在の真実 - 何が構築されているか
│   └── [capability]/       # 単一の焦点を絞った機能
│       ├── spec.md         # 要件とシナリオ
│       └── design.md       # 技術パターン
├── changes/                # 提案 - 何を変更すべきか
│   ├── [change-name]/
│   │   ├── proposal.md     # なぜ、何を、影響
│   │   ├── tasks.md        # 実装チェックリスト
│   │   ├── design.md       # 技術的決定（オプション、基準を参照）
│   │   └── specs/          # デルタ変更
│   │       └── [capability]/
│   │           └── spec.md # ADDED/MODIFIED/REMOVED
│   └── archive/            # 完了した変更
```

## 変更提案の作成

### 決定木

```
新しいリクエスト?
├─ 仕様の動作を復元するバグ修正? → 直接修正
├─ タイポ/フォーマット/コメント? → 直接修正
├─ 新機能/機能性? → 提案を作成
├─ 破壊的変更? → 提案を作成
├─ アーキテクチャ変更? → 提案を作成
└─ 不明確? → 提案を作成（より安全）
```

### 提案の構造

1. **ディレクトリを作成:** `changes/[change-id]/`（kebab-case、動詞主導、ユニーク）

2. **proposal.mdを記述:**
```markdown
## Why（なぜ）
[問題/機会について1〜2文]

## What Changes（何が変わるか）
- [変更の箇条書きリスト]
- [破壊的変更には**BREAKING**をマーク]

## Impact（影響）
- 影響を受ける仕様: [機能をリスト]
- 影響を受けるコード: [主要なファイル/システム]
```

3. **仕様デルタを作成:** `specs/[capability]/spec.md`
```markdown
## ADDED Requirements
### Requirement: 新機能
システムは...を提供するものとします

#### Scenario: 成功ケース
- **WHEN** ユーザーがアクションを実行
- **THEN** 期待される結果

## MODIFIED Requirements
### Requirement: 既存機能
[完全な変更された要件]

## REMOVED Requirements
### Requirement: 古い機能
**Reason**: [削除理由]
**Migration**: [対処方法]
```
複数の機能が影響を受ける場合は、`changes/[change-id]/specs/<capability>/spec.md`配下に複数のデルタファイルを作成します—機能ごとに1つ。

4. **tasks.mdを作成:**
```markdown
## 1. Implementation（実装）
- [ ] 1.1 データベーススキーマを作成
- [ ] 1.2 APIエンドポイントを実装
- [ ] 1.3 フロントエンドコンポーネントを追加
- [ ] 1.4 テストを書く
```

5. **必要に応じてdesign.mdを作成:**
以下のいずれかが当てはまる場合は`design.md`を作成します。そうでなければ省略します:
- クロスカッティングな変更（複数のサービス/モジュール）または新しいアーキテクチャパターン
- 新しい外部依存関係または重要なデータモデルの変更
- セキュリティ、パフォーマンス、または移行の複雑さ
- コーディング前に技術的決定から恩恵を受ける曖昧さ

最小限の`design.md`スケルトン:
```markdown
## Context（コンテキスト）
[背景、制約、ステークホルダー]

## Goals / Non-Goals（目標 / 非目標）
- 目標: [...]
- 非目標: [...]

## Decisions（決定事項）
- 決定: [何をなぜ]
- 検討した代替案: [オプション + 根拠]

## Risks / Trade-offs（リスク / トレードオフ）
- [リスク] → 緩和策

## Migration Plan（移行計画）
[ステップ、ロールバック]

## Open Questions（未解決の質問）
- [...]
```

## 仕様ファイル形式

### 重要: シナリオのフォーマット

**正しい**（####ヘッダーを使用）:
```markdown
#### Scenario: ユーザーログイン成功
- **WHEN** 有効な認証情報が提供される
- **THEN** JWTトークンを返す
```

**間違い**（箇条書きや太字を使用しない）:
```markdown
- **Scenario: ユーザーログイン**  ❌
**Scenario**: ユーザーログイン     ❌
### Scenario: ユーザーログイン      ❌
```

すべての要件には少なくとも1つのシナリオが必要です。

### 要件の表現

- 規範的な要件にはSHALL/MUSTを使用（意図的に非規範的でない限り、should/mayは避ける）

### デルタ操作

- `## ADDED Requirements` - 新機能
- `## MODIFIED Requirements` - 変更された動作
- `## REMOVED Requirements` - 非推奨の機能
- `## RENAMED Requirements` - 名前の変更

ヘッダーは`trim(header)`で一致 - 空白は無視されます。

#### ADDEDとMODIFIEDの使い分け
- ADDED: 単独で要件として成立する新しい機能またはサブ機能を導入します。既存の要件のセマンティクスを変更するのではなく、直交的な変更（例: 「スラッシュコマンド設定」の追加）の場合はADDEDを優先します。
- MODIFIED: 既存の要件の動作、スコープ、または受け入れ基準を変更します。常に完全な更新された要件の内容（ヘッダー + すべてのシナリオ）を貼り付けます。アーカイバーはここで提供したもので要件全体を置き換えます。部分的なデルタは以前の詳細を失います。
- RENAMED: 名前のみが変更される場合に使用します。動作も変更する場合は、RENAMED（名前）とMODIFIED（内容）を新しい名前を参照して使用します。

一般的な落とし穴: 以前のテキストを含めずに、MODIFIEDを使用して新しい関心事を追加すること。これによりアーカイブ時に詳細が失われます。既存の要件を明示的に変更しない場合は、代わりにADDED配下に新しい要件を追加します。

MODIFIED要件を正しく作成する:
1) `openspec/specs/<capability>/spec.md`で既存の要件を見つけます。
2) 要件ブロック全体（`### Requirement: ...`からシナリオまで）をコピーします。
3) `## MODIFIED Requirements`配下に貼り付け、新しい動作を反映するように編集します。
4) ヘッダーテキストが正確に一致することを確認し（空白は区別しない）、少なくとも1つの`#### Scenario:`を保持します。

RENAMEDの例:
```markdown
## RENAMED Requirements
- FROM: `### Requirement: ログイン`
- TO: `### Requirement: ユーザー認証`
```

## トラブルシューティング

### 一般的なエラー

**"Change must have at least one delta"（変更には少なくとも1つのデルタが必要）**
- `changes/[name]/specs/`が.mdファイルと共に存在するか確認
- ファイルに操作プレフィックス（## ADDED Requirements）があるか確認

**"Requirement must have at least one scenario"（要件には少なくとも1つのシナリオが必要）**
- シナリオが`#### Scenario:`形式（4つのハッシュタグ）を使用しているか確認
- シナリオヘッダーに箇条書きや太字を使用しない

**シナリオ解析の静かな失敗**
- 正確な形式が必要: `#### Scenario: Name`
- デバッグ: `openspec show [change] --json --deltas-only`

### 検証のヒント

```bash
# 常に包括的なチェックのために厳密モードを使用
openspec validate [change] --strict

# デルタ解析のデバッグ
openspec show [change] --json | jq '.deltas'

# 特定の要件を確認
openspec show [spec] --json -r 1
```

## ハッピーパススクリプト

```bash
# 1) 現在の状態を探索
openspec spec list --long
openspec list
# オプションの全文検索:
# rg -n "Requirement:|Scenario:" openspec/specs
# rg -n "^#|Requirement:" openspec/changes

# 2) 変更IDを選択してスキャフォールド
CHANGE=add-two-factor-auth
mkdir -p openspec/changes/$CHANGE/{specs/auth}
printf "## Why\n...\n\n## What Changes\n- ...\n\n## Impact\n- ...\n" > openspec/changes/$CHANGE/proposal.md
printf "## 1. Implementation\n- [ ] 1.1 ...\n" > openspec/changes/$CHANGE/tasks.md

# 3) デルタを追加（例）
cat > openspec/changes/$CHANGE/specs/auth/spec.md << 'EOF'
## ADDED Requirements
### Requirement: 二要素認証
ユーザーはログイン時に第二要素を提供する必要があります。

#### Scenario: OTPが必要
- **WHEN** 有効な認証情報が提供される
- **THEN** OTPチャレンジが必要
EOF

# 4) 検証
openspec validate $CHANGE --strict
```

## 複数機能の例

```
openspec/changes/add-2fa-notify/
├── proposal.md
├── tasks.md
└── specs/
    ├── auth/
    │   └── spec.md   # ADDED: 二要素認証
    └── notifications/
        └── spec.md   # ADDED: OTPメール通知
```

auth/spec.md
```markdown
## ADDED Requirements
### Requirement: 二要素認証
...
```

notifications/spec.md
```markdown
## ADDED Requirements
### Requirement: OTPメール通知
...
```

## ベストプラクティス

### シンプルさ優先
- デフォルトで100行未満の新しいコード
- 不十分であることが証明されるまで単一ファイルの実装
- 明確な正当化なしにフレームワークを避ける
- 退屈で実証済みのパターンを選択

### 複雑さのトリガー
以下の場合にのみ複雑さを追加:
- 現在のソリューションが遅すぎることを示すパフォーマンスデータ
- 具体的なスケール要件（>1000ユーザー、>100MBデータ）
- 抽象化を必要とする複数の実証済みのユースケース

### 明確な参照
- コードの場所には`file.ts:42`形式を使用
- 仕様を`specs/auth/spec.md`として参照
- 関連する変更とPRをリンク

### 機能の命名
- 動詞-名詞を使用: `user-auth`、`payment-capture`
- 機能ごとに単一の目的
- 10分の理解可能性ルール
- 説明に「AND」が必要な場合は分割

### 変更IDの命名
- kebab-caseを使用、短く説明的に: `add-two-factor-auth`
- 動詞主導のプレフィックスを優先: `add-`、`update-`、`remove-`、`refactor-`
- 一意性を確保。すでに使用されている場合は、`-2`、`-3`などを追加

## ツール選択ガイド

| タスク | ツール | 理由 |
|------|------|-----|
| パターンでファイルを検索 | Glob | 高速なパターンマッチング |
| コードの内容を検索 | Grep | 最適化された正規表現検索 |
| 特定のファイルを読む | Read | 直接ファイルアクセス |
| 不明なスコープを探索 | Task | 複数ステップの調査 |

## エラー回復

### 変更の競合
1. `openspec list`を実行してアクティブな変更を確認
2. 重複する仕様を確認
3. 変更所有者と調整
4. 提案を組み合わせることを検討

### 検証の失敗
1. `--strict`フラグで実行
2. 詳細についてJSON出力を確認
3. 仕様ファイル形式を確認
4. シナリオが適切にフォーマットされているか確認

### コンテキストの欠落
1. まずproject.mdを読む
2. 関連する仕様を確認
3. 最近のアーカイブを確認
4. 明確化を求める

## クイックリファレンス

### ステージインジケーター
- `changes/` - 提案済み、まだ構築されていない
- `specs/` - 構築およびデプロイ済み
- `archive/` - 完了した変更

### ファイルの目的
- `proposal.md` - なぜと何を
- `tasks.md` - 実装ステップ
- `design.md` - 技術的決定
- `spec.md` - 要件と動作

### CLI必須コマンド
```bash
openspec list              # 進行中は何か?
openspec show [item]       # 詳細を表示
openspec validate --strict # 正しいか?
openspec archive <change-id> [--yes|-y]  # 完了をマーク（自動化には--yesを追加）
```

覚えておいてください: 仕様は真実です。変更は提案です。それらを同期させてください。
