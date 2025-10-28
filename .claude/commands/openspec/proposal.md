---
name: OpenSpec: Proposal
description: 新しいOpenSpec変更をスキャフォールディングし、厳密に検証します。
category: OpenSpec
tags: [openspec, change]
---
<!-- OPENSPEC:START -->
**ガードレール**
- まずはシンプルで最小限の実装を優先し、要求された場合や明確に必要な場合にのみ複雑さを追加します。
- 変更は要求された結果に厳密にスコープを限定します。
- OpenSpecの規約や説明が必要な場合は、`openspec/AGENTS.md`（`openspec/`ディレクトリ内にあります。表示されない場合は`ls openspec`または`openspec update`を実行してください）を参照してください。
- 曖昧または不明確な詳細を特定し、ファイルを編集する前に必要なフォローアップ質問を行います。

**ステップ**
1. `openspec/project.md`を確認し、`openspec list`と`openspec list --specs`を実行し、関連するコードやドキュメント（例: `rg`/`ls`経由）を調査して、現在の動作に基づいて提案を行います。明確化が必要なギャップがあれば記録します。
2. ユニークな動詞主導の`change-id`を選択し、`openspec/changes/<id>/`配下に`proposal.md`、`tasks.md`、および必要に応じて`design.md`をスキャフォールディングします。
3. 変更を具体的な機能または要件にマッピングし、複数スコープの作業を明確な関係とシーケンスを持つ個別の仕様デルタに分割します。
4. ソリューションが複数のシステムにまたがる場合、新しいパターンを導入する場合、または仕様にコミットする前にトレードオフの議論が必要な場合は、`design.md`にアーキテクチャの論理を記録します。
5. `changes/<id>/specs/<capability>/spec.md`（機能ごとに1つのフォルダ）に仕様デルタをドラフトし、`## ADDED|MODIFIED|REMOVED Requirements`を使用して、各要件に少なくとも1つの`#### Scenario:`を含め、関連する機能を相互参照します。
6. `tasks.md`を、ユーザーに見える進捗を提供し、検証（テスト、ツール）を含み、依存関係や並列化可能な作業を強調する、小さく検証可能な作業項目の順序付きリストとしてドラフトします。
7. `openspec validate <id> --strict`で検証し、提案を共有する前にすべての問題を解決します。

**リファレンス**
- 検証が失敗した場合は、`openspec show <id> --json --deltas-only`または`openspec show <spec> --type spec`を使用して詳細を調査します。
- 新しい要件を書く前に、`rg -n "Requirement:|Scenario:" openspec/specs`で既存の要件を検索します。
- 提案が現在の実装の現実と整合するように、`rg <keyword>`、`ls`、または直接ファイルを読み取ってコードベースを探索します。
<!-- OPENSPEC:END -->
