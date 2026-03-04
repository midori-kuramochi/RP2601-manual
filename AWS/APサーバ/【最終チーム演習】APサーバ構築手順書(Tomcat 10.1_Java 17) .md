# APサーバ構築手順書（Tomcat 10.1 + Java 17 / Amazon Linux 2023）

## 0. 前提
- OS：Amazon Linux 2023
- Java：Amazon Corretto 17（dnf）
- Tomcat：10.1.x（tar.gz）
- 前段：Apache httpd（Reverse Proxy）
- ポート方針：外部公開は原則 **80のみ**（8080は原則閉じる）
- 注意：Tomcat 10.1 は **Jakarta EE 10（`jakarta.*`）** 系です。  
  旧来の `javax.*` 前提アプリはそのままでは動かない場合があります（アプリ側の対応が必要）。

参考：
- Tomcat 10.1 Migration Guide  
  https://tomcat.apache.org/migration-10.1.html
- Tomcat 10.1 ダウンロード（最新版確認）  
  https://tomcat.apache.org/download-10.cgi
- Corretto 17（Amazon Linux へのインストール）  
  https://docs.aws.amazon.com/corretto/latest/corretto-17-ug/amazon-linux-install.html

---

## 1. サーバにログインしてrootに切替
```bash
sudo su -
cd /home/ec2-user
```

---

## 2. JDK（Amazon Corretto 17）インストール

### 2-1. インストール
```bash
dnf install -y java-17-amazon-corretto java-17-amazon-corretto-devel
```

### 2-2. バージョン確認
```bash
java -version
javac -version
```

### 2-3. JAVA_HOME の確認（後で使う）
```bash
readlink -f "$(which java)"
# 例：/usr/lib/jvm/java-17-amazon-corretto/...
```

---

## 3. Tomcat 10.1 インストール

### 3-1. tomcatユーザ作成
```bash
useradd -r -m -U -s /sbin/nologin tomcat
```

### 3-2. Tomcatダウンロード（最新版を選ぶ）
Tomcat 10.1 の **最新版 10.1.x** を公式から確認し、tar.gz を取得する。

```bash
cd /tmp

# 例：ダウンロードするバージョンを指定（都度、最新版に置換する）
TOMCAT_VER=10.1.27

wget https://downloads.apache.org/tomcat/tomcat-10/v${TOMCAT_VER}/bin/apache-tomcat-${TOMCAT_VER}.tar.gz
ls -l apache-tomcat-${TOMCAT_VER}.tar.gz
```

### 3-3. 配置（/usr/local）
```bash
mkdir -p /usr/local
tar zxf apache-tomcat-${TOMCAT_VER}.tar.gz -C /usr/local

cd /usr/local
ln -s apache-tomcat-${TOMCAT_VER} tomcat

chown -R tomcat:tomcat /usr/local/apache-tomcat-${TOMCAT_VER}
chown -h tomcat:tomcat /usr/local/tomcat
```

---

## 4. Tomcat設定

### 4-1. setenv.sh 作成（JAVA_HOME / メモリ）
```bash
vi /usr/local/tomcat/bin/setenv.sh
```

```sh
#!/bin/sh
export CATALINA_HOME=/usr/local/tomcat
export JAVA_HOME=/usr/lib/jvm/java-17-amazon-corretto
export JAVA_OPTS="-Xms256m -Xmx1024m -Dfile.encoding=UTF-8"
```

```bash
chmod 755 /usr/local/tomcat/bin/setenv.sh
chown tomcat:tomcat /usr/local/tomcat/bin/setenv.sh
```

### 4-2. server.xml：自動デプロイ無効化
要件「自動デプロイ無効」に合わせて `unpackWARs` と `autoDeploy` を false にする  
（必要に応じて `deployOnStartup` も false）

```bash
vi /usr/local/tomcat/conf/server.xml
```

`<Host ...>` を探して、以下のように設定：

