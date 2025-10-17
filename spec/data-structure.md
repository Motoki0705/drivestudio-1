# Data Structures — OmniRe × SuGaR Integration

主要テンソル構造とメタデータの整合をまとめる。Shape はデフォルトで `(N, …)` を表す。

## 1. Gaussian State (Baseline)
- `means` (`torch.Tensor`, `float32`, shape: `(N, 3)`): カノニカル空間の中心座標。
- `scales` (`torch.Tensor`, `float32`, shape: `(N, 3)`): 対数スケール。`VanillaGaussians.get_scaling = exp(scales)`。
- `quats` (`torch.Tensor`, `float32`, shape: `(N, 4)`): 正規化クォータニオン。
- `features_dc/rest` (`torch.Tensor`, shape: `(N, 1/((L^2-1)), 3)`): SH 係数。
- `opacities` (`torch.Tensor`, shape: `(N, 1)`): ロジット。
- `point_ids` (`torch.LongTensor`, shape: `(N, 1)`): ガウシアンが属するインスタンス ID。

## 2. Instance Metadata
- `instances_pose` (`torch.Tensor`, shape: `(T, K, 4, 4)`): フレーム T、インスタンス K の SE(3)。
- `instances_quats/trans` (`torch.nn.Parameter`, shape: `(T, K, 4)` / `(T, K, 3)`): 学習対象の姿勢補正。
- `instances_size` (`torch.Tensor`, shape: `(K, 3)`): AABB。
- `instances_fv` (`torch.BoolTensor`, shape: `(T, K)`): 各フレームでインスタンスが可視か。

## 3. SuGaR Regularization Artifacts
- `depth_maps` (`FloatTensor`, shape: `(V, H, W)`): カノニカル空間での Gaussian depth。`V` はビュー数。
- `samples_p` (`FloatTensor`, shape: `(M, 3)`): Eq.(9) に基づくサンプル点。
- `f_hat` (`FloatTensor`, shape: `(M,)`): サンプルごとの推定 SDF 値。
- `f_ideal` (`FloatTensor`, shape: `(M,)`): Eq.(7) に基づく理想 SDF。
- `loss/sugar_surface`, `loss/sugar_norm`: scalar。

## 4. Mesh Extraction Structures
- `lambda` (`float`): レベルセット閾値。要件標準値 0.3。
- `level_points` (`FloatTensor`, shape: `(Q, 3)`): レベルセット上の点。
- `level_normals` (`FloatTensor`, shape: `(Q, 3)`): ∇d に基づく正規化法線。
- `M_canon` (`open3d.geometry.TriangleMesh`): Poisson 再構成後のメッシュ。
- `mesh_stats.json`:
  ```json
  {
    "instance_id": {
      "lambda": 0.3,
      "vertices": 102345,
      "triangles": 204686,
      "poisson_depth": 10
    }
  }
  ```

## 5. Surface-bound Gaussian Representation
- `SurfaceBoundGaussians`:
  - `tri_indices` (`LongTensor`, shape: `(N,)`): メッシュ三角形インデックス。
  - `barycentric` (`FloatTensor`, shape: `(N, 3)`): 固定バリセン座標（∑=1）。
  - `scale_2d` (`FloatTensor`, shape: `(N, 2)`): 面内スケール（log 空間）。
  - `theta` (`FloatTensor`, shape: `(N, 1)`): 面内回転（ラジアン）。
  - `normal_offset` (`FloatTensor`, shape: `(N, 1)`): 面外方向補正（0 に近づくよう正則化）。
  - `features/opacities`: 既存ガウシアンと同様。
- 再投影:
  ```python
  # pseudo
  v0, v1, v2 = mesh.vertices[tri_indices]  # (..., 3)
  xyz = b0 * v0 + b1 * v1 + b2 * v2 + n * normal_offset
  ```

## 6. Pipeline Metadata
- `outputs/<run>/checkpoints/stage=<name>.ckpt`: 各ステージの保存ファイル。`trainer_state` に stage 情報を持たせる。
- `docs/trace/RQ-OmniRe-SuGaR.md`: 実行コマンド、Git hash、λ/Poisson depth などの run metadata。
- TensorBoard scalar tags:
  - `train/loss_sugar_surface`, `train/loss_sugar_norm`
  - `mesh/num_vertices`, `mesh/num_triangles`
  - `render/psnr_surface_refine`

## 7. Testing Fixtures
- `tests/fixtures/synthetic_instance.pt`: 小規模のカノニカルガウシアンとメッシュのモック（新規作成予定）。
- `tests/fixtures/depth_planes.pt`: 正則化の挙動確認用の平面サンプル。

## Notes
- 全データ構造は GPU/CPU 双方で動作できるよう `.to(device)` を統一。メッシュ抽出は CPU 実行を前提とし、必要に応じて `.cpu().numpy()` へ変換する。
- JSON/PLY 等の成果物はサイズが大きくなるため git-tracked には含めず、`outputs/` 配下に生成する。
