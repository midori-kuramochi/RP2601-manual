# 2台構成 WordPress（AL2023）
「2台構築：[Web+AP]/[DB]の2台構成でWordpressを構築する。」

---

## 構成
- **Web+APサーバ（1台目）**：Apache + PHP-FPM + WordPress  
- **DBサーバ（2台目）**：MariaDB

以降の表記：
- Web+AP（1台目）のプライベートIP：`WEB_PRIV`
- DB（2台目）のプライベートIP：`DB_PRIV`

---

## 0. セキュリティグループ（最小）

### Web+AP SG（1台目）
- Inbound  
  - TCP 22（自分のIPから）  
  - TCP 80（必要なら 0.0.0.0/0）  
- Outbound  
  - 全許可（演習ならそのままでOK）

### DB SG（2台目）
- Inbound  
  - TCP 22（自分のIPから）  
  - TCP 3306（**Web+APのSGから許可** ※IP直指定よりSG参照が安全でラク）  
- Outbound  
  - 全許可（演習ならそのままでOK）

---

## 1. DBサーバ構築（2台目）

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
- rootパスワード設定
- test DB削除
- 匿名ユーザー削除
- リモートrootログイン禁止（Yes推奨）
- privilege reload（Yes）

### 1-3. リモート接続を受ける設定（重要）
MariaDBが `127.0.0.1` だけで待ち受けていると、Web+APから繋がりません。

1) 待受確認
```bash
sudo ss -lntp | grep 3306 || true
```

2) 必要なら bind-address を変更  
（設定ファイルは環境で場所が違うので、まず探索）
```bash
sudo grep -R "bind-address" /etc/my.cnf* /etc/mysql* 2>/dev/null || true
```

もし `bind-address=127.0.0.1` なら、`0.0.0.0`（または `DB_PRIV`）に変更。  
例（ありがちな場所：`/etc/my.cnf.d/mariadb-server.cnf` 等）：
```ini
[mysqld]
bind-address=0.0.0.0
```

反映：
```bash
sudo systemctl restart mariadb
sudo ss -lntp | grep 3306
```

### 1-4. WordPress用DB/ユーザー作成
※ `WEB_PRIV` は **Web+APサーバのプライベートIP** を入れる

```bash
mysql -u root -p
```

MariaDB内：
```sql
CREATE DATABASE teamB_DB;

CREATE USER 'CL_teamB'@'WEB_PRIV' IDENTIFIED BY '強いパスワード';

GRANT ALL PRIVILEGES ON teamB_DB.* TO 'CL_teamB'@'WEB_PRIV';

FLUSH PRIVILEGES;
EXIT;
```

---

## 2. Web+APサーバ構築（1台目）

### 2-1. Apache/PHP/WordPress準備
```bash
sudo dnf upgrade -y

# Web+AP側はDBサーバではないので mariadb105-server は入れない
sudo dnf install -y httpd php-fpm php-mysqli php-json php-gd wget tar mariadb105

sudo systemctl enable --now httpd
sudo systemctl enable --now php-fpm

sudo systemctl status httpd
sudo systemctl status php-fpm
```

ブラウザで確認：  
`http://<グローバルIP>/` → “It works!” ならOK

### 2-2. 権限（シンプル版）
演習ならまずこれで十分（細かい2775運用はあとででもOK）：

```bash
sudo chown -R apache:apache /var/www
sudo find /var/www -type d -exec chmod 0755 {} \;
sudo find /var/www -type f -exec chmod 0644 {} \;
```

### 2-3. PHP動作確認（やったら消す）
```bash
echo "<?php phpinfo(); ?>" | sudo tee /var/www/html/phpinfo.php >/dev/null
```
ブラウザ：`http://<グローバルIP>/phpinfo.php` が出ればOK

削除：
```bash
sudo rm -f /var/www/html/phpinfo.php
```

---

## 3. WordPress配置（1台目）

### 3-1. WordPress取得＆配置（rsync推奨）
```bash
cd /tmp
wget https://wordpress.org/latest.tar.gz
tar -xzf latest.tar.gz

# cp でもいいけど、上書き事故が少ない rsync 推奨
sudo rsync -av wordpress/ /var/www/html/
```

### 3-2. wp-config.php 作成（2台構成なのでDB_HOSTが重要）
```bash
cd /var/www/html
sudo cp wp-config-sample.php wp-config.php
sudo vi wp-config.php
```

最低限ここを設定：
- `DB_NAME` = `teamB_DB`
- `DB_USER` = `CL_teamB`
- `DB_PASSWORD` = 作ったパスワード
- `DB_HOST` = `DB_PRIV`（例：`172.31.x.x`）

※必要なら明示でポート： `DB_PRIV:3306`

編集後、所有権：
```bash
sudo chown -R apache:apache /var/www/html
```

---

## 4. Web→DB疎通チェック（ここで詰まるので入れる）

### 4-1. Web+AP → DB へ mysql接続
```bash
mysql -u CL_teamB -p -h DB_PRIV teamB_DB
```
入れたらOK（`exit`で抜ける）

### 4-2. もし WordPress で “DB接続確立エラー”が出たら（最短チェック3点）
1) DB側で3306待ち受けしてる？  
   `sudo ss -lntp | grep 3306`

2) SGでDBの3306がWeb+APから許可されてる？  
   （DB SGに Web+AP SG を指定）

3) **SELinux** が原因のパターン（AL2023でありがち）  
Web+APで：
```bash
sudo getenforce
sudo setsebool -P httpd_can_network_connect_db 1
```

---

## 5. ブラウザでセットアップ
`http://<グローバルIP>/` にアクセス → WordPress初期画面が出れば完了。

---

## 「なんで？」に答える（手順書に1行補足するならここ）
- `wp-config.php を /var/www/html に置く理由`  
  → Apacheの公開ディレクトリ（DocumentRoot）が `/var/www/html` だから。ここにWP一式があるとブラウザから実行できる。

- Web+APに `mariadb105-server` が不要な理由  
  → DBは2台目。1台目は接続確認の **mysqlコマンド（クライアント）** があれば十分。
