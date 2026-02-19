# 3台構成 WordPress（AL2023）
「3台構築：[Web]/[AP]/[DB]の3台構成でWordPressを構築する。」

---

## 構成
- **Webサーバ（1台目）**：Apache（リバースプロキシ）  
- **APサーバ（2台目）**：PHP-FPM + WordPress（静的ファイル含む）  
- **DBサーバ（3台目）**：MariaDB

以降の表記：
- Web（1台目）プライベートIP：`WEB_PRIV`
- AP（2台目）プライベートIP：`AP_PRIV`
- DB（3台目）プライベートIP：`DB_PRIV`

---

## 0. セキュリティグループ（最小）

### Web SG（1台目）
- Inbound
  - TCP 22（自分のIPから）
  - TCP 80（必要なら 0.0.0.0/0）
- Outbound
  - TCP 80 → **AP SG**（HTTPでAPへ転送する場合）
  - もしくは演習なら全許可でOK

### AP SG（2台目）
- Inbound
  - TCP 22（自分のIPから）
  - TCP 80（**Web SGから**）  ※Web→APのHTTP
- Outbound
  - TCP 3306 → **DB SG**（AP→DB）

### DB SG（3台目）
- Inbound
  - TCP 22（自分のIPから）
  - TCP 3306（**AP SGから**）
- Outbound
  - 全許可（演習ならOK）

---

## 1. DBサーバ構築（3台目）

### 1-1. インストール＆起動
```bash
sudo dnf upgrade -y
sudo dnf install -y mariadb105-server
sudo systemctl enable --now mariadb
sudo systemctl status mariadb
```

### 1-2. 初期セキュリティ設定
```bash
sudo mysql_secure_installation
```

### 1-3. リモート接続を受ける設定（重要）
```bash
sudo ss -lntp | grep 3306 || true
sudo grep -R "bind-address" /etc/my.cnf* /etc/mysql* 2>/dev/null || true
```

もし `bind-address=127.0.0.1` なら `0.0.0.0`（または `DB_PRIV`）に変更：
```ini
[mysqld]
bind-address=0.0.0.0
```

反映：
```bash
sudo systemctl restart mariadb
sudo ss -lntp | grep 3306
```

### 1-4. WordPress用DB/ユーザー作成（接続元はAP）
※ `AP_PRIV` は **APサーバのプライベートIP**

```bash
mysql -u root -p
```

MariaDB内：
```sql
CREATE DATABASE wp_db;

CREATE USER 'wp_user'@'AP_PRIV' IDENTIFIED BY '強いパスワード';

GRANT ALL PRIVILEGES ON wp_db.* TO 'wp_user'@'AP_PRIV';

FLUSH PRIVILEGES;
EXIT;
```

---

## 2. APサーバ構築（2台目：PHP-FPM + WordPress）

### 2-1. インストール＆起動
```bash
sudo dnf upgrade -y
sudo dnf install -y php-fpm php-mysqli php-json php-gd wget tar mariadb105 rsync

sudo systemctl enable --now php-fpm
sudo systemctl status php-fpm
```

### 2-2. WordPress配置（AP側に置く）
```bash
cd /tmp
wget https://wordpress.org/latest.tar.gz
tar -xzf latest.tar.gz

# 公開用ディレクトリ（APのWebルート）を作る
sudo mkdir -p /var/www/html
sudo rsync -av wordpress/ /var/www/html/
sudo chown -R apache:apache /var/www/html || true
```

> 補足：APは「WordPressを実行する側」。Webは後でAPへ転送するだけなので、WordPress本体はAPに置く。

### 2-3. wp-config.php 作成（DB_HOSTはDB）
```bash
cd /var/www/html
sudo cp wp-config-sample.php wp-config.php
sudo vi wp-config.php
```

最低限ここを設定：
- `DB_NAME` = `wp_db`
- `DB_USER` = `wp_user`
- `DB_PASSWORD` = 作ったパスワード
- `DB_HOST` = `DB_PRIV`（必要なら `DB_PRIV:3306`）

### 2-4. AP → DB疎通チェック（最重要）
```bash
mysql -u wp_user -p -h DB_PRIV wp_db
```

### 2-5. SELinux（詰まりやすい場合）
```bash
sudo getenforce
sudo setsebool -P httpd_can_network_connect_db 1
```

---

## 3. Webサーバ構築（1台目：Apache 리バプロ）

### 3-1. インストール＆起動
```bash
sudo dnf upgrade -y
sudo dnf install -y httpd
sudo systemctl enable --now httpd
sudo systemctl status httpd
```

### 3-2. リバースプロキシ設定（Web → AP）
Apacheで `mod_proxy` を使って、受けたHTTPをAPへ転送します。

1) モジュール確認（入っていればOK）
```bash
sudo httpd -M | egrep "proxy_module|proxy_http_module" || true
```

2) 設定ファイル作成
```bash
sudo vi /etc/httpd/conf.d/reverse-proxy.conf
```

中身（最小）：
```apache
ProxyRequests Off

ProxyPass        /  http://AP_PRIV/
ProxyPassReverse /  http://AP_PRIV/
```

3) 反映
```bash
sudo httpd -t
sudo systemctl restart httpd
```

---

## 4. 動作確認（ブラウザ）
- ブラウザで `http://<WebのグローバルIP>/`  
  → WordPress初期セットアップ画面が出ればOK

---

## 5. よく詰まるポイント（最短チェック）

### A) Webは表示するが、WP画面が崩れる/404が出る
- Web → AP の疎通
```bash
# WebからAPへ
curl -I http://AP_PRIV/ | head
```
- AP側で `/var/www/html` にWPがあるか
```bash
ls -la /var/www/html | head
```

### B) “DB接続確立エラー”
1) AP → DB のmysql接続が通るか  
2) DBの3306待受（bind-address）  
3) DB SG：3306がAP SGから許可されてるか  
4) SELinux（AP側）：`setsebool -P httpd_can_network_connect_db 1`

---

## 補足（この構成の考え方）
- **Web**：入口。外部公開してHTTPを受け、APへ転送する  
- **AP**：WordPress/PHP-FPMでアプリを動かす  
- **DB**：データを保存する（APだけが接続）
