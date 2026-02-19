# 権威DNS + WordPress公開構築手順書（2台構成 / Amazon Linux 2023）

---

## 1. 演習概要

新規プロダクト開発に伴い、開発チームからインフラチームへ以下の作業依頼がありました。

- 権威DNS（BIND）を構築する
- WordPress を構築する
- インターネット上から **ドメイン名でアクセス** できるようにする
- `entrycl.net` から払い出されたサブドメインについて、講師（情シス担当）へ **権限移譲依頼** を行う
- WordPress から MariaDB へ接続する際、**IPアドレスではなくホスト名で接続** する

---

## 2. 前提・要件

### 2-1. ドメイン情報
- 払い出し済みサブドメイン：`kuramochi.entrycl.net`
- パブリックIPv4アドレス（公開用）：`44.251.154.245`

> ※ 本手順書では、公開用IPとして `44.251.154.245` を利用します。  
> `ns` と `www` は公開用AレコードとしてこのIPを設定します。  
> （演習環境により1つの公開IPに集約される想定）

---

### 2-2. サーバ構成（2台）
- **1台目（Webサーバ）**：Apache + PHP-FPM
  - Private IP：`172.31.30.27`
- **2台目（DB/DNSサーバ）**：MariaDB + BIND
  - Private IP：`172.31.28.190`

---

### 2-3. WordPress DB接続情報（指定値）
```php
define( 'DB_NAME', 'DB_kuramochi' );
define( 'DB_USER', 'kuramochi' );
define( 'DB_PASSWORD', 'kuramochi' );
define( 'DB_HOST', 'webサーバのプライベートIP' );   // ← この値を最終的にホスト名へ変更する
```

---

## 3. ゴール（達成条件）

- BIND（named）が起動し、権威DNSとして `kuramochi.entrycl.net` のゾーンを管理している
- `ns` / `www` / `db` レコードが正しく引ける
- WordPress の `DB_HOST` が **IPではなくホスト名**（`db.kuramochi.entrycl.net`）になっている
- `http://www.kuramochi.entrycl.net/` でWordPressが表示される
- 講師へ権限移譲依頼を実施できる状態になっている

---

## 4. 作業全体の流れ

1. DB/DNSサーバでBINDの設定確認・ゾーン作成
2. `ns / www / db` レコードを定義
3. BIND起動・`dig`で名前解決確認
4. Webサーバ側でDNS解決設定（必要に応じて）
5. `wp-config.php` の `DB_HOST` をホスト名に変更
6. Apache/WordPressの表示確認
7. 公開用Aレコード（`ns`,`www`,`@`）を設定
8. 講師へ権限移譲依頼
9. 外部からドメインアクセス確認

---

## 5. 事前確認（セキュリティグループ）

### 5-1. Webサーバ（172.31.30.27）
- SSH（22/TCP）… 自分のIP
- HTTP（80/TCP）… 外部公開（0.0.0.0/0）※演習要件に応じて
- （必要なら）HTTPS（443/TCP）… 外部公開

### 5-2. DB/DNSサーバ（172.31.28.190）
- SSH（22/TCP）… 自分のIP
- DNS（53/UDP）… 外部公開（権威DNSとして応答するため）
- DNS（53/TCP）… 外部公開（同上）
- MariaDB（3306/TCP）… WebサーバPrivate IPから許可(VPCCIDR)

---

## 6. DB/DNSサーバ（172.31.28.190）で権威DNSを設定する

> ※ BIND のインストール自体は完了済み前提。  
> ここでは「動作する構成」にするための確認・設定をまとめています。

---

### 6-1. named.conf のバックアップ

```bash
sudo su -
cp /etc/named.conf /etc/named.conf.$(date +%Y%m%d-%H%M%S).bak
```

---

### 6-2. `/etc/named.conf` の設定確認（重要）

```bash
vi /etc/named.conf
```

#### ① `options` ブロックの確認ポイント
- `listen-on` に DB/DNSサーバのPrivate IP (`172.31.28.190`) が入っていること
- `allow-query` が問い合わせ元を許可していること（演習ではVPC内 + 外部確認を考慮）

