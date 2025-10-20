# Directory Structure — DriveStudio / OmniRe × SuGaR

Repository ルート配下の主なディレクトリと、SuGaR 取り込みに関連する役割を整理する。

```
.
├── configs/                  # Hydra 設定束。SuGaR 用に `omnire_sugar.yaml` を追加予定。
│   ├── datasets/             # 各データセット専用設定（Waymo、NuScenes 等）
│   └── *.yaml                # 既存トレーナ設定（omnire, streetgs, deformablegs, pvg）
├── data/                     # 小規模サンプルリスト・補助データ。要件テスト用 sample シーンを格納。
├── datasets/                 # データローダ & プリプロセスコード。
├── docs/                     # データ準備・研究資料。SuGaR 結果の抜粋を掲載する場合は state/reports から引用。
├── models/                   # Gaussian/Nodal モデル実装。
│   ├── gaussians/            # Vanilla/Deformable Gaussians。本タスクで surface-bound 版を追加。
│   ├── nodes/                # Rigid/Deformable/SMPL Nodes。SuGaR モード拡張対象。
│   ├── trainers/             # MultiTrainer 等。正則化ステージングをここで制御。
│   └── modules.py            # MLP/DeformNetwork 等の共通モジュール。
├── scripts/                  # データセットセットアップ等。
├── spec/                     # SSOT・実装計画ドキュメント（本ファイル含む）。
├── state/                    # 進捗管理・リソース（tickets, reports, trace, state.json）。
├── third_party/              # 外部サブモジュール（SMPLX 等）。
├── tools/                    # CLI エントリ (`train.py`, `eval.py`)。SuGaR パイプラインスクリプトを追加。
├── utils/                    # 汎用ユーティリティ。SuGaR 用メッシュサンプラ/レンダラを `utils/mesh/` として拡張予定。
└── uv.lock / pyproject.toml  # 依存管理（uv + pip）。
```

## Planned Additions
- `configs/omnire_sugar.yaml` — SuGaR ステージ設定を明示。
- `tools/run_sugar_pipeline.py` — 学習→メッシュ抽出→表面拘束を一気通貫で実行。
- `utils/mesh/`（新ディレクトリ） — レベルセットサンプリングや Poisson 入力生成ユーティリティ。
- `models/gaussians/surface_bound.py` — メッシュ拘束ガウシアン表現。
- `state/reports/RQ-OmniRe-SuGaR/` — 要件成果物（表・図・サマリ）をまとめるサブディレクトリ。
- `state/trace/RQ-OmniRe-SuGaR/` — パイプライン実行ログ（JSON）を集約するサブディレクトリ。

## Notes
- 静的背景ガウシアンや既存ベースラインのための構造はそのまま維持し、SuGaR 導入に伴う追加分は既存階層と整合するよう配置する。
- 大規模アセット（学習済み ckpt 等）は `outputs/`（git ignore 前提）配下に生成し、リポジトリには含めない。
