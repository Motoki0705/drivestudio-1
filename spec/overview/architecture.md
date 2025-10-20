# Architecture Overview — OmniRe with SuGaR Integration

## 1. Baseline OmniRe Pipeline
1. **Data ingestion:** `datasets/*` ローダが各フレームのマルチカメラ画像、LiDAR-derived 初期点群、インスタンス軌跡（bbox poses）を提供。
2. **MultiTrainer (`models.trainers.MultiTrainer`):**
   - バックグラウンドは `models.gaussians.VanillaGaussians`。
   - 動的オブジェクトは `models.nodes.RigidNodes`（剛体）と `models.nodes.DeformableNodes` / `SMPLNodes`（非剛体）。
   - 学習ループでは gaussian パラメータ（位置・スケール・クォータニオン・SH 係数）とインスタンス姿勢を最適化。
3. **Rendering:** `gsplat` カーネルで各ビューの色・深度レンダリングを行い、RGB/SSIM/Mask/Depth ロスを計算。
4. **Densification:** `RigidNodes` / `DeformableNodes` が分割・複製を通じてガウシアンを増殖し、詳細を復元。

## 2. SuGaR Integration Points

### 2.1 Regularization Stage (FR-ARCH-001)
- **Depth-map renderer:** 各動的インスタンスのカノニカル座標系でガウシアン深度マップを描画（SuGaR Fig.5 モデル）。
- **SDF approximation:** `R_surface`, `R_norm` を MultiTrainer の loss dict に追加。ステージング（7k pretrain → 2k opacity entropy → 6k SuGaR regularization）を `trainer.schedule` として hydra で制御。
- **State outputs:** カノニカル基底 `_means` が表面整列し、後続メッシュ抽出の入力として利用。

### 2.2 Canonical Mesh Extraction (FR-MESH-002)
- **Sampler:** `utils/mesh/sugar_sampler.py` が深度レイに沿ったレベルセットサンプル（λ=0.3 既定）と法線を生成。
- **Reconstruction:** Open3D Poisson を用いてインスタンス毎にメッシュ `M_canon` を構築し、Quadric Simplification でターゲット三角形数を維持。
- **Artifacts:** `outputs/<run>/meshes/<instance>_canonical.ply` と統計 JSON を作成。`state/reports/RQ-OmniRe-SuGaR/figures/` に可視化を保存。

### 2.3 Mesh-bound Gaussian Architecture (FR-ARCH-003)
- **Surface-bound Gaussians:** 新クラス `SurfaceBoundGaussians` が三角形毎に薄いガウシアン（2D スケール + 面内回転）を生成。バリセン座標を固定し、頂点更新で位置が決まる。
- **Node Integration:** `RigidNodes`/`DeformableNodes`/`SMPLNodes` に `mode="sugar"` を導入。メッシュ更新（剛体姿勢 or 変形ネット）後にガウシアンを再計算。
- **Regularization:** 面外スケール抑制、メッシュラプラシアン、時間方向の平滑化を LossDict へ追加。

### 2.4 Refinement Pipeline (FR-OPS-004)
- **Stage Orchestration:** `tools/run_sugar_pipeline.py`
  1. `train.py` を `stage=pretrain` → `stage=sugar_reg` → `stage=surface_refine` で順次起動。
  2. Stage 間でチェックポイントとメッシュを共有。
  3. 進捗を `state/trace/RQ-OmniRe-SuGaR/<timestamp>-<short-hash>.json` に追記。
- **Config:** `configs/omnire_sugar.yaml` が上記ステージとハイパーパラメータ（λ, Poisson depth, gaussians-per-tri 等）を管理。

## 3. Data & Flow
```text
Camera frames ─┐
                ├─> MultiTrainer ──> Canonical Gaussians ──┐
LiDAR priors ──┘                                          │
                                                        (SuGaR regularization)
                                                             │
Canonical depth maps ──> λ-level sampler ──> M_canon ──┐
                                                       │
                                 Surface-bound Gaussians (per triangle)
                                                       │
                                         Surface refinement training
                                                       │
                                Outputs: refined Gaussians + meshes + metrics
```

## 4. External Dependencies
- **gsplat:** Gaussian rasterization（既存）。
- **Open3D / PyMCubes:** Poisson 再構成・メッシュ操作。
- **PyTorch3D (既存):** KNN, geometry ops（法線計算等に再利用）。

## 5. Testing Strategy
- Unit testsで各段階の出力（SDF loss、サンプル点群、surface-bound forward）を検証。
- Mini pipeline テストで Waymo sample を通し、PSNR/SSIM と SuGaR loss の収束を確認。