例：
```conf
options {
        #listen-on port 53 { 127.0.0.1; 172.31.28.190; };
        #listen-on-v6 port 53 { ::1; };

        allow-query     { any; };

};
```

> `allow-query { any; };` を含めると外部からも問い合わせ可能になります（権威DNSとして使うため）。

---

#### ② ゾーン定義を追加（`options {}` の外）

```conf
zone "kuramochi.entrycl.net" IN {
        type master;
        file "kuramochi.entrycl.net.zone";
};
```

> **注意**：`zone "..."` は `options {}` の中に書かないこと

---

### 6-3. ゾーンファイルを作成（`/var/named/kuramochi.entrycl.net.zone`）

```bash
vi /var/named/kuramochi.entrycl.net.zone
```

#### ゾーンファイル（完成形サンプル）
```zone
$TTL 3600
@   IN SOA  ns.kuramochi.entrycl.net. root.kuramochi.entrycl.net. (
        2026021901 ; serial
        3600       ; refresh
        900        ; retry
        604800     ; expire
        86400      ; minimum
)

    IN NS   ns.kuramochi.entrycl.net.

; 裸ドメイン(apex)もWebへ向ける
@   IN A    <権威DNSサーバのPublicIP>

; 権威DNSサーバ
ns  IN A    <権威DNSサーバのPublicIP>

; WordPress公開先
www IN A    <WebサーバのPublicIP>

; DB接続用（内部向け）
db  IN A    <DBサーバのprivateIP>
```

### レコードの意味
- `@`：`kuramochi.entrycl.net`（裸ドメイン）
- `ns`：権威DNSのホスト名
- `www`：WordPress公開用ホスト名
- `db`：WordPress→MariaDB 接続用ホスト名（内部用）

---

### 6-4. ゾーンファイルの権限・SELinuxコンテキスト確認（念のため）

```bash
chown root:named /var/named/kuramochi.entrycl.net.zone
chmod 640 /var/named/kuramochi.entrycl.net.zone
restorecon -v /var/named/kuramochi.entrycl.net.zone
```

---

### 6-5. 構文チェック → named再起動

```bash
named-checkconf
named-checkzone kuramochi.entrycl.net /var/named/kuramochi.entrycl.net.zone
systemctl restart named
systemctl status named -l --no-pager
```

> `named-checkzone` が `OK`、`named` が `active (running)` ならOK

---

### 6-6. DNS待ち受け確認（53番ポート）

```bash
ss -lntup | grep ':53'
```

確認ポイント：
- `127.0.0.1:53`
- `172.31.28.190:53`

---

## 7. 権威DNSとしての名前解決確認（DB/DNSサーバ上）

### 7-1. localhost宛て確認（named自身）
```bash
dig @localhost ns.kuramochi.entrycl.net
dig @localhost www.kuramochi.entrycl.net
dig @localhost db.kuramochi.entrycl.net
dig @localhost kuramochi.entrycl.net
```

### 7-2. 自分のBIND宛て確認
```bash
dig @[DNSサーバのprivateIP] ns.kuramochi.entrycl.net
dig @[WebサーバのprivateIP] www.kuramochi.entrycl.net
dig @[DBサーバのprivateIP] db.kuramochi.entrycl.net
dig @[DNSサーバのprivateIP] kuramochi.entrycl.net
```

> `ANSWER SECTION` に期待するAレコードが返ればOK

---

## 8. Webサーバ（172.31.30.27）でDBホスト名解決を有効化する

要件：
> WordPressからDBへ接続する際はIPアドレスではなくてホスト名で接続する

そのため、Webサーバから `db.kuramochi.entrycl.net` が引ける必要があります。

---

### 8-1. WebサーバからBINDへ問い合わせ確認

Webサーバで実行：

```bash
dig @[DBサーバのprivateIP]  db.kuramochi.entrycl.net
```

`ANSWER SECTION` に `172.31.28.190` が返ればOK。

---

### 8-2. WebサーバのDNS設定確認（`/etc/resolv.conf`）

```bash
cat /etc/resolv.conf
```

必要に応じて、DB/DNSサーバを名前解決先に設定（演習向け）：

