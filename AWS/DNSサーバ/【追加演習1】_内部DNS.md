# 追加演習1 手順書  
**APはTomcatだけ／WebがApache Reverse Proxy（Web+DNS同居）**  
対象OS：Amazon Linux 2023  
目的：外部は **80のみ公開**、内部で Apache → Tomcat(8080) へリバースプロキシし、**サーバ間は `好きな言葉.local` で名前解決**する。

---

## 0. 要件（チェック）
- [ ] WebサーバーとDNSサーバーは同じサーバー内に構築する（Web+DNS同居）
- [ ] 外部から名前解決して `http://サブドメイン` でアクセス
- [ ] ApacheをReverse Proxyとして Private subnetのAPサーバへ接続し、Tomcatページ表示
- [ ] サーバ間通信はローカルドメインで名前解決
- [ ] ローカルドメインは「好きな言葉.local」
- [ ] Webサーバーは外部へ接続可能（dnf update等）

---

## 1. 事前情報を決める（置き換え用）
以降、`<>` をあなたの値に置き換えて使ってください。

- 外部公開用サブドメイン：`<SUBDOMAIN>.entrycl.net`  
  例：`kuramochi.entrycl.net`
- Web+DNSサーバ（Public subnet）
  - Private IP：`<WEB_PRIV_IP>`
  - Public IP/EIP：`<WEB_PUB_IP>`
- APサーバ（Private subnet）
  - Private IP：`<AP_PRIV_IP>`
- ローカルドメイン（好きな言葉）：`<LOCALWORD>.local`  
  例：`yakiniku.local`
- 内部ホスト名（ローカルDNS）
  - Web：`web.<LOCALWORD>.local` → `<WEB_PRIV_IP>`
  - AP：`ap.<LOCALWORD>.local` → `<AP_PRIV_IP>`

---

## 2. セキュリティグループ（必須）
### 2-1. Web+DNS（Public）
- inbound
  - 22/tcp：自分のIP
  - 80/tcp：0.0.0.0/0（演習要件）
  - 53/udp：0.0.0.0/0（権威DNS）
  - 53/tcp：0.0.0.0/0（権威DNS）
- outbound：All（デフォルトでOK）

### 2-2. AP（Private）
- inbound
  - 22/tcp：（SSM/踏み台方針に従う）
  - 8080/tcp：**Web+DNSのSGからのみ許可**（外部からは許可しない）
- outbound：All（必要に応じて）

> ✅ ポート方針（外部公開は80のみ）  
> - Webは80公開  
> - APの8080は**外部に開けず**、Webからのみ到達可能にする

---

## 3. SSHログイン（Web+DNS / AP）
### 3-1. Web+DNSへSSH
```bash
ssh -i <key.pem> ec2-user@<WEB_PUB_IP>
```

### 3-2. APへSSH（例：踏み台がWebの場合）
```bash
# Webに入った後
ssh ec2-user@<AP_PRIV_IP>
```

---

# Part A：APサーバ（Tomcatのみ）構築
> **あなたのTomcat手順書を採用**し、Reverse Proxy（httpd）はAPに入れません。  
> APは「Tomcat 9 + Java 8 + アプリ（任意）」まで。

---

## A-0. rootへ切り替え
```bash
sudo su -
cd /home/ec2-user
```

---

## A-1. JDK（Amazon Corretto 8）インストール
```bash
dnf install -y java-1.8.0-amazon-corretto java-1.8.0-amazon-corretto-devel
java -version
javac -version
```

JAVA_HOME確認（後で使う）：
```bash
readlink -f "$(which java)"
# 例：/usr/lib/jvm/java-1.8.0-amazon-corretto/...
```

---

## A-2. Tomcat 9 インストール（/usr/local）
### A-2-1. tomcatユーザ作成
```bash
useradd -r -m -U -s /sbin/nologin tomcat
```

### A-2-2. Tomcatダウンロード（最新版に置換）
```bash
cd /tmp
TOMCAT_VER=9.0.115
wget https://downloads.apache.org/tomcat/tomcat-9/v${TOMCAT_VER}/bin/apache-tomcat-${TOMCAT_VER}.tar.gz
ls -l apache-tomcat-${TOMCAT_VER}.tar.gz
```

### A-2-3. 配置（/usr/local/tomcat）
```bash
mkdir -p /usr/local
tar zxf apache-tomcat-${TOMCAT_VER}.tar.gz -C /usr/local

cd /usr/local
ln -s apache-tomcat-${TOMCAT_VER} tomcat

chown -R tomcat:tomcat /usr/local/apache-tomcat-${TOMCAT_VER}
chown -h tomcat:tomcat /usr/local/tomcat
```

---

