# APサーバ構築手順書（Tomcat 9 + Java 8 / Amazon Linux 2023）※AJP連携版

## 0. 前提
- OS：Amazon Linux 2023
- Java：Amazon Corretto 8（dnf）
- Tomcat：9.0.x（tar.gz）
- 前段：Apache httpd（Reverse Proxy）
- 連携方式：Apache → Tomcat を **AJP（8009）**で連携
- ポート方針：
  - 外部公開：原則 **80のみ**
  - Tomcat HTTP(8080)：原則 **外部公開しない**（必要なら localhost 限定）
  - Tomcat AJP(8009)：**localhost 限定 + secret必須**

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

### 3-5. server.xml を設定する（自動デプロイ無効 + AJP有効化）

#### (1) 自動デプロイ無効化
```bash
vi /usr/local/tomcat/conf/server.xml
```

`<Host ...>` を探して、`unpackWARs` と `autoDeploy` を false（必要なら `deployOnStartup` もfalse）

```xml
<Host name="localhost" appBase="webapps"
      unpackWARs="false" autoDeploy="false" deployOnStartup="false">
```

#### (2) AJP Connector を有効化（重要）
**AJP用の secret（共有鍵）**を決める（例：強いランダム文字列）

`<Service name="Catalina">` の中に以下を追加（または既存を修正）：

```xml
<Connector protocol="AJP/1.3"
           address="127.0.0.1"
           port="8009"
           redirectPort="8443"
           secretRequired="true"
           secret="change_me_to_strong_secret" />
```

✅ポイント  
- `address="127.0.0.1"`：AJPをローカルに閉じる  
- `secretRequired="true"` + `secret="..."`：AJPの不正利用対策

#### (3) （推奨）Tomcat HTTP(8080) も外部に出さない
8080を完全に使わないなら Connector ごと削除/無効化でもOK。  
切り分け目的で残す場合は **localhost に縛る**のがおすすめ：

```xml
<Connector port="8080" protocol="HTTP/1.1"
           address="127.0.0.1"
           connectionTimeout="20000"
           redirectPort="8443" />
```

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
ss -lntp | egrep '(:8009|:8080)'
```

---

## 4. Apacheのインストール（AJPでTomcatへ転送）

### 4-1. dnfでインストール
```bash
dnf install -y httpd
```

### 4-2. 必要モジュール確認（proxy / proxy_ajp）
```bash
httpd -M | egrep 'proxy_module|proxy_ajp_module' || true
```

### 4-3. AJPプロキシ設定を追記（conf.d に分離推奨）
```bash
vi /etc/httpd/conf.d/tomcat-ajp.conf
```

以下を追記（**AJP secret を Tomcat 側と一致**させる）

```apache
ProxyRequests Off

# /knowledge を AJP(8009) へ転送
ProxyPass        /knowledge  ajp://127.0.0.1:8009/knowledge secret=change_me_to_strong_secret
ProxyPassReverse /knowledge  ajp://127.0.0.1:8009/knowledge
```

⚠️注意：`secret=` が使えない/エラーになる場合  
環境差で `secret=` パラメータが効かないケースがあります。  
その場合の対処はどちらか：
1) **mod_jk** を採用して secret 連携する（AJP専用で確実）  
2) Apache/httpd を要件満たすバージョンにする  
（**Tomcat側の secretRequired を false にするのは非推奨**）

文法チェック：
```bash
httpd -t
```

### 4-4. 自動有効化と起動
```bash
systemctl enable --now httpd
systemctl status httpd --no-pager
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

```bash
chown -R tomcat:tomcat /usr/local/tomcat/conf/Catalina
systemctl restart tomcat
```

---

## 6. 動作確認（AJP経由）

### 6-1. Apache経由（外部公開）
```bash
curl -I http://127.0.0.1/knowledge/
```

ブラウザ確認：
- `http://グローバルIP/knowledge` で表示されること

### 6-2. （任意）Tomcat直の疎通（localhostのみ）
Tomcat HTTP(8080)を localhost で残している場合：
```bash
curl -I http://127.0.0.1:8080/knowledge/
```

---

## 7. セキュリティ（必須）
- inbound（推奨）
  - 22/tcp：自分のIPのみ
  - 80/tcp：公開範囲
  - **8080/tcp：閉じる**（開けない）
  - **8009/tcp：閉じる**（開けない）
- Tomcat側は `address="127.0.0.1"` にしているため、SGを誤って開けても外部から到達しにくい（ただしSGは閉じるのが原則）

---

## 8. よくあるハマりどころ（最初に見る場所）
- Apache → Tomcat が 503/502
  - `journalctl -u httpd -n 200 --no-pager`
  - `journalctl -u tomcat -n 200 --no-pager`
  - `ss -lntp | egrep '(:8009|:8080)'`
- AJP secret 不一致
  - Tomcat `server.xml` の `secret=...`
  - Apache `ProxyPass ... secret=...` の一致確認
