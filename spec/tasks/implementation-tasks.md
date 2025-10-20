# Implementation Tasks — OmniRe × SuGaR

## Legend
- `[x]` 完了 / merged
- `[~]` 進行中 (PR open or partial)
- `[ ]` 未着手

## P0: Planning & Alignment
- [x] RQ-OmniRe-SuGaR 要件ドラフト作成（`spec/overview/requirements.md`）
- [ ] ステークホルダー合意と Status=Active 化
- [ ] リスク・依存関係レビュー（Open3D 導入可否、GPU メモリ予算）

## P1: FR-ARCH-001 — SuGaR 正則化ステージ
- [ ] C1 カノニカル深度レンダラ拡張  
      実装: `models/renderers/canonical_depth.py` で CanonicalDepthRenderer を追加し、Hydra 設定 (`configs/omnire_sugar.yaml`) から各ビューの Gaussian depth map を生成。  
      テスト: `tests/renderers/test_canonical_depth_renderer.py` で near/far 範囲と multi-instance の分割を検証。
- [ ] C2 f̂(p) 推定ロジックと正則化損失  
      実装: `models/losses/sugar.py` に `estimate_f_hat`, `sugar_surface_loss`, `sugar_normal_loss` を実装し、`loss/sugar_surface`・`loss/sugar_norm` をロガーへ登録。  
      テスト: `tests/models/test_sugar_losses.py` で数値一致と勾配安定性 (`autograd.gradcheck`) を確認。
- [ ] C3 MultiTrainer ステージ遷移 & opacity entropy スケジュール反映  
      実装: `models/trainers/base.py` に StageSchedule を導入し、7k→2k→6k の切替と entropy 減衰を Hydra で制御。`models/trainers/single.py` / `scene_graph.py` からステージを参照。  
      テスト: `tests/trainers/test_sugar_stage_schedule.py` でステージ進行と再開時の復元を検証。
- [ ] C4 ログ出力 (`loss/sugar_surface`, `loss/sugar_norm`) と学習再開対応  
      実装: Trainer の save/load でステージ情報を保持し、TensorBoard/CSV ロガーに SuGaR 指標を追加。  
      テスト: `tests/trainers/test_sugar_resume.py` で checkpoint 復元後もロスが連続するか確認。
- [ ] C5 単体テスト整備  
      実装: `tests/conftest.py` にダミーシーン fixture を追加し、SuGaR 関連テストを pytest で 2 分以内に通過させる。  
      テスト: `pytest -k sugar --maxfail=1` を CI タスク化し、全テストが pass。

詳細要件と成果物は `state/tickets/T-FR-ARCH-001/` 配下の `C1-*.md` ～ `C5-*.md` を参照。

## P1: FR-MESH-002 — カノニカルメッシュ抽出
- [ ] `utils/mesh/sugar_sampler.py`（レベルセット点群＆法線サンプラ）
- [ ] Poisson 再構成 CLI（`scripts/extract_mesh.py` or `tools/extract_mesh.py` 拡張）
- [ ] メッシュ簡略化 + メタ情報書き出し
- [ ] スモークデータ（Waymo sample）での再現性確認
- [ ] 単体テスト `tests/utils/test_sugar_sampler.py`

## P1: FR-ARCH-003 — メッシュ拘束ガウシアン
- [ ] Surface-bound Gaussian クラス（2D scale + 面内回転）
- [ ] `RigidNodes`/`DeformableNodes`/`SMPLNodes` への SuGaR モード追加
- [ ] メッシュ更新とバリセン座標再投影の互換確認
- [ ] 面外スケール抑制・ラプラシアン正則化導入
- [ ] 単体テスト `tests/models/test_surface_bound_nodes.py`

## P1: FR-OPS-004 — パイプライン & CLI
- [ ] `configs/omnire_sugar.yaml` 作成（Hydra タスク＆ステージ設定）
- [ ] `tools/run_sugar_pipeline.py` 実装（train → mesh → refine）
- [ ] Hydra/CLI ドキュメント更新（README / docs）
- [ ] テスト `tests/tools/test_run_sugar_pipeline.py`

## P2: 評価 & 可視化
- [ ] メトリクス集計スクリプト更新（PSNR/SSIM/LPIPS + SuGaR特有）
- [ ] メッシュ可視化（`state/reports/RQ-OmniRe-SuGaR/figures` 用スナップショット生成）
- [ ] `state/trace/RQ-OmniRe-SuGaR/` テンプレ実装

## P2: インフラ・依存
- [ ] Open3D / PyMCubes / 追加ライブラリの依存管理（`requirements.txt`, `pyproject.toml`）
- [ ] CI/Lint ルール更新（ruff, pytest ターゲット）
- [ ] GPU メモリ & 時間計測スクリプト整備

## Done When
- [ ] 4 つの FR が unit test & integration test を通過
- [ ] パイプライン実行ログと成果物が要件表の保存先に配置
- [ ] judge レビュー完了、ドキュメント同期済み