```bash
sudo vi /etc/resolv.conf
```

```conf
nameserver 172.31.28.190
```

> 注意：環境によっては再起動時などに上書きされることがあります。  
> まずは `dig @172.31.28.190 ...` が通ることを確認し、その後 `getent` / `php` で確認するのがおすすめです。

---

### 8-3. Webサーバ上で名前解決確認（OS/PHP）

```bash
getent hosts db.kuramochi.entrycl.net
```

または

```bash
php -r 'echo gethostbyname("db.kuramochi.entrycl.net"), PHP_EOL;'
```

期待結果：
```text
172.31.28.190
```

---

## 9. WordPressの `DB_HOST` をIP → ホスト名へ変更する（要件達成）

---

### 9-1. `wp-config.php` の場所を確認

Webサーバで実行：

```bash
sudo find /var/www -name wp-config.php
```

例：
- `/var/www/html/wordpress/wp-config.php`

---

### 9-2. `wp-config.php` を編集

```bash
sudo vi /var/www/html/wordpress/wp-config.php
```

#### 変更前（指定値）
```php
define( 'DB_NAME', 'DB_kuramochi' );
define( 'DB_USER', 'kuramochi' );
define( 'DB_PASSWORD', 'kuramochi' );
define( 'DB_HOST', 'webサーバのプライベートIP' ); 
```

#### 変更後（要件対応）
```php
define( 'DB_NAME', 'DB_kuramochi' );
define( 'DB_USER', 'kuramochi' );
define( 'DB_PASSWORD', 'kuramochi' );
define( 'DB_HOST', 'db.kuramochi.entrycl.net' );
```

---

### 9-3. WordPressの画面表示確認（WebサーバIPで一度確認）

```text
http://<WebサーバのパブリックIP>/   または   http://<WebサーバのIP>/wordpress
```

> すでに画面表示OKとのことなので、DB_HOST変更後も表示できれば要件達成です。

---

## 10. Apache側のドメイン受け口を確認（www / 裸ドメイン対応）

`http://www.kuramochi.entrycl.net/` で表示させるには、ApacheのVirtualHostがドメイン名を受けられる状態だと安心です。

### 10-1. VirtualHost設定例（推奨）

```bash
sudo vi /etc/httpd/conf.d/wordpress.conf
```

```apache
<VirtualHost *:80>
    ServerName www.kuramochi.entrycl.net
    ServerAlias kuramochi.entrycl.net
    DocumentRoot /var/www/html/wordpress

    <Directory /var/www/html/wordpress>
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog /var/log/httpd/wordpress_error.log
    CustomLog /var/log/httpd/wordpress_access.log combined
</VirtualHost>
```

### 10-2. Apache設定チェック・再起動
```bash
sudo httpd -t
sudo systemctl restart httpd
sudo systemctl status httpd
```

---

## 11. 講師（情シス担当）への権限移譲依頼

`entrycl.net` の親ゾーンから、`kuramochi.entrycl.net` をあなたの権威DNSへ委譲してもらう必要があります。

### 11-1. 伝える情報（必須）
- 委譲希望ドメイン：`kuramochi.entrycl.net`
- 委譲先ネームサーバ：`ns.kuramochi.entrycl.net`
- ネームサーバIP（glue用）：`44.251.154.245`

---

### 11-2. 依頼文テンプレ（そのまま使える）

```text
件名：kuramochi.entrycl.net の権限移譲依頼

お疲れさまです。
新規プロダクト開発に伴う権威DNS構築の件で、下記サブドメインの権限移譲をお願いします。

【委譲希望ドメイン】
kuramochi.entrycl.net

【委譲先ネームサーバ】
ns.kuramochi.entrycl.net

【ネームサーバIP（glue用）】
44.251.154.245

以上、よろしくお願いします。
```

---

## 12. 公開確認（委譲反映後）

### 12-1. DNS確認
```bash
dig ns.kuramochi.entrycl.net
dig www.kuramochi.entrycl.net
dig kuramochi.entrycl.net
```

確認ポイント：
- `ANSWER SECTION` に `44.251.154.245` が返ること
- `db.kuramochi.entrycl.net` は内部用なので、必要に応じて社内/サーバ内で確認

