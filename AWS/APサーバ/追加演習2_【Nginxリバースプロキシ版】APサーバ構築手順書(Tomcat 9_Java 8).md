# APサーバ構築手順書（Tomcat 9 + Java 8 / Amazon Linux 2023）※Nginxリバースプロキシ版

## 0. 前提
- OS：Amazon Linux 2023
- Java：Amazon Corretto 8（dnf）
- Tomcat：9.0.x（tar.gz）
- 前段：**Nginx（Reverse Proxy）**
- 連携方式：Nginx → Tomcat は **HTTP（127.0.0.1:8080）**
  - ※Nginxは標準ではAJP連携しないため、本手順はHTTP連携
- ポート方針：
  - 外部公開：原則 **80のみ**
  - Tomcat HTTP(8080)：**外部公開しない（localhost限定）**

---

## 1. サーバにログインしてrootにスイッチする
```bash
sudo su -
cd /home/ec2-user
```

---

## 2. JDKインストール（Corretto 8）

### 2-1. インストール
```bash
dnf install -y java-1.8.0-amazon-corretto java-1.8.0-amazon-corretto-devel
```

### 2-2. 確認
```bash
java -version
javac -version
```

---

## 3. Tomcatインストール

### 3-1. tomcatユーザを追加する
```bash
useradd -r -m -U -s /sbin/nologin tomcat
```

### 3-2. Tomcatをダウンロードする（9.0.x）
（例：バージョンは適宜最新版に置換）
```bash
cd /tmp
TOMCAT_VER=9.0.115
wget https://downloads.apache.org/tomcat/tomcat-9/v${TOMCAT_VER}/bin/apache-tomcat-${TOMCAT_VER}.tar.gz
ls -l apache-tomcat-${TOMCAT_VER}.tar.gz
```

### 3-3. 解凍して配置する
```bash
mkdir -p /usr/local
tar zxf apache-tomcat-${TOMCAT_VER}.tar.gz -C /usr/local

cd /usr/local
ln -s apache-tomcat-${TOMCAT_VER} tomcat
chown -R tomcat:tomcat /usr/local/apache-tomcat-${TOMCAT_VER}
chown -h tomcat:tomcat /usr/local/tomcat
```

### 3-4. setenv を作成する（JAVA_HOME / メモリ）
```bash
vi /usr/local/tomcat/bin/setenv.sh
```

```sh
#!/bin/sh
export CATALINA_HOME=/usr/local/tomcat
export JAVA_HOME=/usr/lib/jvm/java-1.8.0-amazon-corretto
export JAVA_OPTS="-Xms256m -Xmx1024m -Dfile.encoding=UTF-8"
```

```bash
chmod 755 /usr/local/tomcat/bin/setenv.sh
chown tomcat:tomcat /usr/local/tomcat/bin/setenv.sh
```

### 3-5. server.xml を設定する（自動デプロイ無効 + 8080をlocalhost限定）
```bash
vi /usr/local/tomcat/conf/server.xml
```

#### (1) 自動デプロイ無効化
`<Host ...>` を探して、`unpackWARs` と `autoDeploy` を false（必要なら `deployOnStartup` もfalse）

```xml
<Host name="localhost" appBase="webapps"
      unpackWARs="false" autoDeploy="false" deployOnStartup="false">
```

#### (2) Tomcat HTTP(8080) を localhost に限定（重要）
Tomcatの8080を外部公開しないため、HTTP Connector に `address="127.0.0.1"` を付ける。

```xml
<Connector port="8080" protocol="HTTP/1.1"
           address="127.0.0.1"
           connectionTimeout="20000"
           redirectPort="8443" />
```

> ✅ポイント  
> - これで **同一サーバ内のNginxだけ**が8080へ接続可能になります（外部からは到達しにくくなる）

---

### 3-6. 自動起動のスクリプト（systemd）を作成する
```bash
vi /etc/systemd/system/tomcat.service
```

