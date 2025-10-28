---
name: OpenSpec: Apply
description: 承認されたOpenSpec変更を実装し、タスクを同期させます。
category: OpenSpec
tags: [openspec, apply]
---
<!-- OPENSPEC:START -->
**ガードレール**
- まずはシンプルで最小限の実装を優先し、要求された場合や明確に必要な場合にのみ複雑さを追加します。
- 変更は要求された結果に厳密にスコープを限定します。
- OpenSpecの規約や説明が必要な場合は、`openspec/AGENTS.md`（`openspec/`ディレクトリ内にあります。表示されない場合は`ls openspec`または`openspec update`を実行してください）を参照してください。

**ステップ**
これらのステップをTODOとして追跡し、1つずつ完了させます。
1. `changes/<id>/proposal.md`、`design.md`（存在する場合）、`tasks.md`を読んで、スコープと受け入れ基準を確認します。
2. タスクを順番に進め、編集は最小限にし、要求された変更に焦点を当てます。
3. ステータスを更新する前に完了を確認します。`tasks.md`のすべての項目が完了していることを確認してください。
4. すべての作業が完了した後にチェックリストを更新し、各タスクを`- [x]`とマークして現実を反映させます。
5. 追加のコンテキストが必要な場合は、`openspec list`または`openspec show <item>`を参照します。

**リファレンス**
- 実装中に提案から追加のコンテキストが必要な場合は、`openspec show <id> --json --deltas-only`を使用します。
<!-- OPENSPEC:END -->