---

### 12-2. ブラウザ確認
- `http://www.kuramochi.entrycl.net/`
- `http://kuramochi.entrycl.net/`（裸ドメインも `@` を定義していれば表示可能）

> 演習では、まず **`http://www.kuramochi.entrycl.net/`** で確認するのが確実です。

---

## 13. よくあるトラブルと対処

---

### 13-1. `systemctl start named` が失敗する
#### よくある原因
- ゾーンファイル名の不一致（`/var/named/kuramochi.entrycl.net` と `.zone` の違い）
- `named.conf` の `zone` 定義ミス
- `;` 抜け、`}` 抜け

#### 確認コマンド
```bash
systemctl status named -l --no-pager
journalctl -xeu named.service --no-pager | tail -n 50
named-checkconf
named-checkzone kuramochi.entrycl.net /var/named/kuramochi.entrycl.net.zone
```

---

### 13-2. `dig @localhost ...` で `connection refused`
#### 原因
- `named` が起動していない
- 53番で待ち受けていない
- 設定エラーで `named` が落ちている

#### 確認
```bash
systemctl status named
ss -lntup | grep ':53'
```

---

### 13-3. `http://kuramochi.entrycl.net` で表示されない
#### 原因候補
- ゾーンに `@ IN A ...`（裸ドメインAレコード）がない
- Apacheの `ServerAlias kuramochi.entrycl.net` がない

#### 対処
- ゾーンに `@ IN A 44.251.154.245` を追加
- Apache VirtualHostに `ServerAlias kuramochi.entrycl.net` を追加

---

### 13-4. WordPressがDB接続エラーになる
#### 原因候補
- `db.kuramochi.entrycl.net` がWebサーバで引けない
- `DB_HOST` がIPのまま / 文字列ミス
- MariaDB側の接続許可・ユーザー権限不足

#### 確認
```bash
dig @172.31.28.190 db.kuramochi.entrycl.net
getent hosts db.kuramochi.entrycl.net
php -r 'echo gethostbyname("db.kuramochi.entrycl.net"), PHP_EOL;'
```

---

## 14. 完成チェックシート（提出前確認）

### DNS（BIND）
- [ ] `named` が起動している
- [ ] `named-checkconf` がOK
- [ ] `named-checkzone` がOK
- [ ] `ns.kuramochi.entrycl.net` が引ける
- [ ] `www.kuramochi.entrycl.net` が引ける
- [ ] `db.kuramochi.entrycl.net` が引ける
- [ ] `kuramochi.entrycl.net`（裸ドメイン）も引ける

### WordPress
- [ ] WordPress画面が表示される
- [ ] `wp-config.php` の `DB_HOST` が `db.kuramochi.entrycl.net` になっている
- [ ] ApacheのVirtualHostで `www` と裸ドメインを受けられる

### 公開・委譲
- [ ] `ns` / `www` / `@` のAレコードが公開IP `44.251.154.245` になっている
- [ ] 講師へ権限移譲依頼（NS + glue IP）を送れる情報がそろっている
- [ ] `http://www.kuramochi.entrycl.net/` で表示確認できる

---

## 15. 参考コマンド集（よく使う）

```bash
# BIND設定チェック
named-checkconf
named-checkzone kuramochi.entrycl.net /var/named/kuramochi.entrycl.net.zone

# named状態確認
systemctl status named -l --no-pager
journalctl -xeu named.service --no-pager | tail -n 50

# 53番ポート確認
ss -lntup | grep ':53'

# 権威DNS問い合わせ（BIND自身）
dig @localhost ns.kuramochi.entrycl.net
dig @localhost www.kuramochi.entrycl.net
dig @localhost db.kuramochi.entrycl.net
dig @localhost kuramochi.entrycl.net

# WebサーバからDBホスト名確認
dig @172.31.28.190 db.kuramochi.entrycl.net
getent hosts db.kuramochi.entrycl.net
php -r 'echo gethostbyname("db.kuramochi.entrycl.net"), PHP_EOL;'

# Apache確認
httpd -t
systemctl restart httpd
systemctl status httpd
```

---
