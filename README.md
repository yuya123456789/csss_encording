# CSSS Encoding Project

このリポジトリは、`code/` フォルダ内のJupyter Notebookを中心に、データ前処理、埋め込み生成、回帰モデル、予測および解析を行うためのプロジェクトです。

## 主要ディレクトリ

- `code/`
  - データ処理および解析を行うJupyter Notebook群
- `emb/`
  - 埋め込みデータを保存するpickleファイルなど
- `requirements.txt`
  - Python実行環境の依存パッケージ
- `analysis_corr2/`
  - 分析結果出力用の空ディレクトリ（`absolute_mean`, `isc_base_mean`, `normal_mean`, `figures`）
- `analysis_corr/`, `atlas_label/`, `list/`, `output_csv/`, `parcels_csv/`, `predicted_all_csv/`, `transcript/`
  - 空ディレクトリとして `.gitkeep` が配置されています

## Notebook 構成

### 1. `code/1_story_csv_and_normalized.ipynb`
**パーセルタイムシリーズの抽出と正規化**

fMRIの生データ（NIfTI形式）からSchaefer atlas（400パーセル×17ネットワーク）を用いてパーセル単位で時系列データを抽出します。
- **セル1**: fmriprep処理済みのNIfTIファイルを読み込み、NiftiLabelsMaskerを用いて各パーセルの時系列を抽出。タスク選択時（piemanpni/bronx）に有効区間を指定して切り出し、CSV形式で保存
- **セル2**: 抽出したパーセルデータをMin-Max正規化（0～1範囲）し、後続のモデル学習用に統一フォーマットで保存
- **出力**: `parcels_csv/{task}_parcels_all_subjects.csv`, `parcels_csv/{task}_parcels_all_subjects_normalized.csv`

### 2. `code/2_emb_bronx_pkl.ipynb`
**Bronxタスク向けの言語埋め込み生成**

複数の事前学習済み言語モデル（Word2Vec、GPT-2、GTE、Llama、GPT-OSS など）を用いて、物語トランスクリプト中の単語を埋め込みベクトルに変換します。
- **セル1**: Word2Vecモデル（300次元）の読み込みと単語埋め込みの抽出、トランスクリプトCSVへの埋め込み列の追加、pickle形式での保存
- **セル2**: Transformerベースの各モデル（GPT-2: 768次元、GTE: 1024次元、Llama: 4096次元など）のエンコーダー構築と埋め込み生成の関数定義
- **セル3**: 各モデルの埋め込みを実際に計算し、`construct_predictors` 関数を用いてTR（時間解像度）単位に平均化した埋め込み行列を生成
- **出力**: `emb/bronx_w2v.pkl`, `emb/bronx_gpt2.pkl`, `emb/bronx_gte.pkl`, `emb/bronx_llama.pkl`, `emb/bronx_llama3.pkl`, `emb/bronx_gpt_oss.pkl`

### 3. `code/2_emb_piemanpni_pkl.ipynb`
**Piemanpniタスク向けの言語埋め込み生成**

Bronxと同様の処理をPiemanpniタスク用に実施します。異なるトランスクリプトと刺激時間（400TR、Bronxは357TR）に対応。
- 処理フロー・関数は`2_emb_bronx_pkl.ipynb`と同一
- **出力**: `emb/piemanpni_w2v.pkl`, `emb/piemanpni_gpt2.pkl`, `emb/piemanpni_gte.pkl`, `emb/piemanpni_llama.pkl`, `emb/piemanpni_llama3.pkl`, `emb/piemanpni_gpt_oss.pkl`

### 4. `code/3_Banded_Ridge_bronx.ipynb`
**Bronxデータのバンディッドリッジ回帰分析**

言語埋め込みからfMRI活動を予測するバンディッドリッジ回帰モデルを学習。各言語モデルの予測精度をISC（Inter-Subject Correlation）指標で評価。
- **セル1**: 必要なライブラリの導入、埋め込みデータの読み込み、Bronxタスクのパーセル活動データの整備
- **セル2**: バンディッドリッジ回帰モデルの学習（言語埋め込み → fMRI活動の線形予測）、交差検証による最適ハイパーパラメータ選択
- **セル3**: 各言語モデル×被験者×パーセル単位での相関計算：
  - `pred_vs_group`: 予測活動 vs グループ平均活動の相関
  - `subj_vs_group`: 個人の実測活動 vs グループ平均活動の相関
  - `pred_vs_actual`: 予測活動 vs 個人の実測活動の相関
  - `pred_vs_other_story`: 予測活動 vs 他タスク（Piemanpni）の実測活動の相関（転移学習の評価）
  - その他複数の相関指標をISC関数で計算
