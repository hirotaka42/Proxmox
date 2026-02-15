# SQL Server 2022 on Docker - セットアップ手順書

## 概要

Proxmox上のUbuntu 24.04 VMにDocker経由でSQL Server 2022 Developerを構築する手順。
再現性を担保するため、ゼロからの構築手順を記載する。

## 前提条件

| 項目 | 値 |
|------|------|
| ハイパーバイザー | Proxmox VE |
| ゲストOS | Ubuntu 24.04.4 LTS |
| CPU | 6コア以上 |
| メモリ | 16GB以上（SQL Serverに8GB割当） |
| ディスク1 (sda) | 32GB - OS用 |
| ディスク2 (sdb) | 100GB - Docker/データ用 |
| IPアドレス | 192.168.10.117 |

## 構成図

```
[Proxmox VE]
  └── [Ubuntu 24.04 VM - 192.168.10.117]
        ├── /dev/sda (32GB) → / (OS, LVM)
        ├── /dev/sdb (100GB) → /mnt/data (ext4)
        │     └── /mnt/data/docker/ (Dockerデータルート)
        │           └── volumes/
        │                 ├── sqlserver_sqldata  → /var/opt/mssql/data
        │                 ├── sqlserver_sqllog   → /var/opt/mssql/log
        │                 └── sqlserver_sqlbackup→ /var/opt/mssql/backup
        └── Docker Engine
              └── sqlserver コンテナ (port 1433)
```

## 手順1: データディスクのセットアップ

Proxmox管理画面でVMに100GBのディスク(sdb)を追加した後、VM内で実行:

```bash
# ディスクをext4でフォーマット
sudo mkfs.ext4 -F /dev/sdb

# マウントポイント作成・マウント
sudo mkdir -p /mnt/data
sudo mount /dev/sdb /mnt/data

# 永続化 (fstab)
UUID=$(sudo blkid -s UUID -o value /dev/sdb)
echo "UUID=$UUID /mnt/data ext4 defaults 0 2" | sudo tee -a /etc/fstab
```

## 手順2: Dockerインストール

```bash
# 前提パッケージ
sudo apt-get update
sudo apt-get install -y ca-certificates curl

# Docker公式GPGキー
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# リポジトリ追加
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Docker Engineインストール
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# ユーザーをdockerグループに追加
sudo usermod -aG docker $USER

# Dockerのデータルートを /mnt/data に変更
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<EOF
{
  "data-root": "/mnt/data/docker"
}
EOF

# Docker再起動・自動起動有効化
sudo systemctl restart docker
sudo systemctl enable docker
```

## 手順3: SQL Server起動

```bash
# 作業ディレクトリ作成
mkdir -p ~/sqlserver
cd ~/sqlserver

# docker-compose.yml と .env を配置（本リポジトリのファイルを使用）
# その後起動:
docker compose up -d
```

## 手順4: 動作確認

```bash
# コンテナ状態確認
docker compose ps

# SQL Server内部から接続テスト
docker exec sqlserver /opt/mssql-tools18/bin/sqlcmd \
  -S localhost -U sa -P "$(grep SA_PASSWORD .env | cut -d= -f2)" \
  -Q "SELECT @@VERSION" -C -b
```

## SSMS接続設定

| 項目 | 値 |
|------|------|
| サーバーの種類 | データベースエンジン |
| サーバー名 | `192.168.10.117,1433` |
| 認証 | SQL Server 認証 |
| ログイン | `sa` |
| パスワード | `.env` ファイルの `SA_PASSWORD` を参照 |
| 暗号化 | Optional（自己署名証明書のため） |

## SQL Server設定値

| 項目 | 値 | 備考 |
|------|------|------|
| バージョン | SQL Server 2022 | `mcr.microsoft.com/mssql/server:2022-latest` |
| エディション | Developer | Enterprise全機能、開発/テスト用 |
| 照合順序 | Japanese_CI_AS | 日本語対応、大文字小文字区別なし |
| メモリ上限 | 8192MB | MSSQL_MEMORY_LIMIT_MB |
| コンテナメモリ | 10GB上限 / 4GB最低保証 | deploy.resources |
| タイムゾーン | Asia/Tokyo | |
| 実行ユーザー | root (0:0) | ボリューム権限問題の回避のため |
| ポート | 1433 (ホスト) → 1433 (コンテナ) | |

## ボリューム構成

| ボリューム名 | コンテナ内パス | 用途 |
|-------------|---------------|------|
| sqlserver_sqldata | /var/opt/mssql/data | データベースファイル (.mdf, .ndf) |
| sqlserver_sqllog | /var/opt/mssql/log | トランザクションログ (.ldf) |
| sqlserver_sqlbackup | /var/opt/mssql/backup | バックアップファイル (.bak) |

## 運用コマンド

```bash
cd ~/sqlserver

# 状態確認
docker compose ps

# ログ確認（リアルタイム）
docker compose logs -f

# 停止
docker compose stop

# 起動
docker compose start

# 再作成（設定変更後）
docker compose down && docker compose up -d

# データごと完全削除（注意: データも消える）
docker compose down -v

# バックアップ例（コンテナ内からT-SQL実行）
docker exec sqlserver /opt/mssql-tools18/bin/sqlcmd \
  -S localhost -U sa -P "$(grep SA_PASSWORD .env | cut -d= -f2)" \
  -Q "BACKUP DATABASE [MyDB] TO DISK = '/var/opt/mssql/backup/MyDB.bak'" -C -b
```

## トラブルシューティング

### コンテナが起動しない (Access is denied)
SQL Server 2022はデフォルトで非rootユーザー(mssql)で動作する。
ボリュームの権限問題が発生する場合は `user: "0:0"` を docker-compose.yml に追加する。

### SSMSで接続できない
1. VMのファイアウォール(ufw)が無効であることを確認: `sudo ufw status`
2. ポート1433が開いていることを確認: `nc -z 192.168.10.117 1433`
3. コンテナが起動しているか確認: `docker compose ps`
4. SSMS側の「暗号化」設定を「Optional」にする

### メモリ不足
`MSSQL_MEMORY_LIMIT_MB` を調整する。VMの全メモリをSQL Serverに割り当てないこと。
