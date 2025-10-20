# プロジェクト要件（SSOT）

DOC-ID: RQ-OmniRe-SuGaR  
Version: 0.1.0  
Owner: planner (@motoki)  
Status: Draft  <!-- Draft | Active | Deprecated -->
Last-Updated: 2025-10-17
<!-- 本書は要件の Single Source of Truth（SSOT）です。変更は必ずチケット経由で反映し、他文書は本書を参照します。 -->

## 1. 目的と範囲（Purpose & Scope）
- 目的: OmniRe の動的物体表現に SuGaR の表面整列ガウシアンを取り込み、動的カテゴリのカノニカルメッシュ生成と高品質レンダリングを両立させる。
- スコープ: MultiTrainer 配下の `RigidNodes` / `DeformableNodes` / `SMPLNodes`、SuGaR 由来の正則化・レベルセット抽出・メッシュ拘束ガウシアン再初期化、対応する設定/ログ生成。
- 非スコープ: 静的背景ガウシアン、データローダの新規開発、SuGaR を静的カテゴリへ適用する拡張。
- ステークホルダー: orchestrator/reviewer, planner, coder, tester, judge

## 2. データと実行環境（Data & Environment）
- データセット:
  - `data/waymo/sample_scene/`（Checksum: TBD）― 要件検証用の短尺ローカルシーン
  - `data/nuscenes/sample/`（Checksum: TBD）― マルチカテゴリ動的物体のスモークテスト
- Python/依存: Python 3.9 / PyTorch 2.2, gsplat v1.3.0, Open3D >= 0.18（Poisson 再構成）, trimesh, PyMCubes（代替メッシュ確認用）, PyPDF2（仕様参照に利用済み）
- 決定性: seed=0（Trainer 設定）、canonical depth サンプリングは擬似乱数で再現性確保（`torch.manual_seed` で固定）
- 設定の正典: `configs/omnire_sugar.yaml`（新規）、既存 `configs/omnire.yaml` へ追加フラグ

## 3. 機能要件（FR）
### FR-ARCH-001: 動的カノニカル Gaussians の SuGaR 正則化
- 背景/根拠: 現行 OmniRe の動的カノニカル点群は表面アラインが弱く、メッシュ抽出や編集が困難。SuGaR の SDF ベース正則化（Eq.8,10）を適用して表面整列を促進する。
- 入力: OmniRe トレーニングシーン、各インスタンスのカノニカルガウシアン（`_means`）、フレーム毎のインスタンス姿勢/変形、カメラ姿勢・深度レンダラ。
- 処理:
  - (1) 既存学習 7k step → `opacity_entropy` 2k step → 低不透明度除去 → SuGaR 正則化 6k step のステージングを Trainer に導入。
  - (2) 各動的インスタンスについてカノニカル空間に正規化したビューを生成し、SuGaR 方式で深度マップから `ˆf(p)` をサンプル。
  - (3) `R_surface`（Eq.8）と `R_norm`（Eq.10）を動的ノードの LossDict に追加、ロギングキーを `loss/sugar_surface`, `loss/sugar_norm` とする。
- 出力: 強化された動的カノニカルガウシアン状態、トレーニングログ（Tensorboard/CSV）に新規正則化指標。
- 受入基準（Acceptance）:
  - スモークシーンで学習が 15k step 完走し NaN/Inf 無し。
  - `loss/sugar_surface` が 0.02 未満に収束（Waymo サンプルで参考値）。
  - `outputs/<run>/checkpoints/` に正則化後 ckpt が保存される。
- 成果物（Artifacts）:
  - `state/reports/RQ-OmniRe-SuGaR/tables/RQ-OmniRe-SuGaR_loss.csv`
  - `state/trace/RQ-OmniRe-SuGaR/<timestamp>-<short-hash>.json`
- リンク: Tickets[TBD] / Modules[models/nodes/deformable.py, models/nodes/rigid.py, models/renderers/depth.py] / Tests[tests/models/test_sugar_losses.py]

### FR-MESH-002: 動的インスタンス用カノニカルメッシュ抽出
- 背景/根拠: 表面整列後のガウシアンからカノニカル形状を抽出し、動的物体の編集・評価を可能にする。
- 入力: 正則化後 ckpt、各動的インスタンスのカノニカル Gaussians、SuGaR レベルセット閾値 λ、Open3D Poisson 実装。
- 処理:
  - (1) 深度マップからレベルセット候補点をサンプリングし、SuGaR 手法で線形補間により λ レベルの点群＋法線を生成。
  - (2) インスタンス毎に Poisson 再構成（深さ=10 をデフォルト、config で変更可）、Quadric Error Simplification でターゲット三角形数を制御。
  - (3) 生成メッシュを `outputs/<run>/meshes/<instance_id>_canonical.ply` に書き出し、メタ情報（頂点数/面数/λ）を JSON に記録。
