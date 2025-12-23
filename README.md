# Traefik + Crowdsec セットアップ

Traefik リバースプロキシと Crowdsec WAF を組み合わせたセキュアなDocker環境。

## 機能

- **Traefik v2.11**: リバースプロキシ、自動SSL証明書（Let's Encrypt）
- **Crowdsec**: WAF（Web Application Firewall）、不正アクセス検知・ブロック
- **HTTP→HTTPS自動リダイレクト**
- **gzip圧縮対応**

## セットアップ

### 1. ネットワーク作成

```bash
docker network create traefik-network
```

### 2. 設定ファイルの準備

```bash
# exampleファイルをコピー
cp docker-compose.example.yml docker-compose.yml
cp traefik.example.yml traefik.yml
cp .env.example .env

# traefik.ymlのメールアドレスを編集
vim traefik.yml
```

### 3. Crowdsec Bouncerプラグインのインストール

```bash
mkdir -p plugins-local/src/github.com/maxlerebourg/crowdsec-bouncer-traefik-plugin
cd plugins-local/src/github.com/maxlerebourg/crowdsec-bouncer-traefik-plugin
git clone https://github.com/maxlerebourg/crowdsec-bouncer-traefik-plugin.git .
```

### 4. 必要なディレクトリ作成

```bash
mkdir -p letsencrypt logs
```

### 5. 起動

```bash
docker compose up -d
```

### 6. Crowdsec Bouncer APIキーの取得

```bash
# APIキーを生成
docker exec crowdsec cscli bouncers add traefik-bouncer

# 表示されたキーを.envに設定
echo "CROWDSEC_BOUNCER_KEY=生成されたキー" > .env

# Traefikを再起動
docker compose restart traefik
```

## ファイル構成

```
traefik/
├── docker-compose.yml      # 本番設定（.gitignore）
├── docker-compose.example.yml
├── traefik.yml             # 本番設定（.gitignore）
├── traefik.example.yml
├── .env                    # 認証情報（.gitignore）
├── .env.example
├── crowdsec/
│   ├── acquis.yaml         # ログ取得設定
│   └── whitelist.yaml      # ホワイトリスト
├── letsencrypt/            # SSL証明書（.gitignore）
├── logs/                   # ログ（.gitignore）
└── plugins-local/          # プラグイン（.gitignore）
```

## アプリケーションの追加

各アプリケーションの `docker-compose.yml` に以下のラベルを追加：

```yaml
services:
  your-app:
    labels:
      traefik.enable: true
      traefik.http.routers.your-app.rule: Host(`your-domain.com`)
      traefik.http.routers.your-app.entrypoints: websecure
      traefik.http.routers.your-app.tls.certresolver: myresolver
      traefik.http.routers.your-app.middlewares: gzip-compress,crowdsec
      traefik.http.middlewares.gzip-compress.compress: true
    networks:
      - traefik-network

networks:
  traefik-network:
    external: true
```

## Crowdsec 管理コマンド

```bash
# ブロックされたIP一覧
docker exec crowdsec cscli decisions list

# 特定IPのBAN解除
docker exec crowdsec cscli decisions delete --ip x.x.x.x

# アラート一覧
docker exec crowdsec cscli alerts list

# メトリクス確認
docker exec crowdsec cscli metrics
```

## ホワイトリストの編集

信頼するIPを追加する場合は `crowdsec/whitelist.yaml` を編集：

```yaml
name: custom/whitelist
description: "Whitelist trusted IPs"
whitelist:
  reason: "trusted IPs"
  ip:
    - "x.x.x.x"  # 追加するIP
```

編集後、Crowdsecを再起動：

```bash
docker compose restart crowdsec
```

## トラブルシューティング

### APIが403を返す場合

Crowdsecによるブロックの可能性があります：

```bash
# ブロック一覧を確認
docker exec crowdsec cscli decisions list

# 該当IPを解除
docker exec crowdsec cscli decisions delete --ip x.x.x.x
```

### SSL証明書の問題

```bash
# acme.jsonの権限を確認
chmod 600 letsencrypt/acme.json

# Traefikログを確認
docker logs traefik
```

## 参考

- [Traefik Documentation](https://doc.traefik.io/traefik/)
- [Crowdsec Documentation](https://docs.crowdsec.net/)
- [Crowdsec Traefik Bouncer Plugin](https://github.com/maxlerebourg/crowdsec-bouncer-traefik-plugin)