## A-3. Tomcat設定
### A-3-1. setenv.sh 作成（JAVA_HOME / メモリ）
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

### A-3-2. server.xml：自動デプロイ無効化（必要なら）
```bash
cp /usr/local/tomcat/conf/server.xml /usr/local/tomcat/conf/server.xml.`date +%Y%m%d-%H%M%S`.bak
vi /usr/local/tomcat/conf/server.xml
```

`<Host ...>` を探して設定：
```xml
<Host name="localhost" appBase="webapps"
      unpackWARs="false" autoDeploy="false" deployOnStartup="false">
```

---

## A-4. systemd（Tomcat自動起動）
### A-4-1. unit作成
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

### A-4-2. 反映＆起動
```bash
systemctl daemon-reload
systemctl enable --now tomcat
systemctl status tomcat --no-pager
ss -lntp | grep 8080
```

---

## A-5. アプリ（knowledge）配備（任意）
> `/knowledge` を出したい場合のみ実施。Tomcatトップを出すならスキップ可。

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

反映：
```bash
chown -R tomcat:tomcat /usr/local/tomcat/conf/Catalina
systemctl restart tomcat
```

---

## A-6. 動作確認（APローカル）
### knowledge配備した場合
```bash
curl -I http://127.0.0.1:8080/knowledge/
```

### Tomcatトップ確認だけしたい場合
```bash
curl -I http://127.0.0.1:8080/
```

---

# Part B：Web+DNSサーバ（Apache + BIND）構築
> Webサーバ側に **Apache Reverse Proxy** と **BIND（権威DNS + ローカルDNS）** を置きます。

---

## B-0. rootへ切り替え & OS更新（要件⑥）
```bash
sudo su -
dnf update -y
```

---

## B-1. Apache（httpd）インストール & 起動
```bash
dnf install -y httpd
systemctl enable --now httpd
systemctl status httpd --no-pager
curl -I http://127.0.0.1/
```

---

## B-2. BIND（named）インストール & 起動
```bash
dnf install -y bind bind-dnssec-utils
systemctl enable --now named
systemctl status named --no-pager
ss -lntup | grep ':53'
```

---

## B-3. named.conf 設定（外部ゾーン + 内部.localゾーン）
### B-3-1. バックアップ
```bash
cp /etc/named.conf /etc/named.conf.$(date +%Y%m%d-%H%M%S).bak
```

### B-3-2. `/etc/named.conf` 編集
```bash
vi /etc/named.conf
```

#### options（例）
```conf
options {
        #listen-on port 53 { 127.0.0.1; <WEB_PRIV_IP>; };
        #listen-on-v6 port 53 { ::1; };
        allow-query { any; };
};
```

#### zone定義（optionsの外に追加）
**外部公開：`<SUBDOMAIN>.entrycl.net`**
```conf
zone "<SUBDOMAIN>.entrycl.net" IN {
        type master;
        file "<SUBDOMAIN>.entrycl.net.zone";
};
```

**内部.local：`<LOCALWORD>.local`**
```conf
zone "<LOCALWORD>.local" IN {
        type master;
        file "<LOCALWORD>.local.zone";
};
```

---

## B-4. ゾーンファイル作成（外部公開用）
```bash
vi /var/named/<SUBDOMAIN>.entrycl.net.zone
```

```zone
$TTL 300
@   IN SOA  ns.<SUBDOMAIN>.entrycl.net. root.<SUBDOMAIN>.entrycl.net. (
        2026021901 ; serial
        300        ; refresh
        120        ; retry
        604800     ; expire
        300        ; minimum
)
    IN NS   ns.<SUBDOMAIN>.entrycl.net.

; 権威DNS（このサーバ）
ns  IN A    <WEB_PUB_IP>

; 外部公開：Web入口（Apache）
@   IN A    <WEB_PUB_IP>
www IN A    <WEB_PUB_IP>
```

---

## B-5. ゾーンファイル作成（内部.local用）
```bash
vi /var/named/<LOCALWORD>.local.zone
```

```zone
$TTL 3600
@   IN SOA  ns.<LOCALWORD>.local. root.<LOCALWORD>.local. (
        2026021901
        3600
        900
        604800
        86400
)
    IN NS   ns.<LOCALWORD>.local.

; 内部DNS（Web+DNSサーバ自身）
ns  IN A    <WEB_PRIV_IP>

; 内部ホスト名（サーバ間通信は.localで）
web IN A    <WEB_PRIV_IP>
ap  IN A    <AP_PRIV_IP>
```

---

## B-6. パーミッション & SELinux（念のため）
```bash
chown root:named /var/named/<SUBDOMAIN>.entrycl.net.zone /var/named/<LOCALWORD>.local.zone
chmod 640 /var/named/<SUBDOMAIN>.entrycl.net.zone /var/named/<LOCALWORD>.local.zone
restorecon -v /var/named/<SUBDOMAIN>.entrycl.net.zone /var/named/<LOCALWORD>.local.zone
```

