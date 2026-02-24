# Webサーバ構築手順書（Apache / httpd）

この手順書は、Linuxサーバに Apache（httpd）をインストールし、ブラウザで表示できるところまでをゴールにします。  
「DocumentRoot（公開フォルダ）の変更」や「リダイレクト設定」も含めます。

---

## 0. 事前準備（セキュリティグループ）

### インバウンド（受信）ルール
- SSH（22）：**自分のグローバルIP（マイIP）**
- HTTP（80）：**自分のグローバルIP（マイIP）**

> ✅ 自分だけが確認できればOKの場合は「マイIP」で問題ありません。  
> ⚠️ 誰でも閲覧できるWebサイトにするなら、HTTP（80）を `0.0.0.0/0` にする必要があります。

---

## 1. Apache（httpd）のインストール

### 1-1. root に切り替え
```bash
sudo su -
```

### 1-2. Apache（httpd）をインストール
```bash
dnf update -y
dnf install -y httpd
```

### 1-3. 自動起動をONにしつつ起動（enable + start を一発）
```bash
systemctl enable --now httpd
```

### 1-4. 状態確認
```bash
systemctl status httpd
```

---

## 2. 起動確認（通信できるか）

### 2-1. 80番ポートで待ち受けしているか確認
```bash
ss -lntp | grep :80
```

### 2-2. サーバ内部から HTTP 応答が返るか確認
```bash
curl -I http://localhost/
```

> `HTTP/1.1 200 OK` や `HTTP/1.1 403 Forbidden` などが返ってくれば、Apacheが動いています。

---

## 3. HTMLを置いて表示する（最初の表示確認）

### 3-1. index.html を作る（デフォルトの公開フォルダ）
Apacheのデフォルト公開フォルダは `/var/www/html` です。

```bash
cat > /var/www/html/index.html << 'EOF'
<!doctype html>
<html>
<head><meta charset="utf-8"><title>Apache Test</title></head>
<body>
  <h1>Apache is working!</h1>
  <p>It works!</p>
</body>
</html>
EOF
```

### 3-2. ローカル確認
```bash
curl http://localhost/
```

### 3-3. ブラウザ確認（自分のPCから）
ブラウザで以下にアクセスします：

- `http://<サーバのパブリックIP>/`

---

## 4. ログの見方（重要）

### アクセスログ
```bash
tail -f /var/log/httpd/access_log
```

### エラーログ
```bash
tail -f /var/log/httpd/error_log
```

> 何か表示されない／403／404 のときは **error_log** を見るのが最短です。

---

## 5. Apache設定ファイル（ディレクティブ）

### 設定ファイルの場所
- メイン設定：`/etc/httpd/conf/httpd.conf`
- 追加設定：`/etc/httpd/conf.d/*.conf`

---

## 6. DocumentRoot（公開フォルダ）を変更する

ここでは例として、公開フォルダを `/home/saba` に変更します。

### 6-1. バックアップを取る
```bash
cp -a /etc/httpd/conf/httpd.conf /etc/httpd/conf/httpd.conf.backup.$(date +%Y%m%d-%H%M%S)
ls /etc/httpd/conf | grep httpd.conf
```

### 6-2. 公開フォルダを作成
```bash
mkdir -p /home/saba
```

### 6-3. 権限を設定（Apache が読めるようにする）
```bash
chmod 711 /home
chmod 755 /home/saba
```

### 6-4. DocumentRoot と Directory 設定を修正
```bash
vi /etc/httpd/conf/httpd.conf
```

#### 変更内容（例）
```apache
DocumentRoot "/home/saba"

<Directory "/home/saba">
    AllowOverride None
    Require all granted
</Directory>
```

> ✅ DocumentRoot を変えるだけではダメで、必ず `<Directory>` の許可設定が必要です。

### 6-5. 公開フォルダ側に index.html を作成
```bash
cat > /home/saba/index.html << 'EOF'
<!doctype html>
<html>
<head><meta charset="utf-8"><title>New DocumentRoot</title></head>
<body>
  <h1>DocumentRoot is /home/saba</h1>
</body>
</html>
EOF
```

### 6-6. 設定の文法チェック（必須）
```bash
apachectl configtest
```

`Syntax OK` が出ればOK。

### 6-7. 反映（再起動）
```bash
systemctl restart httpd
```

### 6-8. 動作確認
```bash
curl http://localhost/
```

---

## 7. リダイレクト設定（301）

例：`/sa-mon.html` にアクセスしたら、別URLへ転送する

### 7-1. 設定ファイルを追加作成（conf.d に分離するのがおすすめ）
```bash
vi /etc/httpd/conf.d/redirect.conf
```

### 7-2. 設定内容（例）
```apache
Redirect 301 /sa-mon.html http://<リダイレクト先のPublicIP>/aburi.html
```
※IPアドレスは、ユーザーの到達可能なIPである必要がある→PublicIP

### 7-3. 文法チェック → 反映
```bash
apachectl configtest
systemctl restart httpd
```
### 7-4. ブラウザで確認
http://webサーバのPublicIP/sa-mon.html

---

## 8. よくあるトラブルと確認ポイント

### 8-1. ブラウザで開けない（タイムアウト）
- セキュリティグループのHTTP（80）が開いているか
- サーバにパブリックIPがあるか
- `ss -lntp | grep :80` で待ち受けしているか

### 8-2. 403 Forbidden になる
- DocumentRoot を `/home` にしている場合、権限不足が多い  
  - `chmod 711 /home`
  - `chmod 755 /home/saba`
- `<Directory "..."> Require all granted` があるか
- エラーログ確認：  
  ```bash
  tail -n 50 /var/log/httpd/error_log
  ```

### 8-3. 設定変更したのに反映されない
- `apachectl configtest` を通しているか
- `systemctl restart httpd` を実行したか

---

## 9. 付録：便利な確認コマンド

### Apacheの設定内で DocumentRoot を探す
```bash
grep -n "DocumentRoot" /etc/httpd/conf/httpd.conf
```

### Apacheが読み込む設定ファイル一覧を確認
```bash
httpd -t -D DUMP_INCLUDES
```

---
