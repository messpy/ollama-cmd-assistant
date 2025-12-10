# ollama-cmd-assistant

シェルコマンドの実行結果を Ollama の LLM で日本語解説してくれるコマンドラインアシスタント

## 概要

`olm` は、シェルコマンドの実行結果を自動的に解析し、日本語で分かりやすく解説してくれるツールです。エラーが発生した場合は、原因の診断、改善案、実行可能なコマンド候補まで提案します。

### 主な機能

- **コマンド実行結果の自動解説**: コマンドを実行し、その結果を日本語で簡潔に説明
- **エラー診断**: 失敗したコマンドについて、原因・改善案・代替コマンドを提案
- **インテリジェントな診断情報付加**: 
  - `command not found` の場合は PATH やタイポ候補を調査
  - `No such file` の場合はディレクトリ構造を確認
  - Python エラーの場合はモジュール情報を取得
  - ネットワークエラーの場合は接続テストを実施
- **標準入力解析モード** (`--analyze`): 複数のコマンドをまとめて実行・解析
- **複数 LLM モデル対応**: 
  - 通常実行: cloud → light モデル
  - エラー時: cloud → heavy モデル
  - フォールバックモデルによる冗長性

## 必要要件

- **Ollama**: LLM 実行環境
  - インストール: https://ollama.ai/
  - `ollama serve` が自動起動されます
- **Bash**: シェル環境（Linux / macOS / WSL）

## インストール

1. リポジトリをクローン:
```bash
git clone https://github.com/messpy/ollama-cmd-assistant.git
cd ollama-cmd-assistant
```

2. `olm` スクリプトに実行権限を付与:
```bash
chmod +x olm
```

3. パスを通す（オプション）:
```bash
# ~/.bashrc や ~/.zshrc に追加
export PATH="$PATH:/path/to/ollama-cmd-assistant"
```

または、シンボリックリンクを作成:
```bash
sudo ln -s /path/to/ollama-cmd-assistant/olm /usr/local/bin/olm
```

## 使い方

### 基本的な使い方

```bash
# コマンドを実行して結果を解説
olm ls -la

# エラーが出るコマンドを診断
olm python3 missing_script.py

# 詳細モード（成功時でも heavy モデルを使用）
olm -d docker ps

# 追加プロンプトで指示を追加
olm -p "セキュリティの観点で解説して" cat /etc/passwd
```

### 標準入力解析モード

複数のコマンドをまとめて実行・解析する場合:

```bash
olm --analyze
```

実行後、コマンドを入力（2行連続の空行で終了）:
```bash
ls -la
ps aux | grep nginx

```

コメントを付けてプロンプトに追加指示も可能:
```bash
olm --analyze
df -h  # ディスク容量を確認
free -m  # メモリ使用量も見たい

```

### ヘルプとログ

```bash
# ヘルプ表示
olm --help

# ログ確認
olm --logs
```

## オプション

| オプション | 説明 |
|-----------|------|
| `--help`, `-h` | ヘルプメッセージを表示 |
| `--logs` | 過去の実行ログを表示（最新40行） |
| `--analyze` | 標準入力から複数コマンドを読み取り、実行してまとめて解析 |
| `-d` | 詳細モード（成功時でも heavy モデルを使用） |
| `-p "指示"` | LLM への追加指示を指定 |

## 環境変数

以下の環境変数でカスタマイズ可能:

### ログ設定
```bash
export OLM_LOG_DIR="$HOME/log"              # ログディレクトリ
export OLM_LOG_FILE="$OLM_LOG_DIR/olm.log" # ログファイルパス
```

**注意**: `OLM_LOG_FILE` の設定時には、`OLM_LOG_DIR` が既に定義されている必要があります。

### 出力制限
```bash
export OLM_CMD_MAX_CHARS=1000      # コマンド出力の最大文字数
export OLM_LLM_MAX_CHARS=1000      # LLM 出力の最大文字数
export OLM_MAX_LINE_CHARS=80       # 1行あたりの推奨文字数
export OLM_MAX_EXPLAIN_CHARS=400   # 解説の最大文字数
```

