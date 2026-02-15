# MinIO Object Storage on Docker - セットアップ手順書

## 概要

Proxmox上のUbuntu 24.04 VMにDocker経由でMinIO（S3互換オブジェクトストレージ）を構築する手順。
Azure Blob Storage のローカル代替として使用する。

## 前提条件

| 項目 | 値 |
|------|------|
| ハイパーバイザー | Proxmox VE |
| ゲストOS | Ubuntu 24.04.4 LTS |
| IPアドレス | 192.168.10.50 |
| Docker | インストール済み |

## 構成図

```
[Proxmox VE]
  └── [Ubuntu 24.04 VM - 192.168.10.50]
        └── Docker Engine
              ├── minio コンテナ (port 9000: S3 API, port 9001: Web Console)
              │     └── /data → named volume or bind mount
              └── minio-init コンテナ (起動時にバケット作成後に終了)
```

## ファイル構成

```
minIO/
├── docker-compose.internal.yml   # named volume 版（VM内部ストレージ）
├── docker-compose.external.yml   # bind mount 版（外部ストレージ）
├── .env                          # 環境変数（認証情報・バケット名・データパス）
├── init-bucket.sh                # バケット自動作成スクリプト
└── README.md                     # 本ドキュメント
```

## ストレージ構成の使い分け

用途に応じて2つの docker-compose ファイルを使い分ける。

| 構成 | ファイル | ストレージ | 用途 |
|------|----------|-----------|------|
| internal | `docker-compose.internal.yml` | Docker named volume (`minio-data`) | VM内完結、テスト・軽量用途 |
| external | `docker-compose.external.yml` | bind mount (`MINIO_DATA_PATH`) | 外部ディスク・NFSマウント先 |

### internal 版で起動

```bash
cd ~/minIO
docker compose -f docker-compose.internal.yml up -d
```

### external 版で起動

external 版を使う場合は、事前に `.env` の `MINIO_DATA_PATH` を確認・変更し、マウント先ディレクトリを準備する。

```bash
# 1. マウント先の準備（例: Proxmox から追加した外部ディスク）
sudo mkdir -p /mnt/minio-data
sudo chown 1000:1000 /mnt/minio-data

# 2. .env の MINIO_DATA_PATH を確認（デフォルト: /mnt/minio-data）

# 3. 起動
cd ~/minIO
docker compose -f docker-compose.external.yml up -d
```

### 構成の切り替え

一方の構成から他方に切り替える場合は、先に現在の構成を停止してから起動する。

```bash
# 例: internal → external に切り替え
docker compose -f docker-compose.internal.yml down
docker compose -f docker-compose.external.yml up -d
```

## 環境変数 (.env)

| 変数名 | 説明 | デフォルト値 | 使用構成 |
|--------|------|-------------|---------|
| `MINIO_ROOT_USER` | 管理者ユーザー名（S3のAccess Keyに相当） | `minioadmin` | 両方 |
| `MINIO_ROOT_PASSWORD` | 管理者パスワード（S3のSecret Keyに相当） | `minioadmin123` | 両方 |
| `MINIO_BUCKET_NAME` | 起動時に自動作成するバケット名 | `video-cache` | 両方 |
| `MINIO_DATA_PATH` | bind mount 先パス | `/mnt/minio-data` | external のみ |

```env
MINIO_ROOT_USER=minioadmin
MINIO_ROOT_PASSWORD=minioadmin123
MINIO_BUCKET_NAME=video-cache
MINIO_DATA_PATH=/mnt/minio-data
```

## 手順1: 起動

```bash
cd ~/minIO

# 起動（初回はイメージダウンロードが発生）
# internal 版:
docker compose -f docker-compose.internal.yml up -d
# external 版:
docker compose -f docker-compose.external.yml up -d
```

初回起動時の動作:
1. `minio` コンテナが起動し、healthcheck で準備完了を待機
2. `minio-init` コンテナが `.env` の `MINIO_BUCKET_NAME` で指定されたバケットを作成
3. `minio-init` は作成完了後に自動終了（正常終了: Exit 0）

## 手順2: 動作確認

以下の例では `docker-compose.internal.yml` を使用。external 版の場合はファイル名を読み替える。

```bash
# コンテナ状態確認（使用中のファイルを指定）
docker compose -f docker-compose.internal.yml ps -a

# 期待される結果:
#   minio       ... Up (healthy)
#   minio-init  ... Exited (0)

# init コンテナのログ確認（バケット作成結果）
docker compose -f docker-compose.internal.yml logs minio-init
```

## Web Console接続設定

| 項目 | 値 |
|------|------|
| URL | `http://192.168.10.50:9001` |
| ユーザー名 | `.env` の `MINIO_ROOT_USER` を参照 |
| パスワード | `.env` の `MINIO_ROOT_PASSWORD` を参照 |

ブラウザでアクセスし、バケット一覧に `MINIO_BUCKET_NAME` で指定した名前が表示されることを確認する。

## S3 API接続設定

| 項目 | 値 |
|------|------|
| エンドポイント | `http://192.168.10.50:9000` |
| Access Key | `.env` の `MINIO_ROOT_USER` |
| Secret Key | `.env` の `MINIO_ROOT_PASSWORD` |
| リージョン | 不要（空文字でOK） |

