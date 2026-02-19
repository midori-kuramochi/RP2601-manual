# 演習2：Nginx の構築手順書（Amazon Linux 2023）

## 目的
Apache ではなく **nginx** を Amazon Linux 2023 にインストールし、起動・自動起動設定・ログ/ローテーション確認までを実施する。

---

## 前提
- OS：Amazon Linux 2023
- インスタンスに SSH 接続できること
- セキュリティグループ（またはFW）で HTTP を許可していること  
  - Inbound：TCP **80**（必要なら 443）
- 操作ユーザー：sudo 権限あり

---

## 構築手順

### 1. パッケージの確認
Amazon Linux 2023 は `amazon-linux-extras` を使わずとも `dnf` で nginx を導入できるため、存在確認を行う。

```bash
dnf search nginx
dnf info nginx
```

---

### 2. nginx のインストール
```bash
sudo dnf -y install nginx
nginx -v
```

---

### 3. nginx を起動する（疎通確認）
```bash
sudo systemctl start nginx
sudo systemctl status nginx --no-pager
```

設定ファイルの構文チェック（今後の作業でも必須）
```bash
sudo nginx -t
```

ブラウザで以下へアクセスし、nginx のデフォルト画面が表示されれば OK  
`http://<EC2のパブリックIP>/`

表示例：
![alt text](<スクリーンショット 2026-02-18 161716.png>)

---

### 4. 自動起動設定を入れる
再起動後も nginx が起動するよう自動起動を有効化する。

```bash
sudo systemctl enable nginx
systemctl is-enabled nginx
```

---

### 5. 設定ファイルとドキュメントルート（把握）
- メイン設定：`/etc/nginx/nginx.conf`
- 追加設定：`/etc/nginx/conf.d/*.conf`
- デフォルト公開ディレクトリ：`/usr/share/nginx/html`
- ログ出力先：`/var/log/nginx/`

ドキュメントルート確認例（必要に応じて）
```bash
sudo grep -R "root " -n /etc/nginx | head
```

---

### 6. ログについて（権限は原則変更しない）
nginx のログは通常 `/var/log/nginx/` に出力される。

- `access.log`
- `error.log`

> **注意**：`/var/log/nginx` の所有者や権限を変更して world-readable（o+r）にするのは推奨しない。  
> ログ閲覧は `sudo` を使う（アクセスログにはIP/パス/クエリ等が含まれる可能性がある）。

確認例：
```bash
sudo tail -n 50 /var/log/nginx/access.log
sudo tail -n 50 /var/log/nginx/error.log
```

---

### 7. logrotate 設定を確認する（必要に応じて変更）
まず現状を確認する。

```bash
sudo cat /etc/logrotate.d/nginx
```

#### 7-1. サブディレクトリ配下にもログを出す運用をする場合のみ
例：`/var/log/nginx/siteA/access.log` のように、サイトごとにディレクトリを切ってログを出す場合は、ローテート対象パターンを追加する。

```conf
/var/log/nginx/*.log /var/log/nginx/*/*.log {
    daily
    rotate 10
    missingok
    notifempty
    compress
    dateext
    sharedscripts
    postrotate
        /bin/kill -USR1 `cat /run/nginx.pid 2>/dev/null` 2>/dev/null || true
    endscript
}
```

- `dateext`：ローテート後のファイル名に日付を付けたい場合に追加（例：`access.log-20260218.gz`）
- `delaycompress`：運用方針で必要なら残す／不要なら削除で OK  
  - 理由は「運用上不要」など、要件ベースで記載すると筋が良い
- `create` のパーミッションは **基本デフォルト維持（例：0640）** を推奨  
  - `o+r`（誰でも読める）は避ける

---

### 8. logrotate 動作テスト（任意だが推奨）
```bash
sudo logrotate -d /etc/logrotate.d/nginx   # ドライラン（実行しない）
```

検証時のみ：
```bash
sudo logrotate -f /etc/logrotate.d/nginx   # 強制実行（検証時のみ）
```

---

### 9. 設定変更後の反映
設定を変更した場合は、必ず構文チェック後に reload する。

```bash
sudo nginx -t
sudo systemctl reload nginx
```

---

## 最終確認チェックリスト
- [ ] `nginx -v` でバージョンが表示される
- [ ] `systemctl status nginx` が active (running)
- [ ] ブラウザで `http://<パブリックIP>/` が表示される
- [ ] `systemctl is-enabled nginx` が enabled
- [ ] `/var/log/nginx/access.log` / `error.log` が出力されている
- [ ] logrotate 設定が要件に合っている（必要な場合のみ編集）

---

## 補足（トラブルシュート）
### ブラウザで表示されない
- セキュリティグループで TCP/80 が許可されているか
- nginx が起動しているか
  ```bash
  sudo systemctl status nginx --no-pager
  ```
- ローカルで疎通確認（サーバ内）
  ```bash
  curl -I http://127.0.0.1/
  ```

### 設定反映で失敗する
- 構文エラーがないか
  ```bash
  sudo nginx -t
  ```
- エラーログ確認
  ```bash
  sudo tail -n 100 /var/log/nginx/error.log
  ```
