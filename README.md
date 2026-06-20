## Development setup

このプロジェクトは、ローカルのMacBook上ではVS Codeで編集し、重い処理はGoogle ColabのRuntimeで実行する構成を前提としています。

### Architecture

```text
MacBook Air
  └─ VS Code
      ├─ コード編集
      ├─ Git操作
      ├─ 軽いテストの実行
      └─ Colab拡張でColab Runtimeに接続

Google Colab
  ├─ notebookの実行
  ├─ GPU / 大きめRAMが必要な処理
  └─ ML学習・RAG前処理などの重い処理

GitHub
  └─ ソースコードの正本

Google Drive
  └─ データセット・出力結果・モデル・ログの保存先
```

## Local development

ローカルでは `uv` を使って環境を作成します。

```bash
git clone https://github.com/sihasi/colab.git
cd colab

uv sync
```

軽いテストや静的チェックはローカルで実行します。

```bash
uv run pytest
uv run ruff check .
```

大きなデータ処理、GPUを使う処理、メモリを多く使う処理はローカルでは実行せず、Colab Runtimeで実行します。

## Using Google Colab from VS Code

VS CodeからGoogle Colab Runtimeに接続してnotebookを実行します。

### Required VS Code extensions

* Google Colab
* Jupyter
* Python

### Basic workflow

1. VS Codeで `.ipynb` を開く
2. 右上の `Select Kernel` をクリック
3. `Colab` を選択
4. Googleアカウントでサインイン
5. Colab Runtimeに接続してセルを実行する

## Colab setup

Colab Runtimeは一時的な環境なので、必要なパッケージやコードはnotebookの冒頭で毎回セットアップします。

### 1. Mount Google Drive

```python
from google.colab import drive
drive.mount("/content/drive")
```

### 2. Clone repository and install dependencies

Colabでは `uv sync` ではなく、Colabの現在のPython環境に直接インストールします。

```python
%cd /content

!rm -rf colab
!git clone https://github.com/sihasi/colab.git

%cd colab

!uv pip install --system -r requirements-colab.txt
!uv pip install --system -e .
```

### 3. Enable autoreload

開発中に `src/` 配下のPythonコードを変更する場合は、autoreloadを有効にします。

```python
%load_ext autoreload
%autoreload 2
```

## Running experiments

notebookにはロジックを直接書きすぎず、`src/` 配下の関数を呼び出す形にします。

```python
from <your_package>.train import train

train(
    data_path="/content/drive/MyDrive/datasets/sample.csv",
    output_dir="/content/drive/MyDrive/<your-repo>/outputs/exp001",
)
```

## Recommended project structure

```text
<your-repo>/
  ├─ src/
  │   └─ <your_package>/
  │       ├─ __init__.py
  │       ├─ data.py
  │       ├─ model.py
  │       ├─ train.py
  │       └─ evaluate.py
  │
  ├─ notebooks/
  │   ├─ 001_colab_train.ipynb
  │   └─ 002_rag_preprocess.ipynb
  │
  ├─ scripts/
  │   ├─ train.py
  │   └─ preprocess_docs.py
  │
  ├─ tests/
  │   └─ test_data.py
  │
  ├─ pyproject.toml
  ├─ uv.lock
  ├─ requirements-colab.txt
  ├─ .gitignore
  └─ README.md
```

## Dependency management

ローカルでは `uv sync` を使います。

```bash
uv sync
```

Colabでは `uv pip install --system` を使います。

```python
!uv pip install --system -r requirements-colab.txt
!uv pip install --system -e .
```

Colab上で `uv sync` を使うと `.venv` が作成されますが、Jupyter kernelがその仮想環境を使うとは限らないため、基本的には使用しません。

## Exporting dependencies for Colab

必要に応じて、ローカルでColab用のrequirementsを出力します。

```bash
uv export --format requirements-txt --output-file requirements-colab.txt
```

ただし、ColabにはPyTorchなどが最初から入っている場合があるため、GPU関連ライブラリは必要に応じてColab側で確認してから追加します。

```python
import torch

print(torch.__version__)
print(torch.cuda.is_available())
print(torch.cuda.get_device_name(0) if torch.cuda.is_available() else "no gpu")
```

## Resetting Colab Runtime

Colab側の状態をリセットしたい場合は、目的に応じて使い分けます。

### Restart kernel

変数、import状態、GPUメモリだけをリセットしたい場合。

```text
Command Palette
→ Jupyter: Restart Kernel
```

### Remove Colab server

パッケージ環境や `/content` 配下のファイルも含めて初期化したい場合。

```text
Command Palette
→ Colab: Remove Server
```

依存関係が壊れた場合や、環境を作り直したい場合は `Colab: Remove Server` を使います。

## Notes

* コードの正本はGitHubに置く
* データセットや学習結果はGoogle Driveに置く
* notebookにはロジックを書きすぎない
* 本体ロジックは `src/` 配下に置く
* ローカルMacでは軽いテストだけ実行する
* 重い処理はColab Runtimeで実行する
* Colab Runtimeは一時的な環境なので、冒頭セルで再現可能にする