### Python (boto3) からの接続例

```python
import boto3

s3 = boto3.client("s3",
    endpoint_url="http://192.168.10.50:9000",
    aws_access_key_id="minioadmin",
    aws_secret_access_key="minioadmin123",
)

# バケット一覧
print(s3.list_buckets())

# オブジェクト一覧
response = s3.list_objects_v2(Bucket="video-cache")
for obj in response.get("Contents", []):
    print(obj["Key"], obj["Size"])

# アップロード
s3.upload_file("/path/to/file.mp4", "video-cache", "file.mp4")

# ダウンロード
s3.download_file("video-cache", "file.mp4", "/path/to/download.mp4")
```

## MinIO Client (mc) の使い方

`mc` は MinIO 公式のコマンドラインクライアント。ファイルのアップロード・ダウンロード・同期に使用する。

### インストール (macOS)

```bash
brew install minio/stable/mc
```

### エイリアス設定

```bash
mc alias set myminio http://192.168.10.50:9000 minioadmin minioadmin123
```

### 基本操作

```bash
# バケット一覧
mc ls myminio/

# バケット内のオブジェクト一覧
mc ls myminio/video-cache/

# オブジェクト詳細情報
mc stat myminio/video-cache/example.mp4

# ファイルアップロード
mc cp /path/to/file.mp4 myminio/video-cache/

# ファイルダウンロード
mc cp myminio/video-cache/file.mp4 /path/to/download/

# ディレクトリ一括アップロード（再帰的）
mc cp --recursive /path/to/videos/ myminio/video-cache/

# ディレクトリ同期（差分のみ転送・冪等）
mc mirror /path/to/videos/ myminio/video-cache/

# オブジェクト削除
mc rm myminio/video-cache/file.mp4

# バケット内の全オブジェクト削除
mc rm --recursive --force myminio/video-cache/

# バケットの使用量確認
mc du myminio/video-cache/
```

## MinIO設定値

| 項目 | 値 | 備考 |
|------|------|------|
| イメージ | `minio/minio:latest` | |
| ポート (S3 API) | 9000 | アプリケーション・mc からの接続用 |
| ポート (Web Console) | 9001 | ブラウザ管理画面 |
| メモリ上限 | 4GB | deploy.resources.limits |
| メモリ最低保証 | 1GB | deploy.resources.reservations |
| タイムゾーン | Asia/Tokyo | |
| 再起動ポリシー | unless-stopped | 手動停止以外は自動再起動 |
| init用イメージ | `minio/mc:latest` | バケット作成用 (minio-init) |
| init再起動ポリシー | no | 作成完了後に終了、再起動しない |

## ボリューム構成

| 構成 | ホスト側 | コンテナ内パス | 用途 |
|------|---------|---------------|------|
| internal | named volume `minio-data`（実パス: `/var/lib/docker/volumes/minio_minio-data/_data/`） | /data | バケットデータ（VM内部） |
| external | `MINIO_DATA_PATH`（bind mount） | /data | バケットデータ（外部ストレージ） |

## 運用コマンド

以下の例では `docker-compose.internal.yml` を使用。external 版の場合はファイル名を読み替える。

```bash
cd ~/minIO

# 状態確認
docker compose -f docker-compose.internal.yml ps

# ログ確認（リアルタイム）
docker compose -f docker-compose.internal.yml logs -f minio

# 停止
docker compose -f docker-compose.internal.yml stop

# 起動
docker compose -f docker-compose.internal.yml start

# 再作成（設定変更後）
docker compose -f docker-compose.internal.yml down && docker compose -f docker-compose.internal.yml up -d

# データごと完全削除（注意: バケット内の全データも消える）
# internal 版のみ -v で named volume を削除可能
docker compose -f docker-compose.internal.yml down -v
```

## バケットの追加

`.env` の `MINIO_BUCKET_NAME` は初回起動用。追加のバケットが必要な場合:

```bash
# Web Console (http://192.168.10.50:9001) から作成
# または mc コマンドで作成:
mc mb myminio/new-bucket-name
```

## トラブルシューティング

### minio-init が Exited (1) で終了する
init コンテナのログを確認:
```bash
docker compose -f docker-compose.internal.yml logs minio-init
```
よくある原因:
- MinIO が healthcheck を通過する前に init が実行された → `docker compose -f <使用中のファイル> down && docker compose -f <使用中のファイル> up -d` で再起動
- `.env` の認証情報が不正

### Web Console にアクセスできない
1. コンテナが起動しているか確認: `docker compose -f docker-compose.internal.yml ps`
2. ポートが開いているか確認: `nc -z 192.168.10.50 9001`
3. VMのファイアウォール確認: `sudo ufw status`

### mc で接続できない
1. エイリアスの設定を確認: `mc alias list`
2. エンドポイントのポートが 9000 (API) であることを確認（9001 は Web Console）
3. 認証情報が `.env` の値と一致しているか確認

### external 版で minio が起動しない
よくある原因:
- `MINIO_DATA_PATH` のディレクトリが存在しない → `sudo mkdir -p /mnt/minio-data`
- パーミッション不足 → `sudo chown 1000:1000 /mnt/minio-data`
- `.env` の `MINIO_DATA_PATH` が未設定
