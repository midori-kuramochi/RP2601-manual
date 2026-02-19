# 追加演習3 virtualhost

「Apache(httpd)のVirtualHostで複数サイトをホスティングする」

---

### 演習1 手順書

##### 目的

1台のWebサーバ（Apache/httpd）で、以下3つのWebサイトを**名前ベースVirtualHost**で切り替えて表示する。

* `apple.kuramochi.entrycl.net` → **Apple**
* `grape.kuramochi.entrycl.net` → **Grape**
* `orange.kuramochi.entrycl.net` → **Orange**

---

### 0. 前提・構成

* Webサーバ：Amazon Linux系（dnf が使える想定）
* Web：Apache(httpd)
* DNS：`kuramochi.entrycl.net` は **あなたの権威DNS**で運用（BIND/named想定）
* Web公開：インターネットからアクセス（Public IP）

---

### 1. Webサーバ：Apacheインストール & 起動確認

##### 1-1. インストール

```bash
sudo su -
dnf update -y
dnf install -y httpd
```

##### 1-2. 起動・自動起動

```bash
systemctl enable --now httpd
systemctl status httpd
```

`Active: active (running)` を確認。

##### 1-3. 動作確認（IPで it works!）

ブラウザで下記にアクセスし、`It works!` が表示されること：

* `http://<WebサーバのPublicIP>/`

---

### 2. Webサーバ：コンテンツ（Apple/Grape/Orange）作成

##### 2-1. DocumentRoot用ディレクトリ作成

```bash
mkdir -p /var/www/html/{apple,grape,orange}
```

##### 2-2. 各サイトの index.html 作成

```bash
cat > /var/www/html/apple/index.html <<'EOF'
<!doctype html>
<html><head><meta charset="utf-8"><title>Apple</title></head>
<body><h1>Apple</h1></body></html>
EOF

cat > /var/www/html/grape/index.html <<'EOF'
<!doctype html>
<html><head><meta charset="utf-8"><title>Grape</title></head>
<body><h1>Grape</h1></body></html>
EOF

cat > /var/www/html/orange/index.html <<'EOF'
<!doctype html>
<html><head><meta charset="utf-8"><title>Orange</title></head>
<body><h1>Orange</h1></body></html>
EOF
```

---

### 3. DNS：サブドメインをWebサーバPublic IPへ向ける

##### 3-1. WebサーバPublic IPを確認

Webサーバ上で（どちらか）：

```bash
curl -s ifconfig.me; echo
# または
curl -s https://checkip.amazonaws.com; echo
```

以降、このIPを `<WEB_PUBLIC_IP>` として使う。

##### 3-2. 権威DNS（BIND）のゾーンに Aレコード追加

あなたが運用している `kuramochi.entrycl.net` のゾーンファイルに追記：

例（ゾーンファイルの追記イメージ）：

```dns
apple   IN  A   <WEB_PUBLIC_IP>
grape   IN  A   <WEB_PUBLIC_IP>
orange  IN  A   <WEB_PUBLIC_IP>
```

##### 3-3. named反映（DNSサーバ側）

（環境に合わせて実行）

```bash
sudo named-checkzone kuramochi.entrycl.net /var/named/kuramochi.entrycl.net.zone
sudo systemctl reload named
```

##### 3-4. 名前解決確認（どこでもOK）

```bash
dig apple.kuramochi.entrycl.net A +short
dig grape.kuramochi.entrycl.net A +short
dig orange.kuramochi.entrycl.net A +short
```

すべて `<WEB_PUBLIC_IP>` が返ること。

---

### 4. Webサーバ：VirtualHost 設定

##### 4-1. VirtualHost設定ファイル作成

Webサーバで以下を作成：

```bash
vi /etc/httpd/conf.d/virtualhost.conf
```

中身（そのまま貼り付けOK）：

```apache
<VirtualHost *:80>
    ServerName apple.kuramochi.entrycl.net
    DocumentRoot /var/www/html/apple

    ErrorLog  /var/log/httpd/apple_error.log
    CustomLog /var/log/httpd/apple_access.log combined
</VirtualHost>

<VirtualHost *:80>
    ServerName grape.kuramochi.entrycl.net
    DocumentRoot /var/www/html/grape

    ErrorLog  /var/log/httpd/grape_error.log
    CustomLog /var/log/httpd/grape_access.log combined
</VirtualHost>

<VirtualHost *:80>
    ServerName orange.kuramochi.entrycl.net
    DocumentRoot /var/www/html/orange

    ErrorLog  /var/log/httpd/orange_error.log
    CustomLog /var/log/httpd/orange_access.log combined
</VirtualHost>
```

> ポイント
>
> * `ServerName` は **FQDN（完全修飾ドメイン名）**
> * `DocumentRoot` は **ディレクトリ**
> * ログをサイト別に分離（確認が楽）

---

### 5. Apache反映（構文チェック→再起動）

##### 5-1. 構文チェック

```bash
apachectl configtest
```

`Syntax OK` を確認。

##### 5-2. 反映

```bash
systemctl reload httpd
# うまくいかない場合は restart
# systemctl restart httpd
```

---

### 6. セキュリティグループ確認（AWS）

WebサーバのSGで下記を許可：

* Inbound：TCP **80**（HTTP）

  * Source：演習なら `0.0.0.0/0` でもOK（指示があれば制限）

---

### 7. 動作確認（DNS→HTTP）

##### 7-1. curlで確認（どこでも可）

```bash
curl -s http://apple.kuramochi.entrycl.net  | head
curl -s http://grape.kuramochi.entrycl.net  | head
curl -s http://orange.kuramochi.entrycl.net | head
```

##### 7-2. ブラウザで確認

* `http://apple.kuramochi.entrycl.net` → **Apple**
* `http://grape.kuramochi.entrycl.net` → **Grape**
* `http://orange.kuramochi.entrycl.net` → **Orange**

---

## 8. うまくいかない時の切り分け（最短）

##### 8-1. DNSが正しいか

```bash
dig apple.kuramochi.entrycl.net A +short
```

→ IPが返らない / 違うIPなら **DNS側**。

##### 8-2. ApacheがVirtualHostを読んでるか

```bash
httpd -S
```

→ `apple/grape/orange` の VirtualHost が出てくるか確認。

##### 8-3. ログを見る

```bash
tail -n 50 /var/log/httpd/*error.log
```

---