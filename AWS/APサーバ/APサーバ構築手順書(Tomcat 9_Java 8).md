# APサーバ構築手順書（Tomcat 9 + Java 8 / Amazon Linux 2023）

## 0. 前提
- OS：Amazon Linux 2023
- Java：Amazon Corretto 8（dnf）
- Tomcat：9.0.x（tar.gz）
- 前段：Apache httpd（Reverse Proxy）
- ポート方針：外部公開は原則 **80のみ**（8080は原則閉じる）

参考：
- Tomcat 9 EOL（予定）：2027-03-31  
  https://tomcat.apache.org/tomcat-9.0.x-eos.html
- Tomcat 9 セットアップ（要件）：Java 8 以上  
  https://tomcat.apache.org/tomcat-9.0-doc/setup.html
- Corretto 8（Amazon Linux へのインストール）  
  https://docs.aws.amazon.com/corretto/latest/corretto-8-ug/amazon-linux-install.html
- Tomcat 9 ダウンロード（最新版確認）  
  https://tomcat.apache.org/download-90.cgi

---

## 1. サーバにログインしてrootに切替
```bash
sudo su -
cd /home/ec2-user
```

---

## 2. JDK（Amazon Corretto 8）インストール

### 2-1. インストール
```bash
dnf install -y java-1.8.0-amazon-corretto java-1.8.0-amazon-corretto-devel
```

### 2-2. バージョン確認
```bash
java -version
javac -version
```

### 2-3. JAVA_HOME の確認（後で使う）
```bash
readlink -f "$(which java)"
# 例：/usr/lib/jvm/java-1.8.0-amazon-corretto/...
```

---

## 3. Tomcat 9 インストール

### 3-1. tomcatユーザ作成
```bash
useradd -r -m -U -s /sbin/nologin tomcat
```
>コマンドの意味
useradd：ユーザーを新しく作成するコマンド
-r：システムユーザーとして作る（通常のログイン用ユーザーではない）
→ サービス（Tomcatなど）を動かす用に使うことが多い
-m：ホームディレクトリを作成する
→ /home/tomcat みたいなのが作られる
-U：同名のグループも一緒に作る
→ tomcat ユーザー + tomcat グループが作られる
-s /sbin/nologin：ログインできないシェルを指定する
→ このユーザーでSSHログインさせない（セキュリティ的に安全）
tomcat：作成するユーザー名


### 3-2. Tomcatダウンロード（最新版を選ぶ）
Tomcat 9 の **最新版 9.0.x** を公式から確認し、tar.gz を取得する。

```bash
cd /tmp

# 例：ダウンロードするバージョンを指定（都度、最新版に置換する）
TOMCAT_VER=9.0.115
echo $TOMCAT_VER
(出力例) 9.0.115

wget https://downloads.apache.org/tomcat/tomcat-9/v${TOMCAT_VER}/bin/apache-tomcat-${TOMCAT_VER}.tar.gz
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
*何してる？*：「Tomcatを “毎回同じ条件で・安全に” 動かすための設定」

```bash
vi /usr/local/tomcat/bin/setenv.sh
```

```sh
#!/bin/sh
export CATALINA_HOME=/usr/local/tomcat
export JAVA_HOME=/opt/amazon-corretto-8.482.08.1-linux-x64
export JAVA_OPTS="-Xms256m -Xmx1024m -Dfile.encoding=UTF-8"
```
>`export CATALINA_HOME=/usr/local/tomcat`
　= Tomcatのインストール先を指定
　　catalina.sh などが「Tomcat本体どこ？」を判断しやすくなる
`export JAVA_HOME=/opt/amazon-corretto-8.482.08.1-linux-x64`
　= Javaのインストール先を指定
　　TomcatはJavaで動くので、ここがズレると起動失敗することがある
`export JAVA_OPTS="..."`
　= Tomcat（正確にはJVM）起動時のオプション設定。
`-Xms256m`
　= 初期メモリを256MBにする
`-Xmx1024m`
　= 最大メモリを1024MB（1GB）にする
`-Dfile.encoding=UTF-8`
　= 文字コードをUTF-8に固定（文字化け防止に役立つ）


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

>この設定で👇を無効化してる：
`・WARを勝手に展開しない`
`・配置したアプリを勝手に検知して反映しない`
`・起動時に勝手にデプロイしない（必要なら）`
何のために？？
`「Tomcatが勝手にアプリを載せたり更新したりしないようにして、手動で確実に管理するための設定」`

---

## 5. systemd（Tomcat自動起動）

### 5-1. unit作成
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
# サービス開始時に実行するコマンド
ExecStop=/usr/local/tomcat/bin/catalina.sh stop
# サービス停止時に実行するコマンド
# 👆systemd が裏で catalina.sh を呼んでくれるイメージ！

SuccessExitStatus=143
TimeoutStartSec=1
TimeoutStopSec=120

# SuccessExitStatus=143 : Tomcat停止時に 143（SIGTERM由来）で終わっても正常扱いにする → これがないと「失敗」と誤判定されることがある
# TimeoutStartSec / TimeoutStopSec：起動・停止の待ち時間（秒）→ Tomcatは少し時間かかることあるので余裕を持たせる

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

## 7. Javaアプリ（knowledge）の配備（自動デプロイ無効前提）

`autoDeploy=false` のため、**Context定義で明示的に配備**する。

### 7-1. 配備先を作成して展開
```bash
mkdir -p /usr/local/tomcat/apps/knowledge
cd /usr/local/tomcat/apps/knowledge

wget https://github.com/support-project/knowledge/releases/download/v1.13.1/knowledge.war
jar xf knowledge.war
rm -f knowledge.war

chown -R tomcat:tomcat /usr/local/tomcat/apps/knowledge
```

### 7-2. Context定義（/knowledge）
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

## 8. 動作確認

### 8-1. Tomcat直（原則ローカルで確認）
```bash
curl -I http://127.0.0.1:8080/knowledge/
```

### 8-2. Apache経由（外部公開）
```bash
curl -I http://127.0.0.1/knowledge/
```

ブラウザ確認：
- `http://グローバルIP/knowledge` で表示されること

---

## 9. セキュリティグループ推奨
- inbound
  - 22/tcp：自分のIPのみ
  - 80/tcp：公開範囲に合わせる
  - 8080/tcp：**原則閉じる**（開けるなら社内IP/踏み台のみ）
- outbound：通常 all open（要件次第）

---

## 10. 運用メモ（重要）
- Tomcat 9 は 2027-03-31 にEOL予定。長期運用なら移行（Tomcat 10.1 + Java 17 など）計画を検討する。
- Tomcat は脆弱性対応のため、**バージョン固定を避け**、公式の最新版を追従する運用が安全。