- 出力: 各動的インスタンスのカノニカルメッシュ (`.ply`)、メッシュ統計 JSON。
- 受入基準:
  - 最低1カテゴリ（車両 or 歩行者）でメッシュ生成が完了し、`mesh_stats.json` に頂点数 > 5k が記録される。
  - メッシュ抽出ジョブの Wall time が 15 分以内（Waymo サンプル環境、1 GPU）で完走。
  - 同一 ckpt からの再実行で出力メッシュの頂点数差が ±1% 以内（決定性確認）。
- 成果物:
  - `state/reports/RQ-OmniRe-SuGaR/figures/RQ-OmniRe-SuGaR_mesh_preview.png`
  - `state/reports/RQ-OmniRe-SuGaR/tables/RQ-OmniRe-SuGaR_mesh_stats.csv`
- リンク: Tickets[TBD] / Modules[scripts/extract_mesh.py, utils/mesh/sugar_sampler.py] / Tests[tests/utils/test_sugar_sampler.py]

### FR-ARCH-003: メッシュ拘束ガウシアンによる動的カノニカル空間の再設計
- 背景/根拠: メッシュ抽出後も点ベースのカノニカル表現のままでは SuGaR の利点を活かしきれない。三角形上に張り付いた薄いガウシアンを保持し、時間方向の変形と整合させる必要がある。
- 入力: カノニカルメッシュ、各三角形の法線/面積、既存インスタンスの姿勢・変形パラメータ。
- 処理:
  - (1) 動的ノードに「SuGaR モード」を追加し、バリセン座標プリセット上に `n_gaussians_per_tri` 個の 2D ガウシアンを生成（学習可能スケール×2 + 面内回転 + SH/opacity）。
  - (2) `RigidNodes` はメッシュ頂点を剛体姿勢でワールドへ射影、`DeformableNodes`/`SMPLNodes` は既存変形ネットで頂点を更新した後にガウシアン位置を再計算。
  - (3) バリセン座標自体は固定とし、メッシュ頂点移動/法線によりガウシアンが追従する設計を実装。
  - (4) 学習ループにメッシュラプラシアン正則化とガウシアン面外厚み抑制項を追加。
- 出力: SuGaR メッシュ拘束ガウシアンを含む新 ckpt、`configs/omnire_sugar.yaml` による再現可能設定。
- 受入基準:
  - `rigid` / `deformable` ノードで SuGaR モードを有効化した学習がクラッシュ無しで 5k step 継続。
  - バリセン座標が `collect_gaussians_from_ids` API で参照できる。
  - メッシュ拘束後のレンダリングで VRAM 消費が baseline +20% 以内（Waymo サンプル、バッチ1）。
- 成果物:
  - `state/reports/RQ-OmniRe-SuGaR/figures/RQ-OmniRe-SuGaR_mesh_vs_gaussians.png`
  - `state/reports/RQ-OmniRe-SuGaR/tables/RQ-OmniRe-SuGaR_memory.csv`
- リンク: Tickets[TBD] / Modules[models/nodes/deformable.py, models/nodes/rigid.py, models/nodes/smpl.py, models/gaussians/surface_bound.py] / Tests[tests/models/test_surface_bound_nodes.py]

### FR-OPS-004: ステージオーケストレーションと CLI 対応
- 背景/根拠: SuGaR 導入により学習～メッシュ抽出～再最適化の3ステージが必要。CLI から一貫して呼び出せるよう統合フローを提供する。
- 入力: `tools/train.py`, `tools/extract_mesh.py`, `tools/refine_surface.py`（新規）へのコマンドライン引数、Hydra 設定。
- 処理:
  - (1) `tools/train.py` に `trainer.stage` 設定を追加（`pretrain`, `sugar_reg`, `surface_refine`）。
  - (2) `tools/run_sugar_pipeline.py` を新設し、学習→抽出→再学習を順次実行、各ステージ完了後にメタ情報を記録。
  - (3) パイプライン結果を `state/trace/T-FR-OPS-004/<timestamp>-<short-hash>.json` に自動追記。
- 出力: パイプライン実行スクリプト、Hydra 設定更新、ログファイル。
- 受入基準:
  - サンプルシーンで `python tools/run_sugar_pipeline.py --config configs/omnire_sugar.yaml` が完走。
  - `state/trace/T-FR-OPS-004/` にステージ別の git hash・コマンドが記録される。
  - CLI ヘルプに SuGaR 関連オプションが表示される。
- 成果物:
  - `state/trace/RQ-OmniRe-SuGaR/<timestamp>-<short-hash>.json`
