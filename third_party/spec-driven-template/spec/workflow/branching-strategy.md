# Branching Strategy（Planner 指揮下）

## 目的
- マルチエージェント開発における GitHub リポジトリ運用を標準化し、`state/` と同期したトレーサビリティを確保する。
- planner がブランチ・PR のライフサイクルを統制し、受入基準とリリース品質を維持する。

## ブランチ階層
- `main`: 常にデプロイ可能な安定ブランチ。judge 承認済みの成果のみマージ。
- `release/<version>`: 必要に応じ作成。リリース準備・タグ付け・緊急修正を管理（任意運用）。
- `feature/<ticket-id>-<slug>`: coder が作業するブランチ。チケットと 1:1 対応。

## ワークフロータイムライン（Who / When / What）

1. **Kickoff（Ticket → Ready）**
   - **When**: 依存が解消し、planner がチケットを `Ready` に移したタイミング。
   - **Who**: planner。
   - **What**: `feature/<ticket-id>-<slug>` の命名をチケットに記載し、必要ならローカルで作成して push。担当 coder を決定し、GitHub 上で作業方針を共有。

2. **Development（Ready → In Progress）**
   - **When**: planner が coder をアサインし、`In Progress` に遷移した直後。
   - **Who**: coder（planner は進捗モニタ）。
   - **What**: feature ブランチをチェックアウトして実装・テスト。主要コマンドや成果物を `state/trace/` に記録しながら、進展ごとに push。

3. **Review Preparation（Finishing Development）**
   - **When**: coder が受入基準を満たしたと判断した時。
   - **Who**: coder → planner。
   - **What**: ローカル/CI テストを完了し、`state/reports/<ticket-id>.md` を更新。planner へレビュー準備完了を通知。必要に応じ rebase / 最新化を実施。

4. **PR Creation（In Progress → In Review）**
   - **When**: planner が内容確認後、PR 作成を許可した時。
   - **Who**: coder が PR を作成し、planner がメタ情報を整備。
   - **What**: coder はテンプレートに沿って PR を作成し、`tickets`・`reports`・テスト証跡をリンク。planner は説明を確認し、judge をレビュアに割当。`state/state.json` を `In Review` に更新。

5. **Review & Approval**
   - **When**: PR がオープンした直後からマージまで。
   - **Who**: judge（レビュー）、coder（対応）、planner（進行管理）。
   - **What**: judge は 1 営業日以内にレビュー開始し、GitHub でコメント・Approve/Request Changes を実行。coder は指摘に対応して再度テスト／`trace`／PR を更新。planner は状態と期限を監視し、停滞時は介入。

6. **Merge & Post Merge（In Review → Done）**
   - **When**: judge が Approve し、CI がグリーンになった後。
   - **Who**: planner。
   - **What**: `main`（または `release/*`）へマージし、タグ付け / リリースノートが必要なら作成。feature ブランチを削除し、`state/state.json` を `Done` に更新。`reports` にマージ完了を追記。

7. **Hotfix / Rollback**
   - **When**: 重大障害・緊急修正が必要になった場合。
   - **Who**: planner 主導、必要に応じ coder / judge。
   - **What**: `hotfix/<issue>` ブランチを planner が作成し、迅速に修正。完了後 `main` と該当 `release/*` にマージ、再発防止策を `reports`／`tickets` に記録。

## GitHub アクション項目（planner 担当）
- ブランチ作成依頼／命名共有
- PR 作成・説明整備の指示
- レビュア/approver のアサインと期限設定
- マージ前の確認（CI 成否、テスト結果、reports の整合）
- タグ付け・リリースメモ作成（必要に応じて）

## MCP 経由 GitHub 操作（planner ガイド）
- **前提設定**: `~/.codex/config.toml` の `mcp_servers.github.env` に `GITHUB_PERSONAL_ACCESS_TOKEN` を設定し、`npx -y @modelcontextprotocol/server-github` を起動したターミナルを開いたままにする。
- **共通呼び出し**: Codex CLI から `/tool <name> { ... }` 形式で GitHub MCP ツールを実行する。主要ツールと用途は次の通り。
  - ブランチ作成: `github__create_branch {"owner":"Motoki0705","repo":"drivestudio-1","branch":"feature/...","from_branch":"main"}`。
  - ファイル編集コミット: `github__create_or_update_file {"owner":"...","repo":"...","branch":"feature/...","path":"src/...","message":"...","content":"<base64>"}`。
  - Issue 発行: `github__create_issue {"owner":"...","repo":"...","title":"...","body":"..."}`（チケット同期用）。
  - PR 作成: `github__create_pull_request {"owner":"...","repo":"...","head":"feature/...","base":"main","title":"...","body":"..."}`。
  - 監視系: `github__list_issues`, `github__list_pull_requests`, `github__get_pull_request_status` などで進捗を照会し、`state/` と突き合わせる。
