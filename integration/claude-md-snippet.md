# CLAUDE.md 統合スニペット

このスニペットをプロジェクトの `CLAUDE.md` に追加してください。

---

## 追加するセクション

```markdown
### Codex Review Workflow (必須)

**重要**: 各kiro:specフェーズ完了後、必ずCodexレビューを実行すること。

```bash
# 各フェーズ完了後にCodexレビューを実行
/sdd-codex-review requirements [feature]  # kiro:spec-requirements後
/sdd-codex-review design [feature]        # kiro:spec-design後
/sdd-codex-review tasks [feature]         # kiro:spec-tasks後
/sdd-codex-review impl [feature]          # kiro:spec-impl後
```

ワークフロー:
1. `/kiro:spec-requirements [feature]` → `/sdd-codex-review requirements [feature]`
2. `/kiro:spec-design [feature]` → `/sdd-codex-review design [feature]`
3. `/kiro:spec-tasks [feature]` → `/sdd-codex-review tasks [feature]`
4. `/kiro:spec-impl [feature] [task]` → `/sdd-codex-review impl [feature]`

各フェーズでAPPROVEDを取得してから次へ進む。
```

---

## 配置例

`CLAUDE.md` の構造例：

```markdown
# CLAUDE.md

## Project Overview
...

## Development Commands
...

## Spec-Driven Development

### Codex Review Workflow (必須)

**重要**: 各kiro:specフェーズ完了後、必ずCodexレビューを実行すること。

```bash
# 各フェーズ完了後にCodexレビューを実行
/sdd-codex-review requirements [feature]  # kiro:spec-requirements後
/sdd-codex-review design [feature]        # kiro:spec-design後
/sdd-codex-review tasks [feature]         # kiro:spec-tasks後
/sdd-codex-review impl [feature]          # kiro:spec-impl後
```

ワークフロー:
1. `/kiro:spec-requirements [feature]` → `/sdd-codex-review requirements [feature]`
2. `/kiro:spec-design [feature]` → `/sdd-codex-review design [feature]`
3. `/kiro:spec-tasks [feature]` → `/sdd-codex-review tasks [feature]`
4. `/kiro:spec-impl [feature] [task]` → `/sdd-codex-review impl [feature]`

各フェーズでAPPROVEDを取得してから次へ進む。

## Architecture
...
```

---

## 注意事項

- このスニペットを追加することで、Claude Codeは各フェーズ完了後にCodexレビューを実行するよう認識します
- スキルが正しくインストールされていない場合、`/sdd-codex-review` コマンドは認識されません
- スキルのインストールは `~/.claude/skills/sdd-codex-review/` に配置することで完了します
