---
title: "Lightsailを用いて Dify 0.6.14 をデプロイする（HTTPS化も）"
emoji: "⛵"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["lightsail", "dify", "https"]
published: false
---

# 1. Amazon Lightsailでデプロイする

## 1.1. Lightsailインスタンスを作成する

Amazon Lightsailにて画像のようにインスタンスを作成します。

設計図はUbuntu 22.04 LTSを選択してください。他の設計図だと本手順通りに進めません。

![](/images/dcc1df0bacf1d8/1.png)

また、サイズは2GBを選択してください。1GBだとメモリ不足で起動しないことがあります。

![](/images/dcc1df0bacf1d8/2.png)

## 1.2. スクリプトを用いてDifyをインストールする

SSHを使用してインスタンスに接続します。

![](/images/dcc1df0bacf1d8/3.png)

以下のコマンドを入力してください。

```bash
curl -fsSL <https://gist.githubusercontent.com/gijigae/50f400a5f91a39fbf2f0d0a652a9c409/raw/install-dify.sh> -o install-dify.sh

sudo sh install-dify.sh
```

5分程度待つとDifyが起動します。

## 1.3. Difyの起動を確認する

インスタンス画面でパブリックIPアドレスを確認し、ブラウザで以下のURLにアクセスします。

`http://<パブリックIP>/install`

Difyの画面が表示されれば、起動に成功しています。プロトコルが `https` ではなく、 `http` であることに注意してください。

# 2. HTTPS化する

## 2.1 独自ドメインを取得する

「ドメインとDNS」タブの「ドメインの登録」をクリックし、取得したいドメインを入力してください。
![](/images/dcc1df0bacf1d8/4.png)

自動更新の設定や連絡先情報を入力し、ドメインを登録します。

## 2.2. 静的IPを作る

作成したインスタンスの「ネットワーキング」タブをクリックし、「静的IPをアタッチする」をクリックします。

静的IPを入力し、「作成およびアタッチ」をクリックしてください。

## 2.3. HTTPSのポートを許可する

「ルールを追加」をクリックし、HTTPSのルールを作成してください。

## 2.4. 独自ドメインを静的IPに割りあてる

「ドメインとDNS」タブの「DNSゾーンの作成」をクリックし、独自ドメインを選択します。

「DNSゾーンの作成」→「割り当て」→作成したドメイン名を選択→「割り当てる」をクリックしてください。


## 2.5. SSL証明書を発行する

「インスタンス」タブの「SSHを使用して接続」をクリックしてください。

[Let's Encrypt 証明書のインストールのために、 Certbot パッケージを Lightsail インスタンスにインストールするにはどうすればよいですか？](https://repost.aws/ja/knowledge-center/lightsail-install-certbot-package) を参考にcerbotをインストールします。

今回（Ubuntu 22.04 LTS）の場合は以下のコマンドです。

```bash
sudo snap install core;
sudo snap refresh core;
sudo snap install --classic certbot
```

[スタンダード Let's Encrypt SSL 証明書を Lightsail インスタンスにインストールするにはどうすればよいですか?](https://repost.aws/ja/knowledge-center/lightsail-standard-ssl-certificate) を参考にSSL証明書を取得する

今回の場合は以下のコマンドです。

```bash
sudo service apache2 stop
sudo certbot certonly --standalone -d <独自ドメイン>
```

表示された証明書とキーのパスは以下の手順で使うため、メモしておいてください。

証明書とキーをシンボリックリンクで参照させます。

```bash
sudo ln -s /etc/letsencrypt/live/<独自ドメイン>/fullchain.pem ~/dify/docker/nginx/ssl/fullchain.pem
sudo ln -s /etc/letsencrypt/live/<独自ドメイン>/privkey.pem ~/dify/docker/nginx/ssl/privkey.pem
```


## 2.6. .envファイルを作成する

~/dify/docker/.env.example をコピーして.envファイルを作成します。

```bash
cp ~/dify/docker/.env.example ~/dify/docker/.env
```

.envファイルを編集するため、vimを起動します。

```bash
sudo vim .env
```

577行目あたり（Dify 0.6.14の場合）を以下の内容に編集します。

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

保存して終了します。

## 2.7. Dockerを再起動

Dcokerを再起動し、HTTPS化に成功していることを確認します。

```bash
sudo docker compose down
sudo docker compose up
```

`https://<独自ドメイン>/install` にアクセスできれば、HTTPS化成功です。