```xml
<Host name="localhost" appBase="webapps"
      unpackWARs="false" autoDeploy="false" deployOnStartup="false">
```

---

## 5. systemd（Tomcat自動起動）

### 5-1. unit作成
```bash
vi /etc/systemd/system/tomcat.service
```

```ini
[Unit]
Description=Apache Tomcat 10.1
After=network.target

[Service]
Type=forking
User=tomcat
Group=tomcat

Environment=CATALINA_HOME=/usr/local/tomcat
Environment=JAVA_HOME=/usr/lib/jvm/java-17-amazon-corretto
Environment=CATALINA_PID=/usr/local/tomcat/temp/tomcat.pid

ExecStart=/usr/local/tomcat/bin/catalina.sh start
ExecStop=/usr/local/tomcat/bin/catalina.sh stop

SuccessExitStatus=143
TimeoutStartSec=120
TimeoutStopSec=120

[Install]
WantedBy=multi-user.target
```

### 5-2. 反映＆起動
```bash
systemctl daemon-reload
systemctl enable --now tomcat
systemctl status tomcat --no-pager
ss -lntp | grep 8080
```

---

## 6. Apache（httpd）インストール＆Reverse Proxy

### 6-1. httpdインストール
```bash
dnf install -y httpd
systemctl enable --now httpd
systemctl status httpd --no-pager
```

### 6-2. Proxy設定（conf.d に分離）
`/etc/httpd/conf.d/` 配下に設定を分離し、運用しやすくする。

```bash
vi /etc/httpd/conf.d/tomcat-proxy.conf
```

例：`/knowledge` を Tomcat（8080）へ転送

```apache
ProxyRequests Off

ProxyPass        /knowledge  http://127.0.0.1:8080/knowledge
ProxyPassReverse /knowledge  http://127.0.0.1:8080/knowledge
```

反映：
```bash
httpd -t
systemctl restart httpd
```

---

## 7. アプリ配備（自動デプロイ無効前提）

`autoDeploy=false` のため、**Context定義で明示的に配備**する。

### 7-1. 配備先を作成して展開（例）
```bash
mkdir -p /usr/local/tomcat/apps/myapp
cd /usr/local/tomcat/apps/myapp

# 例：アプリのwarを配置して展開（URLやファイル名は環境に合わせる）
# wget https://example.com/myapp.war
# jar xf myapp.war
# rm -f myapp.war

chown -R tomcat:tomcat /usr/local/tomcat/apps/myapp
```

### 7-2. Context定義（例：/myapp）
```bash
mkdir -p /usr/local/tomcat/conf/Catalina/localhost
vi /usr/local/tomcat/conf/Catalina/localhost/myapp.xml
```

```xml
<Context path="/myapp" docBase="/usr/local/tomcat/apps/myapp" reloadable="false" />
```

反映：
```bash
chown -R tomcat:tomcat /usr/local/tomcat/conf/Catalina
systemctl restart tomcat
```

---

## 8. 動作確認

### 8-1. Tomcat直（原則ローカルで確認）
```bash
curl -I http://127.0.0.1:8080/myapp/
```

### 8-2. Apache経由（外部公開）
```bash
curl -I http://127.0.0.1/myapp/
```

ブラウザ確認：
- `http://グローバルIP/myapp` で表示されること

---

## 9. セキュリティグループ推奨
- inbound
  - 22/tcp：自分のIPのみ
  - 80/tcp：公開範囲に合わせる
  - 8080/tcp：**原則閉じる**（開けるなら社内IP/踏み台のみ）
- outbound：通常 all open（要件次第）

---

## 10. 運用メモ（重要）
- Tomcat 10.1 は Jakarta EE 10（`jakarta.*`）です。`javax.*` のアプリはそのまま動かない場合があるため、アプリの対応状況を確認する。
- Tomcat は脆弱性対応のため、**バージョン固定を避け**、公式の最新版を追従する運用が安全。
