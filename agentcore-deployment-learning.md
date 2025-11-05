# Amazon Bedrock AgentCore デプロイメント学習レポート

**学習日:** 2025-11-06
**対象:** `02_runtime` ディレクトリの `prepare_agent.py` と AgentCore デプロイメントプロセス

---

## 1. 02_runtime ディレクトリの構造

### ディレクトリ構成
```
02_runtime/
├── prepare_agent.py              # Agent準備ツール
├── clean_resources.py            # リソースクリーンアップ
├── README.md                     # 英語ドキュメント
├── README_ja.md                  # 日本語ドキュメント
├── __init__.py
├── .gitignore
├── .dockerignore
└── deployment/                   # デプロイメント用テンプレート
    ├── invoke.py                 # 同期型エントリーポイント
    ├── invoke_async.py           # 非同期型エントリーポイント
    └── requirements.txt          # Python依存パッケージ
```

### 主要ファイルの役割

| ファイル | 説明 |
|---------|------|
| `prepare_agent.py` | ソースコード配置とIAMロール作成を自動化 |
| `clean_resources.py` | デプロイされたAgentCoreリソースを削除 |
| `deployment/invoke.py` | 同期実行のエントリーポイント |
| `deployment/invoke_async.py` | 非同期ストリーミング実行のエントリーポイント |

---

## 2. prepare_agent.py の機序

### デプロイメント全体フロー（3段階）

```
段階1: prepare_agent.py        段階2: agentcore CLI      段階3: agentcore CLI
(ローカル準備)                 (設定・構築)              (起動・実行)
       ↓                            ↓                        ↓
1. ソースコピー            →  2. configure         →  3. launch / invoke
2. IAMロール作成               (AgentCore設定)
3. configコマンド生成
```

### 段階1の詳細処理

#### 1-1. ソースディレクトリのコピー (`create_source_directory()`)
- `deployment/` ディレクトリを作成
- ソースディレクトリから全ての `*.py` ファイルをコピー
- 例: `01_code_interpreter/cost_estimator_agent/*.py` → `deployment/cost_estimator_agent/`

**コード位置:** `prepare_agent.py:74-99`

#### 1-2. IAM ロールの作成 (`create_agentcore_role()`)
**コード位置:** `prepare_agent.py:101-299`

作成される権限：
- **Bedrock権限:** `bedrock:InvokeModel`, `bedrock:InvokeModelWithResponseStream`
- **ECR権限:** イメージの取得とトークン管理
- **CloudWatch権限:** ロギングとメトリクス出力
- **X-Ray権限:** トレーシング機能
- **AgentCore権限:** アクセストークン取得、Code Interpreter操作
- **Pricing権限:** 料金API利用（コストエスティメーター用）

Trust Policy により、`bedrock-agentcore.amazonaws.com` がこのロールを引き受けることを許可。

#### 1-3. agentcore configureコマンドの生成
**コード位置:** `prepare_agent.py:49-72`

```python
command = "\n".join([
    "",
    f"uv run agentcore configure --entrypoint {deployment_dir}/invoke.py \\",
    f"--name {self.agent_name} \\",
    f"--execution-role {role_info['role_arn']} \\",
    f"--requirements-file {deployment_dir}/requirements.txt \\",
    f"--region {self.region} "
])
```

### 出力例

```bash
uv run agentcore configure --entrypoint deployment/invoke.py \
--name cost_estimator_agent \
--execution-role arn:aws:iam::123456789012:role/AgentCoreRole-cost_estimator_agent \
--requirements-file deployment/requirements.txt \
--region ap-northeast-1
```

このコマンドはユーザーに表示され、ユーザーが手動で実行する必要があります。

---

## 3. uv コマンドの理解

### `uv run` の意味

```
uv run agentcore configure
│    │   │          │
│    │   │          └─ AgentCore設定処理を実行
│    │   └─ Bedrockパッケージに含まれるコマンド
│    └─ Pythonツールを実行
└─ パッケージマネージャー
```

### 処理フロー

1. `uv` が `requirements.txt` を読み込む
2. 必要なパッケージが存在するか確認
   - インストール済み → キャッシュから読み込み（高速）
   - 未インストール → インストール実施
3. インストール済みパッケージ内の CLI コマンドを実行