```ini
[Unit]
Description=Apache Tomcat 9
After=network.target

[Service]
Type=forking
User=tomcat
Group=tomcat

Environment=CATALINA_HOME=/usr/local/tomcat
Environment=JAVA_HOME=/usr/lib/jvm/java-1.8.0-amazon-corretto
Environment=CATALINA_PID=/usr/local/tomcat/temp/tomcat.pid

ExecStart=/usr/local/tomcat/bin/catalina.sh start
ExecStop=/usr/local/tomcat/bin/catalina.sh stop

SuccessExitStatus=143
TimeoutStartSec=120
TimeoutStopSec=120

[Install]
WantedBy=multi-user.target
```

反映・起動：
```bash
systemctl daemon-reload
systemctl enable --now tomcat
systemctl status tomcat --no-pager
```

待ち受け確認：
```bash
ss -lntp | egrep '(:8080)'
# 127.0.0.1:8080 になっていること
```

---

## 4. Nginxのインストール（TomcatへHTTPで転送）

### 4-1. dnfでインストール
```bash
dnf install -y nginx
```

### 4-2. リバースプロキシ設定（/etc/nginx/conf.d/ に分離）
```bash
vi /etc/nginx/conf.d/tomcat-proxy.conf
```

例：`/knowledge` を Tomcat（127.0.0.1:8080）へ転送

```nginx
server {
    listen 80;
    server_name _;

    # /knowledge を Tomcat へ
    location /knowledge/ {
        proxy_pass http://127.0.0.1:8080/knowledge/;

        # 逆プロキシでよく使うヘッダ
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

> ✅ポイント  
> - `location /knowledge/` と `proxy_pass http://.../knowledge/` の末尾 `/` を揃えると、パス変換でハマりにくいです  
> - アプリが `/knowledge`（末尾スラ無し）に来る場合もあるので、必要なら次のリダイレクトを追加します：
>   - `location = /knowledge { return 301 /knowledge/; }`

### 4-3. 設定テスト＆起動
```bash
nginx -t
systemctl enable --now nginx
systemctl status nginx --no-pager
```

---

## 5. javaアプリの配備（自動デプロイ無効のまま）

### 5-1. 配備場所のディレクトリ作成
```bash
mkdir -p /usr/local/tomcat/apps/knowledge
cd /usr/local/tomcat/apps/knowledge
```

### 5-2. アプリケーションのダウンロード
```bash
wget https://github.com/support-project/knowledge/releases/download/v1.13.1/knowledge.war
```

### 5-3. 解凍・不要ファイル削除・権限
```bash
jar xf knowledge.war
rm -f knowledge.war
chown -R tomcat:tomcat /usr/local/tomcat/apps/knowledge
```

### 5-4. Context定義（自動デプロイ無効のため明示配備）
```bash
mkdir -p /usr/local/tomcat/conf/Catalina/localhost
vi /usr/local/tomcat/conf/Catalina/localhost/knowledge.xml
```

```xml
<Context path="/knowledge" docBase="/usr/local/tomcat/apps/knowledge" reloadable="false" />
```

反映：
```bash
chown -R tomcat:tomcat /usr/local/tomcat/conf/Catalina
systemctl restart tomcat
```

---

## 6. 動作確認（Nginx経由）

### 6-1. Nginx経由（外部公開）
```bash
curl -I http://127.0.0.1/knowledge/
```

ブラウザ確認：
- `http://グローバルIP/knowledge/` で表示されること

### 6-2. （任意）Tomcat直の疎通（localhostのみ）
```bash
curl -I http://127.0.0.1:8080/knowledge/
```

---

## 7. セキュリティ（必須）
- inbound（推奨）
  - 22/tcp：自分のIPのみ
  - 80/tcp：公開範囲
  - **8080/tcp：閉じる**（開けない）
- Tomcat側 `address="127.0.0.1"` により、外部から8080へ到達しない構成になる（ただしSGも閉じる）

---

## 8. よくあるハマりどころ（最初に見る場所）
- 502 Bad Gateway（Nginx → Tomcatへ届かない）
  - `ss -lntp | grep 8080`（127.0.0.1:8080 で待ち受けているか）
  - `journalctl -u tomcat -n 200 --no-pager`
  - `journalctl -u nginx -n 200 --no-pager`
- パスが二重になる/崩れる
  - `location` と `proxy_pass` の末尾 `/` を揃える
- `/knowledge`（末尾なし）で404
  - `location = /knowledge { return 301 /knowledge/; }` を追加して揃える
