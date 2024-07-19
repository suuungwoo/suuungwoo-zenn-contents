---
title: "Lightsailを用いて Dify 0.6.14 をデプロイする（HTTPS化も）"
emoji: "🗂"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["lightsail"]
published: false
---

# 1. Amazon Lightsailでデプロイする

## 1.1. Lightsailインスタンスを作成する

設計図はUbuntu 22.04 LTS

サイズは2GBメモリ

## 1.2. スクリプトを用いてDifyをインストールする

SSHを使用してインスタンスに接続する。

以下のスクリプトを貼り付ける。

```bash
curl -fsSL <https://gist.githubusercontent.com/gijigae/50f400a5f91a39fbf2f0d0a652a9c409/raw/install-dify.sh> -o install-dify.sh

sudo sh install-dify.sh
```

5分程度待つ。

## 1.3. Difyの起動を確認する

インスタンス画面でパブリックIPアドレスを確認し、ブラウザで以下のURLにアクセスする。

`http://<パブリックIP>/install`

プロトコルはhttpであることに注意。

# 2. HTTPS化する

## 2.1 独自ドメインを取得する

1. 「ドメインとDNS」をクリックする
2. 「ドメインの登録」をクリックする
3. 取得したいドメインを入力する
4. 自動更新の設定、連絡先情報を入力する
5. 「ドメインの登録」をクリックする

## 2.2. 静的IPを作る

1. 「インスタンス」をクリックする
2. 作成したインスタンスをクリックする
3. 「ネットワーキング」タブをクリックする
4. 「静的IPをアタッチする」をクリックする
5. 静的IPを入力し、「作成およびアタッチ」をクリックする

## 2.3. HTTPSのポートを許可する

1. 「ルールを追加」をクリックする
2. HTTPSのルールを作成する

## 2.4. 独自ドメインを静的IPに割りあてる

1. 「ドメインとDNS」をクリックする
2. 「DNSゾーンの作成」をクリックする
3. 独自ドメインを選択する
4. 「DNSゾーンの作成」をクリックする
5. 「割り当て」タブをクリックする
6. 作成したドメイン名を選択する
7. 「割り当てる」をクリックする

## 2.5. SSL証明書を発行する

1. 「インスタンス」をクリックする
2. 「SSHを使用して接続」をクリックする
3. [Let's Encrypt 証明書のインストールのために、 Certbot パッケージを Lightsail インスタンスにインストールするにはどうすればよいですか？](https://repost.aws/ja/knowledge-center/lightsail-install-certbot-package) を参考にcerbotをインストールする
    
    今回（Ubuntu 22.04 LTS）の場合
    
    ```bash
    sudo snap install core;
    sudo snap refresh core;
    sudo snap install --classic certbot
    ```
    
4. [スタンダード Let's Encrypt SSL 証明書を Lightsail インスタンスにインストールするにはどうすればよいですか?](https://repost.aws/ja/knowledge-center/lightsail-standard-ssl-certificate) を参考にSSL証明書を取得する
    
    今回の場合
    
    ```bash
    sudo service apache2 stop
    sudo certbot certonly --standalone -d <独自ドメイン>
    ```
    
    結果
    
    証明書とキーのパスは下の手順で使うためメモしておく。
    
5. 証明書とキーをシンボリックリンクで参照する
    
    ```bash
    sudo ln -s /etc/letsencrypt/live/<独自ドメイン>/fullchain.pem ~/dify/docker/nginx/ssl/fullchain.pem
    sudo ln -s /etc/letsencrypt/live/<独自ドメイン>/privkey.pem ~/dify/docker/nginx/ssl/privkey.pem
    ```
    

## 2.6. .envファイルを作成する

~/dify/docker/.env.example をコピーして.envファイルを作成する。

```bash
cp ~/dify/docker/.env.example ~/dify/docker/.env
```

.envファイルを編集する

```bash
sudo vim .env
```

577行目あたりを以下の内容に編集する。

```bash
# ------------------------------
# Environment Variables for Nginx reverse proxy
# ------------------------------
NGINX_SERVER_NAME=<独自ドメイン>
NGINX_HTTPS_ENABLED=true  # ←trueに設定
# HTTP port
NGINX_PORT=80
# SSL settings are only applied when HTTPS_ENABLED is true
NGINX_SSL_PORT=443
# If HTTPS_ENABLED is true, you're required to add your own SSL certificates/keys to the `/nginx/ssl` directory
# and modify the env vars below accordingly.
NGINX_SSL_CERT_FILENAME=fullchain.pem  # ←先ほどコピーした証明書ファイル名
NGINX_SSL_CERT_KEY_FILENAME=privkey.pem  # ←先ほどコピーした暗号キーファイル名
NGINX_SSL_PROTOCOLS=TLSv1.1 TLSv1.2 TLSv1.3
```

保存して終了する。

## 2.7. Dockerを再起動

```bash
sudo docker compose down
sudo docker compose up
```

`https://<独自ドメイン>/install` にアクセスし、HTTPSでアクセスできることを確認する。
