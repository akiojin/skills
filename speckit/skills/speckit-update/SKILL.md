---
name: speckit-update
description: GitHub Spec Kit (https://github.com/github/spec-kit) のベースバージョン更新やテンプレート/スクリプト同期を行うための手順。Spec Kitの更新、上流リリースとの差分適用、templates/commands/scriptsの取り込み、ローカル運用（日本語化・ブランチ非操作・SPEC-[UUID8桁]）の維持が必要なときに使用する。
---

# Spec Kit Update

GitHub Spec Kit: https://github.com/github/spec-kit

## Overview

GitHub公式 spec-kit の新リリースを既存リポジトリの `.specify/` に反映し、既存の運用ルール（日本語化、ブランチ非操作、SPEC ID形式）を保持したまま更新する。

## Workflow

### 1) 現状把握

- リポジトリ内の spec-kit ベースバージョンを特定する（例: `rg -n "v0\\.0\\.[0-9]+"`）。
- spec-kit 由来の対象を列挙する（例: `.specify/`, `.claude/commands/speckit.*.md`）。

### 2) 上流バージョン確認

- GitHubの spec-kit 最新タグ/リリースを確認する（Web検索またはタグ一覧）。
- 更新対象タグを決め、該当タグを `git clone` で取得する。

### 3) 差分取得

- 上流タグ間 diff か、上流→ローカル diff を取得する。
- 変更対象を重点確認する:
  - `templates/`
  - `templates/commands/`
  - `scripts/`（bash/powershell）
  - `memory/`
- 追加/削除ファイルも必ず把握する。

### 4) ローカル適用

- 上流の変更をローカルに反映する。
- **必須のローカル制約を再適用**:
  - すべて日本語
  - ブランチ操作の禁止（スクリプト/コマンドに checkout/switch/create が残らない）
  - SPEC ID は `SPEC-[UUID8桁]`
- 既存の運用差分がある場合は、短い理由を記録する。

### 5) ドキュメント更新

- バージョン表記を更新する（例: `CLAUDE.md`, `.specify/README.md`）。
- 更新履歴がある場合は追記する。

### 6) 検証と報告

- 主要スクリプトの動作を軽く確認する（`--help`, `--json` 等）。
- `rg` で英語の残存や旧バージョン表記がないか確認する。
- 変更点サマリ、影響範囲、未対応の差分を明確に報告する。

## Guardrails

- ブランチ作成/切替は行わない（ユーザー管理）。
- SPEC ID 形式は必ず `SPEC-[UUID8桁]` を維持する。
- 上流の英語テンプレートは日本語に翻訳してから反映する。