- **セル4・5**: 相関結果の集計、パーセル×ネットワーク単位での可視化、統計指標の計算
- **出力**: `output_csv/bronx_*_pred*.csv` （モデルごとの予測結果・相関指標）

### 5. `code/3_Banded_Ridge_piemanpni.ipynb`
**Piemanpniデータのバンディッドリッジ回帰分析**

Bronxと同様の回帰分析をPiemanpniタスクに適用。学習と評価の構造は同一で、トランスクリプト・パーセル活動・刺激時間が異なります。
- **出力**: `output_csv/piemanpni_*_pred*.csv`

### 6. `code/4_predicted_normarized.ipynb`
**予測結果の正規化と評価**

バンディッドリッジ回帰で得られた予測値を、元のfMRI活動と同じスケールに正規化し、モデル間の予測精度を統一的に比較。

### 7. `code/5_all_corr_output_bronx.ipynb`
**Bronx向け相関解析の出力と集計**

バンディッドリッジ回帰から得られた多種類の相関指標（pred_vs_group、subj_vs_group、pred_vs_actual、pred_vs_other_story など）を、モデルごと・パーセルごと・被験者ごとに集計し、CSV形式で出力。
- 各モデルの平均相関値の計算と比較
- パーセルレベルでの相関分布のヒストグラム生成
- 被験者レベルでの相関平均の計算
- **出力**: `analysis_corr2/{w2v|gpt2|gte|llama|llama3|gpt_oss}_bronx_*_mean_corr.csv`

### 8. `code/5_all_corr_output_piemanpni.ipynb`
**Piemanpni向け相関解析の出力と集計**

7番と同様の処理をPiemanpniタスクに適用。
- **出力**: `analysis_corr2/{w2v|gpt2|gte|llama|llama3|gpt_oss}_piemanpni_*_mean_corr.csv`

### 9. `code/6_analysis.ipynb`
**全体的な相関解析と統計**

Bronx・Piemanpni両タスクから得られた相関データを統合し、モデル間の性能比較、統計的有意性検定を実施。
- 各モデル・各相関指標の平均値と標準偏差の計算
- モデル間の差の有意性検定（t検定など）
- パーセルごとの相関分布の可視化（ヒストグラム、ボックスプロット）
- ネットワーク別の相関パターン分析

### 10. `code/6_squared_analysis.ipynb`
**相関の2乗（説明分散）の分析**

相関係数を2乗することで、各言語モデルがfMRI活動の分散をどの程度説明できるかを定量化。
- 説明分散（$R^2$）の計算：相関値2乗 = 説明分散の割合
- モデル別・パーセル別・ネットワーク別の$R^2$分布の分析
- より詳細な性能比較が可能

### 11. `code/7_bootstrap_analysis.ipynb`
**ブートストラップ再サンプリング解析**

被験者をランダムに復元抽出し、相関指標の頑健性と信頼区間を推定。
- 複数回のブートストラップサンプリング（例：1000回）で被験者サブセットを生成
- 各サンプルについてモデル性能を再計算
- 信頼区間（95% CI）の構築、統計的な不確実性の評価
- モデル間の有意差がブートストラップでも再現されるかの検証

## セットアップ

1. Python仮想環境を作成

```bash
python3 -m venv venv
source venv/bin/activate
```

2. 依存パッケージをインストール

```bash
pip install -r requirements.txt
```

3. Jupyter Notebookを起動

```bash
jupyter lab
```

## 注意事項

- ノートブック内にAPIトークンや秘密情報を直接含めないでください。
- トークンを使う場合は環境変数や別ファイルで管理し、`.gitignore` に追加して公開リポジトリに含めないようにしてください。

## 追加情報

- `analysis_corr2/` 以下には、解析結果の保存用フォルダが作成されています。
- 空ディレクトリをGitで管理するために、各ディレクトリに `.gitkeep` が追加されています。
