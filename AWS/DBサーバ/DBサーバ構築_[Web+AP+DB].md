# 1台構成 WordPress（AL2023）
「1台構築：[Web+AP+DB] を同一サーバ上でWordPressを構築する。」

---

## 構成
- **1台（Web+AP+DB）**：Apache + PHP-FPM + MariaDB + WordPress

以降の表記：
- サーバのグローバルIP：`PUBLIC_IP`

---

## 0. セキュリティグループ（最小）
- Inbound
  - TCP 22（自分のIPから）
  - TCP 80（必要なら 0.0.0.0/0）
- Outbound
  - 全許可（演習ならそのままでOK）

> 1台構成ではDBはローカル接続（localhost）なので、3306を外部公開しないのが基本。

---

## 1. OS更新
```bash
sudo dnf upgrade -y
```

---

## 2. ミドルウェア導入（Apache / PHP-FPM / MariaDB）
```bash
# Web+AP+DBを全部入れる
sudo dnf install -y httpd php-fpm php-mysqli php-json php-gd wget tar mariadb105-server

# 起動＆自動起動
sudo systemctl enable --now httpd
sudo systemctl enable --now php-fpm
sudo systemctl enable --now mariadb

# 状態確認
sudo systemctl status httpd
sudo systemctl status php-fpm
sudo systemctl status mariadb
```

ブラウザ確認：  
`http://PUBLIC_IP/` → “It works!” が出ればApache OK

---

## 3. MariaDB 初期設定
```bash
sudo mysql_secure_installation
```
- rootパスワード設定
- test DB削除
- 匿名ユーザー削除
- リモートrootログイン禁止（Yes推奨）
- privilege reload（Yes）

---

## 4. WordPress用DB/ユーザー作成（ローカル接続）
```bash
mysql -u root -p
```

MariaDB内：
```sql
CREATE DATABASE wp_db;

CREATE USER 'wp_user'@'localhost' IDENTIFIED BY '強いパスワード';

GRANT ALL PRIVILEGES ON wp_db.* TO 'wp_user'@'localhost';

FLUSH PRIVILEGES;
EXIT;
```

---

## 5. 権限（シンプル版）
演習ならまずこれで十分（細かい2775運用はあとででもOK）：

```bash
sudo chown -R apache:apache /var/www
sudo find /var/www -type d -exec chmod 0755 {} \;
sudo find /var/www -type f -exec chmod 0644 {} \;
```

---

## 6. PHP動作確認（やったら消す）
```bash
echo "<?php phpinfo(); ?>" | sudo tee /var/www/html/phpinfo.php >/dev/null
```

ブラウザ：`http://PUBLIC_IP/phpinfo.php` が表示されればOK

削除：
```bash
sudo rm -f /var/www/html/phpinfo.php
```

---

## 7. WordPress配置

### 7-1. 取得＆配置（rsync推奨）
```bash
cd /tmp
wget https://wordpress.org/latest.tar.gz
tar -xzf latest.tar.gz

# cpでもOKだが、上書き事故が少ない rsync 推奨
sudo rsync -av wordpress/ /var/www/html/
```

### 7-2. wp-config.php 作成（DB_HOSTはlocalhost）
```bash
cd /var/www/html
sudo cp wp-config-sample.php wp-config.php
sudo vi wp-config.php
```

最低限ここを設定：
- `DB_NAME` = `wp_db`
- `DB_USER` = `wp_user`
- `DB_PASSWORD` = 作ったパスワード
- `DB_HOST` = `localhost`

編集後、所有権：
```bash
sudo chown -R apache:apache /var/www/html
```

---

## 8. SELinux（詰まりやすい場合）
DBはローカルだが、環境によっては制限で詰まることがあります。

```bash
sudo getenforce
# Enforcing の場合、必要なら許可
sudo setsebool -P httpd_can_network_connect_db 1
```

---

## 9. ブラウザでセットアップ
`http://PUBLIC_IP/` にアクセス → WordPress初期画面が出れば完了。

---

## 10. よく詰まるポイント（最短チェック）

### A) “Error establishing a database connection”
1) DBサービス起動確認
```bash
sudo systemctl status mariadb
```
2) ユーザー/DB名/パスが一致しているか（wp-config.php）
3) 手動でログインできるか
```bash
mysql -u wp_user -p -h localhost wp_db
```

### B) パーミッション系（アップロードできない等）
```bash
sudo chown -R apache:apache /var/www/html
sudo find /var/www/html -type d -exec chmod 0755 {} \;
sudo find /var/www/html -type f -exec chmod 0644 {} \;
```

---

## 補足（この構成の考え方）
- 1台構成はシンプルだが、負荷分散/冗長化はできない  
- DBは `localhost` で接続する（外部に3306を開けない）
