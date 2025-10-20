# Spec-Driven Template セットアップ手順

本書では `third_party/spec_driven_template/` に配置された仕様主導テンプレートを別プロジェクトへ展開する手順を記載する。

## 1. テンプレートの取得
1. `third_party/spec_driven_template/` ディレクトリを新規リポジトリ、または対象プロジェクトへコピーする。
2. コピー後、不要な履歴やバイナリがないか確認する（`resources/`, `state/resource/` など）。

## 2. 名前・ID の初期化
- `AGENTS.md`, `spec/overview/requirements.md`, `state/state.json` の `T-EXAMPLE-FEATURE` や `<Project-Code>` を実際の識別子へ置き換える。
- `state/tickets/T-EXAMPLE-FEATURE/` をチケット ID に合わせてリネームし、`index.md` と `C*-*.md` を編集する。

## 3. ドキュメントのカスタマイズ
| ファイル | 更新内容 |
|----------|----------|
| `spec/index.md` | 体制やリソースに合わせてディレクトリ説明を調整 |
| `spec/overview/*.md` | 具体的な要件・アーキテクチャ・データ構造を記入 |
| `spec/tasks/implementation-tasks.md` | 実際の FR とチェックポイントを整理 |
| `spec/workflow/branching-strategy.md` | 自組織の Git フロー・CI ポリシーに合わせて追記 |
| `AGENTS.md` | プロジェクト名・ロール固有の読み進め方を更新 |

## 4. state/ 初期セットアップ
1. `state/state.json` のステータスと担当を実プロジェクトの状況に合わせて更新。
2. `state/reports/` と `state/trace/` は空のまま開始し、作業が進んだら埋める。
3. リソースがあれば `state/resource/` に資料を追加し、参照先を `spec/` や `state/tickets/` からリンク。

## 5. テンプレート運用チェックリスト
- [ ] planner が SSOT (`spec/overview/requirements.md`) と `state/` の同期方法を把握している。
- [ ] coder / judge / researcher が `AGENTS.md` に沿ってオンボーディングできる。
- [ ] `implementation-tasks.md` と各 `C*-*.md` が関数・クラス契約レベルで具体化されている。
- [ ] ブランチ作成手順や既知リスクが `spec/workflow/branching-strategy.md` に整理されている。

## 6. 継続的改善
- テンプレートを利用した結果得られた学びは `docs/` に追加し、再利用性を高める。
- 汎用的に使えるアップデートはテンプレートへフィードバックし、他プロジェクトへ横展開する。