### `requirements.txt` 内のエントリーポイント

```
bedrock-agentcore>=0.1.0  ← このパッケージに agentcore コマンドが含まれる
```

パッケージ内部の `setup.py` または `pyproject.toml` で以下のように定義されている：

```
entry_points: {
    'console_scripts': {
        'agentcore': 'bedrock_agentcore.cli:main'
    }
}
```

インストール時に、このエントリーポイント定義により `agentcore` という実行可能コマンドが登録される。

### キャッシュ機構

- 初回実行: パッケージをインストール（数秒）
- 2回目以降: キャッシュから復元（1秒未満）
- キャッシュ場所: `~/.cache/uv/` (Linux/Mac)

---

## 4. agentcore configure コマンドの動作

### 処理フロー

```
agentcore configure 実行
       ↓
1. 設定ファイル生成 (.bedrock_agentcore.yaml)
       ↓
2. ECR リポジトリ作成
       ↓
3. Dockerfile 生成
       ↓
4. Docker イメージビルド
       ↓
5. Docker イメージを ECR にプッシュ
       ↓
6. AgentCore Runtime に登録
```

### 各ステップの詳細

#### ステップ1: 設定ファイル生成
生成ファイル: `.bedrock_agentcore.yaml`
```yaml
name: cost_estimator_agent
entrypoint: deployment/invoke.py
execution_role_arn: arn:aws:iam::123456789012:role/AgentCoreRole-cost_estimator_agent
requirements_file: deployment/requirements.txt
region: ap-northeast-1
```

#### ステップ2-3: ECR リポジトリ作成と Dockerfile 生成
- AWS ECR にリポジトリを作成
- Python アプリケーション用の Dockerfile を自動生成

#### ステップ4-5: Docker ビルドと ECR プッシュ
```bash
docker build -t cost_estimator_agent:latest .
docker push 123456789012.dkr.ecr.ap-northeast-1.amazonaws.com/cost_estimator_agent:latest
```

#### ステップ6: AgentCore Runtime に登録
AWS Bedrock AgentCore API を呼び出して Runtime として登録

### 重要な依存関係

**Docker がインストールされていることが必須です。**

`agentcore configure` 実行時に Docker コマンドを内部で呼び出すため、ローカルマシンに Docker がインストール済みである必要があります。

---

## 5. デプロイメント全体の処理フロー比較

| フェーズ | 実行ツール | 役割 |
|---------|-----------|------|
| **準備** | `prepare_agent.py` | ソースコピー、IAMロール作成、configコマンド生成 |
| **設定・構築** | `agentcore configure` | Dockerファイル生成、イメージビルド、ECR登録、Runtime登録 |
| **起動** | `agentcore launch` | AgentCore Runtime 起動 |
| **実行** | `agentcore invoke` | エージェント実行 |

---

## 6. 環境要件の確認結果

### Docker のインストール状況
```bash
$ uv run docker --version
error: Failed to spawn: `docker`
  Caused by: No such file or directory (os error 2)
```

**現在のマシンには Docker がインストールされていません。**

### 必要な対応
`agentcore configure` を実行する前に、Docker をインストールする必要があります：
- **macOS/Windows:** Docker Desktop をインストール
- **Linux:** `sudo apt-get install docker.io` コマンドでインストール

---

## 学習まとめ

### キーポイント

1. **prepare_agent.py は準備工具**
   - デプロイの直接的な実行はしない
   - IAMロール作成とソースコピーのみ実施
   - ユーザーが実行すべき `agentcore configure` コマンドを生成

2. **uv run は毎回requirements.txtを参照**
   - キャッシュにより2回目以降は高速
   - 環境の一貫性が保証される

3. **agentcore configure が複雑な処理を自動化**
   - Dockerイメージのビルドから ECR プッシュまで自動化
   - AWS リソース作成（ECR リポジトリ、Runtime登録）も実施

4. **Docker は必須**
   - ローカルマシンにインストール必須
   - `agentcore configure` 実行時に内部的に使用される

### デプロイメント の流れ理解
```
ローカル作業（prepare_agent.py）
        ↓
ユーザー実行（agentcore configure）
        ↓
AWS側で実行・登録
        ↓
実行可能状態（agentcore launch / invoke）
```

---

**作成日:** 2025-11-06
