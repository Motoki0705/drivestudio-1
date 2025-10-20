# Spec Overview & Index

## 全体サマリ
- **開発目標**：`overview/requirements.md` を単一の真実の源泉（SSOT）とし、マルチエージェント体制で要求→計画→実装→審査までを循環させる。
- **体制**：planner が orchestrator を兼務し、coder / judge / researcher が `workflow/agents.md` の契約に従って連携。
- **オペレーション**：進捗は `state/`、GitHub 運用は `workflow/branching-strategy.md`、役割別手順は `roles/` に集約。

## ディレクトリ別概要とリンク

### overview/ — プロダクト全体像
- [requirements.md](overview/requirements.md)：ビジネス要件・受入基準のSSOT。変更は全てチケット経由で反映。
- [architecture.md](overview/architecture.md)：システム構成図と主要コンポーネントの役割整理。
- [data-structure.md](overview/data-structure.md)：ドメインデータとAPIスキーマ定義。
- [directory-structure.md](overview/directory-structure.md)：リポジトリ構造と各フォルダの用途。

### workflow/ — 流れとルール
- [agents.md](workflow/agents.md)：ティア構造、ロール責務、状態遷移の可視化。
- [state-formats.md](workflow/state-formats.md)：`state/` 配下のファイル仕様（tickets, state.json, trace, reports）。
- [branching-strategy.md](workflow/branching-strategy.md)：GitHub ブランチ命名、PR 手順、CI/マージ規範。

### roles/ — エージェント別プレイブック
- [agent-planner.md](roles/agent-planner.md)：planner の日次ルーチン、判断基準、エスカレーション。
- [agent-coder.md](roles/agent-coder.md)：実装〜テスト〜レポート更新のワークフロー。
- [agent-judge.md](roles/agent-judge.md)：審査プロセス、チェックリスト、差戻し方針。
- [agent-researcher.md](roles/agent-researcher.md)：調査依頼の受け方、情報の信頼性評価、報告テンプレ。

### tasks/ — 実務タスク
- [implementation-tasks.md](tasks/implementation-tasks.md)：進行中／予定の実装タスク、優先順位、担当。

## 参照ガイド
- **新規メンバー**：まず `overview/requirements.md` と `workflow/agents.md` を読み、体制とゴールを把握。
- **planner**：日次で `roles/agent-planner.md` を参照し、`workflow/state-formats.md` と `workflow/branching-strategy.md` をセット運用。
- **coder / judge**：担当するチケットに入る前に各プレイブックと `state-formats.md` を確認。
- **researcher**：調査依頼時に `roles/agent-researcher.md` を参照し、成果を `reports` や `requirements.md` に反映する際は planner と連携。