- リンク: Tickets[TBD] / Modules[tools/run_sugar_pipeline.py, configs/omnire_sugar.yaml] / Tests[tests/tools/test_run_sugar_pipeline.py]

## 4. 非機能要件（NFR）
- 再現性:
  - Hydra コンフィグでステージ遷移・正則化重み・λ を明示し、`configs/` に格納。
  - 乱数シードを `utils/random.py` に一元化し、SuGaR 関連サンプルは全て seed 依存。
- ログ/トレース:
  - TensorBoard ログと CSV を `outputs/<run>/logs/` に保存。
  - パイプライン実行時は `state/trace/<ticket-id>/` に環境（GPU, CUDA, Open3D 版）を自動記述。
- コード品質:
  - 新規モジュールに型ヒント必須、`ruff` で lint、`black` 互換フォーマットを維持。
- パフォーマンス:
  - 正則化ステージの 1 iter あたり時間を baseline +25% 以内（Waymo サンプル）に抑える。

## 5. 指標と評価（Metrics & Evaluation）
- 指標: `loss/sugar_surface`, `loss/sugar_norm`, PSNR, SSIM, LPIPS, メッシュ Chamfer-L1, メッシュ頂点数。
- 評価手順:
  - Waymo サンプルで full pipeline を実行し、既存 OmniRe baseline と PSNR/SSIM を比較。
  - カノニカルメッシュと GT CAD（任意）で Chamfer-L1 を計測。
  - surface_refine 後に LPIPS を測定し 0.32 以下を目標。
- レポーティング:
  - 表: `state/reports/RQ-OmniRe-SuGaR/tables/RQ-OmniRe-SuGaR_metrics.csv`
  - 図: `state/reports/RQ-OmniRe-SuGaR/figures/RQ-OmniRe-SuGaR_psnr.png`

## 6. 検証計画（Verification & Validation）
- スモーク: `pytest -k "sugar"`, `python tools/run_sugar_pipeline.py --config configs/omnire_sugar.yaml --max_steps 200 --dryrun`
- 回帰: 既存 OmniRe baseline と同条件でレンダリングし、PSNR 差分 ≤ 0.5dB を確認。
- レビュー: judge が `state/reports` / `state/trace` / メッシュ出力を確認し承認。

## 7. 成果物と命名規約（Artifacts & Naming）
- Results: `state/reports/RQ-OmniRe-SuGaR*`
- Trace: `state/trace/RQ-OmniRe-SuGaR/`
- メッシュ: `outputs/<run>/meshes/<instance>_{canonical|refined}.ply`
- メッシュ統計: `outputs/<run>/meshes/mesh_stats.json`

## 8. トレーサビリティ（Traceability Matrix）
| RQ-ID | Ticket-ID | Module (src/) | Test (tests/) | Results (state/reports) | Trace (state/trace) |
|------|-----------|---------------|---------------|-------------------------|--------------------|
| FR-ARCH-001 | TBD | models/nodes/deformable.py | tests/models/test_sugar_losses.py | state/reports/RQ-OmniRe-SuGaR/tables/RQ-OmniRe-SuGaR_metrics.csv | state/trace/RQ-OmniRe-SuGaR/ |
| FR-MESH-002 | TBD | utils/mesh/sugar_sampler.py | tests/utils/test_sugar_sampler.py | state/reports/RQ-OmniRe-SuGaR/tables/RQ-OmniRe-SuGaR_mesh_stats.csv | state/trace/RQ-OmniRe-SuGaR/ |
| FR-ARCH-003 | TBD | models/nodes/rigid.py | tests/models/test_surface_bound_nodes.py | state/reports/RQ-OmniRe-SuGaR/figures/RQ-OmniRe-SuGaR_mesh_vs_gaussians.png | state/trace/RQ-OmniRe-SuGaR/ |
| FR-OPS-004 | TBD | tools/run_sugar_pipeline.py | tests/tools/test_run_sugar_pipeline.py | state/reports/RQ-OmniRe-SuGaR/tables/RQ-OmniRe-SuGaR_loss.csv | state/trace/RQ-OmniRe-SuGaR/ |

## 9. 変更管理（Change Control）
- 変更要求: GitHub Issue（`enhancement` ラベル）、judge にレビュー依頼、main へのマージ前に planner 承認。
- バージョン管理: 更新時は冒頭 Version/Last-Updated を更新し、関連 Issue/PR をリンク。
- 同期: `implementation-tasks.md` にタスク分解を追加し、完了後 SSOT と突合。

## 10. 未決事項（Open Issues）
- [ ] Poisson 再構成に使用するライブラリ（Open3D vs CGAL）最終決定（担当: planner / 期限: 2025-10-24）
- [ ] カノニカル深度レンダラの GPU 実装最適化方針（担当: coder / 期限: 2025-11-07）
