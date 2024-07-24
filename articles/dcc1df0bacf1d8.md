---
title: "保存版！Amazon Lightsailで Dify 0.6.14 をHTTPSデプロイ"
emoji: "⛵"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["lightsail", "dify", "https"]
published: false
---

# 1. はじめに

Difyは、オープンソースのAIアプリケーション開発プラットフォームです。誰でもノーコードで、チャットボット・AIアシスタントや要約・分析ツール、画像生成アプリ、計算ツールなど、様々なAIアプリケーションを作成することができます。近年、AI技術の進歩とニーズの高まりにより、AIアプリケーションの開発が注目されています。Difyは、専門知識がなくても、手軽にAIアプリケーションを開発できるツールとして人気を集めています。

そんなDifyをインターネット上で公開するには、セキュリティ対策が必須です。しかし、Difyのバージョン0.6.14以降では少し仕様が変わっており、HTTPSデプロイの手順がネット上で見当たらなかったので、まとめてみました。

本記事では、Lightsailインスタンスのセットアップから、独自ドメインの取得、SSL証明書の発行まで、手順を詳しく解説します。

# 2. Amazon Lightsailでデプロイする

Lightsailは、Difyをデプロイし、安全かつ快適に運用するための最適なプラットフォームです。簡単で迅速な導入、拡張性の高いインフラ、高い信頼性と可用性、グローバルな展開、手頃な価格など、多くのメリットを提供します。

## 2.1. Lightsailインスタンスを作成する

Amazon Lightsailにて画像のようにインスタンスを作成します。

設計図はUbuntu 22.04 LTSを選択してください。他の設計図だと本記事の手順通りに進めません。

![](/images/dcc1df0bacf1d8/1.png)

また、サイズは2GBを選択してください。1GBだとメモリ不足で起動しないことがあります。

![](/images/dcc1df0bacf1d8/2.png)

## 2.2. スクリプトを用いてDifyをインストールする

SSHを使用してインスタンスに接続します。

![](/images/dcc1df0bacf1d8/3.png)

以下のコマンドを入力してください。

```bash
curl -fsSL <https://gist.githubusercontent.com/gijigae/50f400a5f91a39fbf2f0d0a652a9c409/raw/install-dify.sh> -o install-dify.sh

sudo sh install-dify.sh
```

5分程度待つとDifyが起動します。

## 2.3. Difyの起動を確認する

インスタンス画面でパブリックIPアドレスを確認し、ブラウザで以下のURLにアクセスします。

`http://<パブリックIP>/install`

Difyの画面が表示されれば、起動に成功しています。プロトコルが `https` ではなく、 `http` であることに注意してください。

# 3. HTTPS化する

DifyアプリをHTTPS化するメリットは、セキュリティ強化、信頼性の向上、SEO対策等が挙げられます。Difyアプリの安全性を高め、ユーザーの信頼を獲得し、より多くの人に利用してもらうためには、HTTPS化が不可欠です。

## 3.1. 独自ドメインを取得する

「ドメインとDNS」タブの「ドメインの登録」をクリックし、取得したいドメインを入力してください。
![](/images/dcc1df0bacf1d8/4.png)

自動更新の設定や連絡先情報を入力し、ドメインを登録します。

## 3.2. 静的IPを作る

作成したインスタンスの「ネットワーキング」タブをクリックし、「静的IPをアタッチする」をクリックします。

静的IPを入力し、「作成およびアタッチ」をクリックしてください。

## 3.3. HTTPSのポートを許可する

「ルールを追加」をクリックし、HTTPSのルールを作成してください。

## 3.4. 独自ドメインを静的IPに割りあてる

「ドメインとDNS」タブの「DNSゾーンの作成」をクリックし、独自ドメインを選択します。

「DNSゾーンの作成」→「割り当て」→作成したドメイン名を選択→「割り当てる」をクリックしてください。


## 3.5. SSL証明書を発行する

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


## 3.6. .envファイルを作成する

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

## 3.7. Dockerを再起動

Dcokerを再起動し、HTTPS化に成功していることを確認します。

```bash
sudo docker compose down
sudo docker compose up
```

`https://<独自ドメイン>/install` にアクセスできれば、HTTPS化成功です。

# 4. まとめ

DifyをAmazon LightsailでHTTPSデプロイする方法を詳しく説明しました。Difyは、AIアプリケーションを簡単に開発できる強力なツールです。この記事が、皆様のDify活用のお役に立てば幸いです。

# 5. 参考

- [DifyをAmazon Lightsailで動かす](https://qiita.com/hideki/items/5f76901ad790c8c14d6d)
- [絶対に失敗しないDifyデプロイの手順、AWS Lightsail編](https://note.com/sangmin/n/nbb4db69784e8)
- [DifyをHTTPSでデプロイする(AWS LightSail)](https://zenn.dev/shoheiweb/articles/f5627d03019620)
- [Dify 0.6.12をHTTPS化する方法](https://note.com/hisaki_nambu/n/nbb02fd2e1113)