### コマンド候補数
```bash
export OLM_MAX_SUGGEST_CMDS=5      # 提案コマンドの最大数
export OLM_MAX_CONFIRM_CMDS=3      # 確認コマンドの最大数
```

### LLM モデル設定
```bash
export OLM_MODEL_STD="gpt-oss:20b-cloud"                  # 標準モデル (cloud)
export OLM_MODEL_LGT="phi3:3.8b-mini-4k-instruct-q4_k_m" # ライトモデル
export OLM_MODEL_HVY="llama3:8b"                          # ヘビーモデル
export OLM_MODEL_FA="qwen2.5:0.5b"                        # フォールバック（ライト）
export OLM_MODEL_FB="mistral-nemo:latest"                 # フォールバック（ヘビー）
```

## 使用例

### 例1: ファイルが見つからないエラー
```bash
$ olm cat /etc/nonexistent.conf
cat: /etc/nonexistent.conf: No such file or directory

解説=:
指定されたファイル /etc/nonexistent.conf が存在しないため cat コマンドが失敗しました。

エラー原因(3行)=:
1. ファイルパスが誤っている
2. ファイルが削除されている
3. 別の場所に配置されている可能性

改善案(3行)=:
1. ファイルパスを確認する
2. find コマンドで該当ファイルを検索
3. 類似する設定ファイルを探す

コマンド候補=:
# /etc 配下で conf ファイルを検索
$ find /etc -name "*.conf" -type f 2>/dev/null

確認コマンド=:
$ ls -la /etc/
```

### 例2: Python モジュールが見つからない
```bash
$ olm python3 -c "import requests"
Traceback (most recent call last):
  File "<string>", line 1, in <module>
ModuleNotFoundError: No module named 'requests'

解説=:
requests モジュールがインストールされていないため import に失敗しました。

エラー原因(3行)=:
1. requests ライブラリが未インストール
2. 仮想環境が有効化されていない可能性
3. pip でインストールが必要

改善案(3行)=:
1. pip で requests をインストール
2. 仮想環境を確認・有効化
3. requirements.txt があれば一括インストール

コマンド候補=:
# requests をインストール
$ pip3 install requests

確認コマンド=:
$ pip3 list | grep requests
$ which python3
```

### 例3: 成功したコマンドの解説
```bash
$ olm docker ps
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES

解説=:
現在実行中の Docker コンテナを表示しましたが、稼働中のコンテナはありませんでした。

コマンド候補=:
# 停止中のコンテナも含めて確認
$ docker ps -a

確認コマンド=:
$ docker info
```

## LLM モデルの動作

### モデル選択ロジック

1. **通常実行（成功時）**:
   - cloud モデル → light モデル → フォールバック（light）

2. **エラー時**:
   - cloud モデル → heavy モデル → フォールバック（heavy）

3. **--analyze モード**:
   - cloud モデル → heavy モデル → フォールバック（heavy）

4. **詳細モード（-d）**:
   - cloud モデル → heavy モデル → フォールバック（heavy）

### フォールバック機能

上位モデルが利用できない場合、自動的に次のモデルへフォールバックします。

## ログ

すべての実行履歴は `$HOME/log/olm.log`（デフォルト）に記録されます:

```bash
# ログ確認
olm --logs

# ログファイルの場所
echo $OLM_LOG_FILE
```

ログには以下の情報が記録されます:
- 実行日時
- コマンド
- 終了ステータス
- 出力内容
- 使用した LLM モデル
- 解説内容

## ライセンス

MIT License - 詳細は [LICENSE](LICENSE) を参照してください。

## 作者

Copyright (c) 2025 messsykenty

## 貢献

Issue や Pull Request を歓迎します！

## 関連リンク

- [Ollama](https://ollama.ai/) - LLM 実行環境
- [GitHub Repository](https://github.com/messpy/ollama-cmd-assistant)