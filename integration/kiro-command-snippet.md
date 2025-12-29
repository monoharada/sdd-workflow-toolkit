# Kiroコマンド連携スニペット

このスニペットは、`/kiro:spec-impl` コマンドに Codex レビューの自動呼び出しを追加するためのものです。

---

## 概要

`/kiro:spec-impl` コマンドで実装が完了した後、自動的に `/sdd-codex-review impl` を呼び出すようにします。

---

## 追加するセクション

Kiroコマンド定義ファイル（例: `.claude/commands/kiro/spec-impl.md`）の最後に以下を追加：

```markdown
## 実装完了後の自動レビュー

### 重要: Codexレビューの自動実行

タスク実装が完了したら、**必ず** Skill tool を使用して `sdd-codex-review` を呼び出すこと。

これは実装完了の**必須ステップ**です。スキップしないでください。

### 呼び出し方法

```
実装完了後、以下を実行：

1. Skill tool を使用して sdd-codex-review スキルを発動
2. 引数: impl [feature-name]
3. APPROVED が返されるまでループ
```

### 期待される動作

1. 実装コードの変更を検出
2. Codex CLI を使用して外部レビューを実行
3. レビュー結果を解析
4. NEEDS_REVISION の場合は修正を適用して再レビュー
5. APPROVED になったら次のタスクへ

### シミュレート禁止

- Codex CLI を実際に実行すること
- レビュー結果を自分で生成しないこと
- Session ID を報告に含めること
```

---

## 完全なコマンド定義例

以下は `/kiro:spec-impl` コマンドの完全な定義例です：

```markdown
---
name: kiro:spec-impl
description: Execute spec tasks using TDD methodology
arguments:
  - name: feature-name
    description: The feature to implement
  - name: task-numbers
    description: Optional specific task numbers (e.g., "1.1,1.2")
---

# Spec Implementation Command

## Overview

Implement tasks from `.kiro/specs/[feature]/tasks.md` using TDD methodology.

## Workflow

1. Read spec files (requirements.md, design.md, tasks.md)
2. Identify target tasks
3. For each task:
   a. Write failing test (RED)
   b. Implement minimal code (GREEN)
   c. Refactor (REFACTOR)
4. Update spec.json progress

## 実装完了後の自動レビュー

### 重要: Codexレビューの自動実行

タスク実装が完了したら、**必ず** Skill tool を使用して `sdd-codex-review` を呼び出すこと。

### 呼び出し

Skill tool invoke: "sdd-codex-review"
args: impl $1

### 条件

- APPROVED が返されるまで修正・再レビューをループ
- 最大6回のリトライ
- 6回超過時はユーザー介入を要求
```

---

## 注意事項

1. **Skill tool の使用**: `/sdd-codex-review` は Skill tool で呼び出す必要があります
2. **シミュレート禁止**: Codex CLI を実際に実行し、Session ID を報告に含めること
3. **ループ処理**: APPROVED になるまでレビュー→修正→再レビューのループを継続
4. **ユーザー介入**: 6回超過時は自動処理を停止し、ユーザーに判断を委ねる