---

## B-7. 構文チェック → named再起動
```bash
named-checkconf
named-checkzone <SUBDOMAIN>.entrycl.net /var/named/<SUBDOMAIN>.entrycl.net.zone
named-checkzone <LOCALWORD>.local /var/named/<LOCALWORD>.local.zone

systemctl restart named
systemctl status named -l --no-pager
```

---

## B-8. DNS動作確認（Web+DNS上）
### 外部ゾーン
```bash
dig @localhost ns.<SUBDOMAIN>.entrycl.net
dig @localhost <SUBDOMAIN>.entrycl.net
dig @localhost www.<SUBDOMAIN>.entrycl.net
```

### 内部.local（要件④⑤）
```bash
dig @localhost ap.<LOCALWORD>.local
dig @localhost web.<LOCALWORD>.local
```

---

# Part C：Apache Reverse Proxy（Web → AP Tomcat）
> **要件③の本体**：Web側Apacheが、`ap.<LOCALWORD>.local:8080` へ転送する。

---

## C-1. WebからAPへ疎通確認（内部.localで）
```bash
# 名前解決できるか
dig @localhost ap.<LOCALWORD>.local

# Tomcatに到達できるか（knowledge配備した場合）
curl -I http://ap.<LOCALWORD>.local:8080/knowledge/

# Tomcatトップを出す場合
# curl -I http://ap.<LOCALWORD>.local:8080/
```

> ここでタイムアウトしたら：  
> ①APのtomcat起動、②APのSG(8080)許可、③localゾーンのAレコード を確認

---

## C-2. Reverse Proxy設定（Web側Apache）
### /knowledge をAPへ転送する例（knowledge配備した場合）
```bash
vi /etc/httpd/conf.d/tomcat-proxy.conf
```

```apache
ProxyRequests Off

ProxyPass        /knowledge  http://ap.<LOCALWORD>.local:8080/knowledge
ProxyPassReverse /knowledge  http://ap.<LOCALWORD>.local:8080/knowledge
```

反映：
```bash
httpd -t
systemctl restart httpd
```

---

## C-3. 動作確認
### Webサーバ上（ローカル確認）
```bash
curl -I http://127.0.0.1/knowledge/
```

### 外部（ブラウザ）
- `http://<SUBDOMAIN>.entrycl.net/knowledge/`
- または `http://www.<SUBDOMAIN>.entrycl.net/knowledge/`

---

# Part D：要件②（外部から名前解決してアクセス）
> `entrycl.net` の親ゾーンから `<SUBDOMAIN>.entrycl.net` を委譲してもらう必要があります。


## D-1. 外部から確認
手元PCなどで：
```bash
dig ns.<SUBDOMAIN>.entrycl.net
dig <SUBDOMAIN>.entrycl.net
dig www.<SUBDOMAIN>.entrycl.net
```

ブラウザ：
- `http://<SUBDOMAIN>.entrycl.net/knowledge/`

---

# Part E：トラブルシュート（頻出）
## E-1. Web→APのcurlが通らない
Web+DNSで：
```bash
curl -I http://ap.<LOCALWORD>.local:8080/knowledge/
```
通らない場合の優先順位：
1. APでTomcat起動してる？
   ```bash
   systemctl status tomcat --no-pager
   curl -I http://127.0.0.1:8080/knowledge/
   ```
2. APのSGで8080がWebから許可されてる？
3. WebのBINDで `ap.<LOCALWORD>.local` が引けてる？
   ```bash
   dig @localhost ap.<LOCALWORD>.local
   ```

## E-2. 外部から /knowledge が 502/504
- 502：接続拒否/名前解決ミス/ProxyPass先が違う
- 504：タイムアウト（SG/ルート/起動してない）

Webログ：
```bash
tail -n 200 /var/log/httpd/error_log
tail -n 200 /var/log/httpd/app_error.log 2>/dev/null || true
```

---

# 完成チェックリスト（提出前）
- [ ] Web+DNSで `dnf update -y` ができる（要件⑥）
- [ ] 外部から `dig <SUBDOMAIN>.entrycl.net` が引ける（要件②）
- [ ] Web+DNSでBINDが起動し外部53に応答できる（要件①②）
- [ ] `ap.<LOCALWORD>.local` がWeb+DNSで引ける（要件④⑤）
- [ ] Web+DNS → AP の `curl http://ap.<LOCALWORD>.local:8080/...` が通る
- [ ] `http://<SUBDOMAIN>.entrycl.net/knowledge/` でTomcat（アプリ）が表示される（要件③）

---
