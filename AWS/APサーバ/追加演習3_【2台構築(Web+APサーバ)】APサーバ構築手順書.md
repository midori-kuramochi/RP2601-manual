# 追加演習3 手順書（Web/AP分離・Apache版）
対象：Amazon Linux 2023 / Tomcat 9 + Java8 / Apache Reverse Proxy

## 0. 前提（構成）
- **Webサーバ（WEB01）**
  - Apache(httpd) をインストールし **リバースプロキシ**として動作
  - 外部公開：**80のみ**
- **APサーバ（AP01）**
  - Java（Corretto 8） + Tomcat 9 をインストールし、アプリを動作
  - 外部公開：原則なし（Tomcat 8080 は **Webサーバからのみ**許可）

### 0-1. ネットワーク前提（例）
- WEB01（Public IPあり / Private IP：`10.0.1.10`）
- AP01（Private IPのみ / Private IP：`10.0.2.10`）
- ※IPはあなたの環境に置き換えてOK

### 0-2. セキュリティグループ（超重要）
**WEB01（Web）SG**
- inbound：80/tcp（必要範囲）、22/tcp（自分のIP）
- outbound：all（またはAP01の8080へ出られればOK）

**AP01（AP）SG**
- inbound：
  - 8080/tcp：**WEB01のSG**（またはWEB01のPrivate IP）からのみ許可
  - 22/tcp：踏み台/自分のIP（構成に合わせる）
- outbound：all

> ✅この演習の核：**Tomcat(8080)をインターネットに出さず、Webサーバだけが叩ける状態**にする。

---

# 1. APサーバ（AP01）構築（Tomcat 9 + Java8）

## 1-1. ログインしてrootへ
```bash
sudo su -
cd /home/ec2-user
```

## 1-2. JDK（Amazon Corretto 8）インストール
```bash
dnf install -y java-1.8.0-amazon-corretto java-1.8.0-amazon-corretto-devel
java -version
javac -version
```

## 1-3. Tomcat インストール

### tomcatユーザ作成
```bash
useradd -r -m -U -s /sbin/nologin tomcat
```

### Tomcatダウンロード＆配置（例）
```bash
cd /tmp
TOMCAT_VER=9.0.115
wget https://downloads.apache.org/tomcat/tomcat-9/v${TOMCAT_VER}/bin/apache-tomcat-${TOMCAT_VER}.tar.gz

mkdir -p /usr/local
tar zxf apache-tomcat-${TOMCAT_VER}.tar.gz -C /usr/local
cd /usr/local
ln -s apache-tomcat-${TOMCAT_VER} tomcat

chown -R tomcat:tomcat /usr/local/apache-tomcat-${TOMCAT_VER}
chown -h tomcat:tomcat /usr/local/tomcat
```

## 1-4. setenv.sh 作成（Java/メモリ）
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

## 1-5. server.xml 設定（自動デプロイ無効 + 8080待受の考え方）
```bash
vi /usr/local/tomcat/conf/server.xml
```

### (1) 自動デプロイ無効化
```xml
<Host name="localhost" appBase="webapps"
      unpackWARs="false" autoDeploy="false" deployOnStartup="false">
```

### (2) 8080の待受について（分離構成のポイント）
- **分離構成では WebサーバからAPサーバへ届く必要があるため** `address="127.0.0.1"` は付けない（外す）

TomcatのHTTP Connector例（address指定なし）：
```xml
<Connector port="8080" protocol="HTTP/1.1"
           connectionTimeout="20000"
           redirectPort="8443" />
```

> ✅セキュリティは **SGでWEB01からのみ許可**して守る（Tomcatを閉じるのはネットワーク側）

## 1-6. systemd 設定（自動起動）
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

```bash
systemctl daemon-reload
systemctl enable --now tomcat
systemctl status tomcat --no-pager
ss -lntp | grep 8080
```

## 1-7. アプリ配備（例：knowledge）
```bash
mkdir -p /usr/local/tomcat/apps/knowledge
cd /usr/local/tomcat/apps/knowledge
wget https://github.com/support-project/knowledge/releases/download/v1.13.1/knowledge.war
jar xf knowledge.war
rm -f knowledge.war
chown -R tomcat:tomcat /usr/local/tomcat/apps/knowledge
```

Context定義：
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

## 1-8. APサーバ単体で動作確認（ローカル）
```bash
curl -I http://127.0.0.1:8080/knowledge/
```

---

# 2. Webサーバ（WEB01）構築（Apache Reverse Proxy）

## 2-1. ログインしてrootへ
```bash
sudo su -
cd /home/ec2-user
```

## 2-2. Apache(httpd) インストール
```bash
dnf install -y httpd
systemctl enable --now httpd
systemctl status httpd --no-pager
```

## 2-3. 必要モジュール確認（proxy）
```bash
httpd -M | egrep 'proxy_module|proxy_http_module' || true
```

## 2-4. リバースプロキシ設定
```bash
vi /etc/httpd/conf.d/tomcat-proxy.conf
```

※APサーバのプライベートIPに置換（例：`10.0.2.10`）

```apache
ProxyRequests Off
ProxyPreserveHost On

ProxyPass        /knowledge  http://10.0.2.10:8080/knowledge
ProxyPassReverse /knowledge  http://10.0.2.10:8080/knowledge
```

反映：
```bash
httpd -t
systemctl restart httpd
```

## 2-5. WebサーバからAPサーバへの疎通確認（超大事）
```bash
curl -I http://10.0.2.10:8080/knowledge/
```

---

# 3. 全体の動作確認（ブラウザ/外部）
外部クライアントから：
- `http://WEB01のグローバルIP/knowledge` にアクセスして表示されること

---

# 4. 仕上げチェック（OK判定）

## 4-1. AP01（Tomcat）
```bash
systemctl status tomcat --no-pager
ss -lntp | grep 8080
curl -I http://127.0.0.1:8080/knowledge/
```

## 4-2. WEB01（Apache）
```bash
systemctl status httpd --no-pager
httpd -t
curl -I http://10.0.2.10:8080/knowledge/
curl -I http://127.0.0.1/knowledge
```

---

# 5. よくあるトラブル

## 502/503が出る（Apache→Tomcat）
- WEB01→AP01の疎通：
  - `curl -I http://10.0.2.10:8080/knowledge/`
- AP01側：
  - `journalctl -u tomcat -n 200 --no-pager`
  - `ss -lntp | grep 8080`
- SG確認：
  - AP01の8080が **WEB01からだけ**許可になっているか

## 404が出る
- TomcatのContext定義（knowledge.xml）とパスが合ってるか
- `autoDeploy=false` にしたなら **Context定義が必須**
