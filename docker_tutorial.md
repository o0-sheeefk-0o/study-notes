# Docker 完全ガイド - 基礎から実践まで

## 目次
1. [Dockerとは](#dockerとは)
2. [Dockerのシステム構造](#dockerのシステム構造)
3. [基本的な操作](#基本的な操作)
4. [実践: FastAPI + PostgreSQL環境の構築](#実践-fastapi--postgresql環境の構築)
5. [よく使うコマンド一覧](#よく使うコマンド一覧)
6. [トラブルシューティング](#トラブルシューティング)

---

## Dockerとは

Dockerは、アプリケーションを「コンテナ」という単位で動かすためのツールです。

### 簡単な例え話
- **従来の方法**: 引っ越しのとき、家具をバラバラに運ぶ
- **Docker**: 家具も家電も全部入った「コンテナ」ごと運ぶ

### Dockerの利点

1. **環境の違いに悩まされない**
   - 「自分のパソコンでは動くのに、サーバーでは動かない...」という問題を解決

2. **セットアップが簡単**
   - 面倒な手順がコマンド1つで済む

3. **軽量で速い**
   - 仮想マシン（VM）より起動が速く、リソースも少ない

### 基本用語

| 用語 | 説明 |
|------|------|
| **イメージ** | アプリの設計図・レシピのようなもの |
| **コンテナ** | イメージから作られた、実際に動いている環境 |
| **Dockerfile** | イメージの作り方を書いたレシピファイル |
| **Docker Hub** | クラウド上のイメージ倉庫 |

---

## Dockerのシステム構造

```
┌─────────────────────────────────────┐
│     あなた（ユーザー）                │
└──────────┬──────────────────────────┘
           │ docker コマンド
           ↓
┌─────────────────────────────────────┐
│    Docker CLI（コマンドライン）        │
└──────────┬──────────────────────────┘
           │
           ↓
┌─────────────────────────────────────┐
│   Docker Engine（エンジン本体）        │
│  ┌─────────────────────────────┐   │
│  │  Docker Daemon（常駐プログラム）│  │
│  │  - コンテナの管理             │   │
│  │  - イメージの管理             │   │
│  │  - ネットワークの管理          │   │
│  └─────────────────────────────┘   │
└──────────┬──────────────────────────┘
           │
           ↓
┌─────────────────────────────────────┐
│        ホストOS（あなたのPC）          │
└─────────────────────────────────────┘
```

### イメージとコンテナの関係

```
Dockerfile（レシピ）
    ↓ docker build
Docker Image（設計図）
    ↓ docker run
Container（実際に動く環境）
```

### Docker Hubとの関係

```
Docker Hub（クラウド上のイメージ倉庫）
    ↓ docker pull（ダウンロード）
ローカルのイメージ
    ↓ docker run
コンテナ
```

---

## 基本的な操作

### 1. インストール確認

```bash
# Dockerがインストールされているか確認
docker --version

# より詳細な情報
docker info
```

### 2. イメージを取得する

```bash
# Docker Hubから公式のNginxイメージを取得
docker pull nginx

# ダウンロードしたイメージを確認
docker images
```

**出力例:**
```
REPOSITORY   TAG       IMAGE ID       CREATED        SIZE
nginx        latest    abc123def456   2 weeks ago    187MB
```

### 3. コンテナを起動する

```bash
# Nginxコンテナを起動
docker run -d -p 8080:80 --name my-nginx nginx
```

**オプションの説明:**
- `-d`: バックグラウンドで実行
- `-p 8080:80`: ホストの8080ポートをコンテナの80ポートに接続
- `--name`: コンテナに名前をつける

**確認方法:**
ブラウザで `http://localhost:8080` にアクセス

### 4. コンテナの状態を確認

```bash
# 実行中のコンテナを表示
docker ps

# 全てのコンテナを表示（停止中も含む）
docker ps -a
```

**出力例:**
```
CONTAINER ID   IMAGE   COMMAND                  STATUS         PORTS                  NAMES
abc123def456   nginx   "nginx -g 'daemon of…"   Up 2 minutes   0.0.0.0:8080->80/tcp   my-nginx
```

### 5. コンテナ内部を操作

```bash
# コンテナの中に入る（シェルを起動）
docker exec -it my-nginx bash

# 中でコマンドを実行できる
# 例: ls, cat /etc/nginx/nginx.conf など
# 終了するには exit と入力
```

### 6. ログを確認

```bash
# コンテナのログを表示
docker logs my-nginx

# ログをリアルタイムで追跡
docker logs -f my-nginx
```

### 7. コンテナを停止・削除

```bash
# コンテナを停止
docker stop my-nginx

# コンテナを削除
docker rm my-nginx

# イメージを削除
docker rmi nginx
```

---

## 実践例: 簡単なWebアプリをDockerで動かす

### プロジェクト構造
```
my-web-app/
├── Dockerfile
└── index.html
```

### 手順

#### 1. 作業ディレクトリを作成

```bash
mkdir my-web-app
cd my-web-app
```

#### 2. HTMLファイルを作成

**index.html**
```html
<!DOCTYPE html>
<html>
<head>
    <title>My Docker App</title>
</head>
<body>
    <h1>Hello from Docker!</h1>
    <p>これは私のDockerアプリです</p>
</body>
</html>
```

#### 3. Dockerfileを作成

**Dockerfile**
```dockerfile
# ベースイメージを指定
FROM nginx:alpine

# HTMLファイルをコンテナにコピー
COPY index.html /usr/share/nginx/html/index.html

# ポート80を公開
EXPOSE 80
```

#### 4. イメージをビルド

```bash
# カレントディレクトリのDockerfileからイメージを作成
docker build -t my-web-app:1.0 .

# イメージが作成されたか確認
docker images
```

#### 5. コンテナを起動

```bash
docker run -d -p 8080:80 --name my-app my-web-app:1.0
```

**確認:**
`http://localhost:8080` にアクセスして、自分のHTMLが表示されることを確認

---

## 実践: FastAPI + PostgreSQL環境の構築

### プロジェクト構造

```
my-fastapi-project/
├── docker-compose.yml
├── Dockerfile
├── requirements.txt
├── .env
└── app/
    ├── __init__.py
    ├── main.py
    └── database.py
```

### ステップ1: プロジェクトディレクトリを作成

```bash
mkdir my-fastapi-project
cd my-fastapi-project
mkdir app
```

### ステップ2: 必要なファイルを作成

#### requirements.txt

```txt
fastapi==0.104.1
uvicorn[standard]==0.24.0
sqlalchemy==2.0.23
psycopg2-binary==2.9.9
python-dotenv==1.0.0
```

#### .env

```env
# データベース設定
POSTGRES_USER=myuser
POSTGRES_PASSWORD=mypassword
POSTGRES_DB=mydb
POSTGRES_HOST=db
POSTGRES_PORT=5432

# アプリケーション設定
DATABASE_URL=postgresql://myuser:mypassword@db:5432/mydb
```

#### Dockerfile

```dockerfile
# Python 3.11のベースイメージ
FROM python:3.11-slim

# 作業ディレクトリを設定
WORKDIR /app

# 依存パッケージファイルをコピー
COPY requirements.txt .

# 依存パッケージをインストール
RUN pip install --no-cache-dir -r requirements.txt

# アプリケーションコードをコピー
COPY ./app ./app

# ポート8000を公開
EXPOSE 8000

# アプリケーションを起動
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000", "--reload"]
```

#### docker-compose.yml

```yaml
version: '3.8'

services:
  # FastAPIアプリケーション
  web:
    build: .
    container_name: fastapi-app
    ports:
      - "8000:8000"
    volumes:
      # ローカルのappディレクトリをコンテナにマウント（開発時の自動リロード用）
      - ./app:/app/app
    env_file:
      - .env
    depends_on:
      - db
    networks:
      - app-network

  # PostgreSQLデータベース
  db:
    image: postgres:15-alpine
    container_name: postgres-db
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
    ports:
      - "5432:5432"
    volumes:
      # データの永続化
      - postgres-data:/var/lib/postgresql/data
    networks:
      - app-network

  # pgAdmin（オプション：データベース管理ツール）
  pgadmin:
    image: dpage/pgadmin4:latest
    container_name: pgadmin
    environment:
      PGADMIN_DEFAULT_EMAIL: admin@example.com
      PGADMIN_DEFAULT_PASSWORD: admin
    ports:
      - "5050:80"
    depends_on:
      - db
    networks:
      - app-network

# ボリューム定義（データ永続化用）
volumes:
  postgres-data:

# ネットワーク定義
networks:
  app-network:
    driver: bridge
```

#### app/__init__.py

```python
# 空のファイルでOK（appをPythonパッケージとして認識させるため）
```

#### app/database.py

```python
from sqlalchemy import create_engine, Column, Integer, String
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker
import os

# 環境変数からデータベースURLを取得
DATABASE_URL = os.getenv("DATABASE_URL", "postgresql://myuser:mypassword@db:5432/mydb")

# データベースエンジンを作成
engine = create_engine(DATABASE_URL)

# セッションを作成
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

# ベースクラス
Base = declarative_base()

# データベース接続の依存性
def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

# サンプルモデル
class User(Base):
    __tablename__ = "users"
    
    id = Column(Integer, primary_key=True, index=True)
    name = Column(String, index=True)
    email = Column(String, unique=True, index=True)
```

#### app/main.py

```python
from fastapi import FastAPI, Depends, HTTPException
from sqlalchemy.orm import Session
from pydantic import BaseModel
from typing import List

from app.database import engine, Base, get_db, User

# テーブルを作成
Base.metadata.create_all(bind=engine)

app = FastAPI(title="My FastAPI App")

# Pydanticモデル（リクエスト/レスポンス用）
class UserCreate(BaseModel):
    name: str
    email: str

class UserResponse(BaseModel):
    id: int
    name: str
    email: str
    
    class Config:
        from_attributes = True

# ルート
@app.get("/")
def read_root():
    return {
        "message": "FastAPI + PostgreSQL Docker環境へようこそ！",
        "endpoints": {
            "users": "/users",
            "create_user": "POST /users",
            "health": "/health"
        }
    }

@app.get("/health")
def health_check():
    return {"status": "healthy"}

@app.post("/users", response_model=UserResponse)
def create_user(user: UserCreate, db: Session = Depends(get_db)):
    # 既存のユーザーをチェック
    db_user = db.query(User).filter(User.email == user.email).first()
    if db_user:
        raise HTTPException(status_code=400, detail="Email already registered")
    
    # 新しいユーザーを作成
    new_user = User(name=user.name, email=user.email)
    db.add(new_user)
    db.commit()
    db.refresh(new_user)
    return new_user

@app.get("/users", response_model=List[UserResponse])
def get_users(skip: int = 0, limit: int = 10, db: Session = Depends(get_db)):
    users = db.query(User).offset(skip).limit(limit).all()
    return users

@app.get("/users/{user_id}", response_model=UserResponse)
def get_user(user_id: int, db: Session = Depends(get_db)):
    user = db.query(User).filter(User.id == user_id).first()
    if user is None:
        raise HTTPException(status_code=404, detail="User not found")
    return user

@app.delete("/users/{user_id}")
def delete_user(user_id: int, db: Session = Depends(get_db)):
    user = db.query(User).filter(User.id == user_id).first()
    if user is None:
        raise HTTPException(status_code=404, detail="User not found")
    db.delete(user)
    db.commit()
    return {"message": "User deleted successfully"}
```

### ステップ3: コンテナを起動

```bash
# バックグラウンドで起動
docker-compose up -d

# ログを確認
docker-compose logs -f
```

### ステップ4: 動作確認

#### 1. ブラウザでアクセス
```
http://localhost:8000
```

#### 2. API ドキュメントを確認
FastAPIは自動でAPIドキュメントを生成します：
```
http://localhost:8000/docs
```

#### 3. curlでテスト

```bash
# ユーザーを作成
curl -X POST "http://localhost:8000/users" \
  -H "Content-Type: application/json" \
  -d '{"name": "山田太郎", "email": "yamada@example.com"}'

# ユーザー一覧を取得
curl http://localhost:8000/users

# 特定のユーザーを取得
curl http://localhost:8000/users/1
```

### ステップ5: データベースを確認

#### 方法1: コンテナ内から直接

```bash
# PostgreSQLコンテナに入る
docker exec -it postgres-db psql -U myuser -d mydb

# SQLコマンド実行
SELECT * FROM users;

# 終了
\q
```

#### 方法2: pgAdminを使用

```
http://localhost:5050
```

- Email: admin@example.com
- Password: admin

**サーバー接続設定：**
- Host: db
- Port: 5432
- Username: myuser
- Password: mypassword

---

## よく使うコマンド一覧

### 基本コマンド

| コマンド | 説明 | 例 |
|---------|------|-----|
| `docker pull` | イメージをダウンロード | `docker pull ubuntu` |
| `docker images` | ローカルのイメージ一覧 | `docker images` |
| `docker run` | コンテナを起動 | `docker run -d nginx` |
| `docker ps` | 実行中のコンテナ一覧 | `docker ps` |
| `docker ps -a` | 全てのコンテナ一覧 | `docker ps -a` |
| `docker stop` | コンテナを停止 | `docker stop コンテナ名` |
| `docker start` | コンテナを起動 | `docker start コンテナ名` |
| `docker restart` | コンテナを再起動 | `docker restart コンテナ名` |
| `docker rm` | コンテナを削除 | `docker rm コンテナ名` |
| `docker rmi` | イメージを削除 | `docker rmi イメージ名` |
| `docker exec` | 実行中のコンテナでコマンド実行 | `docker exec -it コンテナ名 bash` |
| `docker logs` | コンテナのログ表示 | `docker logs コンテナ名` |
| `docker logs -f` | ログをリアルタイムで追跡 | `docker logs -f コンテナ名` |
| `docker build` | Dockerfileからイメージ作成 | `docker build -t 名前:タグ .` |

### Docker Composeコマンド

| コマンド | 説明 | 例 |
|---------|------|-----|
| `docker-compose up` | コンテナを起動 | `docker-compose up -d` |
| `docker-compose down` | コンテナを停止・削除 | `docker-compose down` |
| `docker-compose down -v` | コンテナ、ネットワーク、ボリュームを削除 | `docker-compose down -v` |
| `docker-compose ps` | コンテナの状態を確認 | `docker-compose ps` |
| `docker-compose logs` | ログを確認 | `docker-compose logs -f web` |
| `docker-compose build` | イメージを再ビルド | `docker-compose build` |
| `docker-compose up --build` | 再ビルドして起動 | `docker-compose up -d --build` |
| `docker-compose restart` | サービスを再起動 | `docker-compose restart web` |
| `docker-compose exec` | コンテナでコマンド実行 | `docker-compose exec web bash` |

### よく使うオプション

#### docker run のオプション

```bash
docker run [オプション] イメージ名

# 主なオプション
-d                    # バックグラウンドで実行
-p ホスト:コンテナ      # ポートマッピング
--name 名前           # コンテナに名前をつける
-v ホスト:コンテナ      # ボリュームマウント
-e KEY=VALUE         # 環境変数を設定
--rm                 # 停止時に自動削除
-it                  # インタラクティブモード（シェル操作用）
--network ネットワーク名  # ネットワークを指定
```

#### 例

```bash
# 複数オプションを組み合わせた例
docker run -d \
  --name my-app \
  -p 8080:80 \
  -v $(pwd):/app \
  -e ENV=production \
  nginx:alpine
```

---

## トラブルシューティング

### よくある問題と解決方法

#### 1. ポートがすでに使用されている

**エラー:**
```
Error response from daemon: driver failed programming external connectivity on endpoint
```

**解決方法:**
```bash
# 使用中のポートを確認
lsof -i :8080

# または別のポートを使用
docker run -p 8081:80 nginx
```

#### 2. データベース接続エラー

**原因:** データベースの起動が遅い

**解決方法:**
```bash
# データベースのログを確認
docker-compose logs db

# webコンテナを再起動
docker-compose restart web
```

#### 3. コンテナが起動しない

**確認手順:**
```bash
# コンテナのステータスを確認
docker ps -a

# ログを確認
docker logs コンテナ名

# 詳細なエラー情報を確認
docker inspect コンテナ名
```

#### 4. イメージのビルドが失敗する

**確認手順:**
```bash
# キャッシュを使わずに再ビルド
docker build --no-cache -t イメージ名 .

# docker-composeの場合
docker-compose build --no-cache
```

#### 5. ディスク容量不足

**解決方法:**
```bash
# 未使用のコンテナを削除
docker container prune

# 未使用のイメージを削除
docker image prune

# 未使用のボリュームを削除
docker volume prune

# 全てをまとめて削除
docker system prune -a
```

#### 6. コンテナ内のファイルが更新されない

**原因:** ボリュームマウントの問題

**解決方法:**
```bash
# コンテナを再起動
docker-compose restart

# または再ビルド
docker-compose up -d --build
```

---

## ベストプラクティス

### 1. .dockerignore を使う

**.dockerignore**
```
node_modules
__pycache__
*.pyc
.git
.env
*.log
```

### 2. マルチステージビルドを使う

軽量なイメージを作成：

```dockerfile
# ビルドステージ
FROM python:3.11 AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --user -r requirements.txt

# 実行ステージ
FROM python:3.11-slim
WORKDIR /app
COPY --from=builder /root/.local /root/.local
COPY . .
ENV PATH=/root/.local/bin:$PATH
CMD ["python", "app.py"]
```

### 3. 環境変数を使う

機密情報はコードに直接書かない：

```yaml
# docker-compose.yml
services:
  web:
    env_file:
      - .env
    environment:
      - DEBUG=${DEBUG}
```

### 4. ヘルスチェックを設定する

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s \
  CMD curl -f http://localhost:8000/health || exit 1
```

### 5. ログをローテーションする

```yaml
services:
  web:
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
```

---

## 参考リンク

- [Docker公式ドキュメント](https://docs.docker.com/)
- [Docker Hub](https://hub.docker.com/)
- [FastAPI公式ドキュメント](https://fastapi.tiangolo.com/)
- [PostgreSQL公式ドキュメント](https://www.postgresql.org/docs/)

---

## まとめ

### Dockerの基本フロー

1. **イメージを取得または作成**
   - `docker pull` または `docker build`

2. **コンテナを起動**
   - `docker run` または `docker-compose up`

3. **動作確認**
   - ブラウザ、curl、ログなどで確認

4. **停止・削除**
   - `docker stop` / `docker-compose down`

### 開発時のワークフロー

1. Dockerfileとdocker-compose.ymlを作成
2. `docker-compose up -d` で起動
3. コードを編集（自動リロード）
4. `docker-compose logs -f` でログ確認
5. 必要に応じて `docker-compose restart`

この資料を使って、Dockerの基礎から実践的な使い方まで学習できます。わからないことがあれば、各セクションを参照してください！