- **推奨ルーチン**
  1. チケットを `Ready` に移す際に `github__create_branch` でブランチを先行作成し、命名を `state/tickets` に追記。
  2. coder から成果物報告を受けたら、内容を確認しつつ `github__create_or_update_file` で必要な付随ドキュメントをコミットさせる。
  3. PR 解放時は `github__create_pull_request` を用い、テンプレ記載事項（チケット番号・テスト結果）を `body` に含める。
  4. レビュー中は `github__get_pull_request_status` / `github__get_pull_request_files` で差分と CI を確認し、Judge へのフォローを実施。
- **トラブルシューティング**
  - 認証エラー時は MCP サーバーログを確認し、PAT のスコープ（`repo`）や環境変数名のスペルミスを点検。
  - ネットワーク制限に注意し、必要なら `curl https://api.github.com` で疎通確認を取ってから再試行する。
- **Git push / PR 作成手順（再現用）**
  1. ネットワークを確認：`curl -I https://github.com` と `curl -I https://api.github.com` が 200 を返すことを確認。
  2. SSH 資格情報を検証：`ssh -T git@github.com` でホストに到達できるかチェック。到達できない場合は DNS 設定や PAT の有無を点検。
  3. リモート URL を統一：`git remote set-url origin git@github.com:Motoki0705/drivestudio-1.git` で SSH 経由に切替え、`git remote -v` で確認。
  4. ブランチを push：`git push -u origin <branch-name>`。初回は必要に応じて `with_escalated_permissions` 付きで実行し、`trace` にコマンドと結果を記録。
  5. `gh` CLI のデフォルト設定：`gh repo set-default` でリポジトリを選択し、`gh pr create --base main --head <branch-name> --title "..." --body "..."` を実行。
  6. 併せて MCP ツール（`github__create_pull_request`）で PR を生成した場合はリンクと番号を `state/trace` に残す。

## 既知インシデントと再発防止策
- 2025-10-20 に発生した「feature/T-FR-ARCH-001 ブランチの親コミット誤りによる大規模差分発生について」（[Issue #9](https://github.com/Motoki0705/drivestudio-1/issues/9)）では、`spec/mcp-workflow-docs-followup` 上で新規 feature ブランチを切ったために 180 ファイル超の不要差分が混入した。  
- 再発防止として以下を必須フローに追加する:  
  1. ブランチ作成前に `git status` がクリーンで、`git branch --show-current` が `main` を指すことを確認する。  
  2. `git merge-base --is-ancestor origin/main HEAD` が成功することをチェックし、失敗した場合は `git pull --rebase origin main`。  
  3. ブランチ作成直後に `git diff --stat origin/main...HEAD` を実行し、意図した差分範囲のみになっているかセルフレビューする。  
  4. 想定外の差分が混入した場合は Issue #9 の対処手順（ブランチ削除→main へ戻る→再作成→差分最小化）を踏襲し、`state/trace/` へ記録する。  
- planner は Ready 遷移時にこのチェックリストを coder に共有し、`state/tickets/T-FR-ARCH-001/C*-*.md` に記載した成果物点検と併せて遵守させる。

## PR テンプレート（推奨項目）
- Linked Ticket(s): `T-0123`
- Summary: 実装内容／テスト結果の要約
- Test Evidence: コマンド／スクリーンショット／ログ
- Checklist: 受入基準、ドキュメント更新の有無など

## CI / テスト方針
- feature ブランチ PR 時に主要テストを自動実行。失敗時は coder が修正し再度提出。
- main ブランチでは smoke テスト / セキュリティチェックを走らせ、安定性を監視。
- planner は CI 状況を `reports` と突き合わせ、テスト未実施や結果不一致があれば差戻し。

## 権限ポリシー
- マージ権限は planner（必要に応じ副管理者）に限定。
- force push / rebase は planner が承認した場合のみ。通常は fast-forward / merge commit を使用。
- ブランチ削除はマージ後に planner が実施し、`trace` へ記録する。
