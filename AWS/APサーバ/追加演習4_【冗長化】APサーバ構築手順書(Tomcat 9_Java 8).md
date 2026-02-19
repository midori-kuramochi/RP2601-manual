# APサーバ演習4 手順書（AP冗長化・Apache版）
対象：Amazon Linux 2023 / Tomcat 9 + Java8 / Apache Reverse Proxy + Load Balancing  
前提：**WebはApache 1台（WEB01）**、APは **2台（AP01, AP02）**、アプリは両方 `/knowledge`

---

## 0. 前提（構成）
### 0-1. 構成イメージ
```
Internet
   │  (80)
   ▼
WEB01 (Apache: Reverse Proxy + Load Balancer)
   ├── AP01 (Tomcat:8080 /knowledge)
   └── AP02 (Tomcat:8080 /knowledge)
```

### 0-2. IP例（置換してOK）
- WEB01 Private IP：`10.0.1.10`（Public IPあり）
- AP01  Private IP：`10.0.2.10`（Privateのみ）
- AP02  Private IP：`10.0.2.11`（Privateのみ）

### 0-3. セキュリティグループ（超重要）
**WEB01（Web）SG**
- inbound：80/tcp（必要範囲）、22/tcp（自分のIP/踏み台）
- outbound：AP01/AP02 の 8080 へ到達できること（通常 all でOK）

**AP01/AP02（AP）SG**
- inbound：
  - 8080/tcp：**WEB01のSG**（または WEB01 のPrivate IP）からのみ許可
  - 22/tcp：踏み台/自分のIP（構成に合わせる）
- outbound：all

> ✅この演習の核：Tomcat(8080)を外に出さず、Webサーバからだけアクセスできるようにして「APを2台に増やして負荷分散・片系停止でも継続」させる。

---

# 1. APサーバ（AP01 / AP02）構築（Tomcat 9 + Java8）

## 1-1. 共通：ログインしてrootへ
AP01 と AP02 の両方で実施
```bash
sudo su -
cd /home/ec2-user
```

## 1-2. 共通：JDK（Amazon Corretto 8）インストール
```bash
dnf install -y java-1.8.0-amazon-corretto java-1.8.0-amazon-corretto-devel
java -version
javac -version
```

## 1-3. 共通：Tomcatインストール
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

## 1-4. 共通：setenv.sh 作成
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

## 1-5. 共通：server.xml（自動デプロイ無効 + 8080待受）
```bash
vi /usr/local/tomcat/conf/server.xml
```

### (1) 自動デプロイ無効化
```xml
<Host name="localhost" appBase="webapps"
      unpackWARs="false" autoDeploy="false" deployOnStartup="false">
```

### (2) Tomcat 8080 は Webサーバから届く必要がある
分離/冗長化では `address="127.0.0.1"` を付けない（外す）。
```xml
<Connector port="8080" protocol="HTTP/1.1"
           connectionTimeout="20000"
           redirectPort="8443" />
```

> ✅セキュリティは **SGでWEB01からのみ許可**して守る

## 1-6. 共通：systemd（Tomcat自動起動）
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

## 1-7. 共通：アプリ配備（/knowledge）
AP01 と AP02 の両方で **同じ手順**で配備

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

## 1-8. APサーバ単体の動作確認（ローカル）
AP01 / AP02 それぞれで
```bash
curl -I http://127.0.0.1:8080/knowledge/
```

---

# 2. Webサーバ（WEB01）構築（Apache Load Balancer）

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

## 2-3. 必要モジュール確認（proxy / balancer）
```bash
httpd -M | egrep 'proxy_module|proxy_http_module|proxy_balancer_module|lbmethod_byrequests_module' || true
```

> もし `proxy_balancer_module` などが出ない場合は、`/etc/httpd/conf.modules.d/` のLoadModuleが無効になっている可能性があります。

## 2-4. ロードバランサ設定（/knowledge を AP01/AP02 に振り分け）
```bash
vi /etc/httpd/conf.d/tomcat-balancer.conf
```

以下の例では **ラウンドロビン（byrequests）**で振り分けします。

```apache
ProxyRequests Off
ProxyPreserveHost On

# バランサ定義（AP01/AP02）
<Proxy "balancer://tomcat_knowledge">
    BalancerMember "http://10.0.2.10:8080" route=ap01
    BalancerMember "http://10.0.2.11:8080" route=ap02

    # 振り分け方式：リクエスト数ベース（一般的）
    ProxySet lbmethod=byrequests
</Proxy>

# /knowledge をバランサへ
ProxyPass        "/knowledge"  "balancer://tomcat_knowledge/knowledge"
ProxyPassReverse "/knowledge"  "balancer://tomcat_knowledge/knowledge"
```

### （任意）スティッキーセッション（ログイン系アプリで有効）
アプリがセッションをTomcatのメモリに持つタイプだと、APが切り替わるとログインが外れることがあります。  
その場合は sticky を検討します（演習要件次第）。

Tomcat側のJSESSIONIDに `route` が付く前提で、Apache側をこうします：

```apache
<Proxy "balancer://tomcat_knowledge">
    BalancerMember "http://10.0.2.10:8080" route=ap01
    BalancerMember "http://10.0.2.11:8080" route=ap02
    ProxySet lbmethod=byrequests stickysession=JSESSIONID
</Proxy>
```

※この場合、Tomcat側で `jvmRoute` を設定する必要があることがあります（server.xml の Engine に設定）。  
演習で求められたら追加します。

## 2-5. 反映（文法チェック→再起動）
```bash
httpd -t
systemctl restart httpd
systemctl status httpd --no-pager
```

---

# 3. WEB01 → AP01/AP02 疎通確認（超重要）
WEB01 から両方に届くことを確認：

```bash
curl -I http://10.0.2.10:8080/knowledge/
curl -I http://10.0.2.11:8080/knowledge/
```

通らない場合は：
- AP01/AP02のSGで 8080 が WEB01 から許可されていない
- Tomcatが起動していない / 8080待受していない
- ルーティング/サブネット/ACL

---

# 4. 全体の動作確認（外部）
外部クライアントから：
- `http://WEB01のグローバルIP/knowledge` にアクセスして表示されること

---

# 5. 冗長化の動作確認（片系停止テスト）
## 5-1. 両方稼働している状態でアクセス
```bash
curl -I http://WEB01のグローバルIP/knowledge
```

## 5-2. AP01 を停止してもサービス継続するか
AP01で：
```bash
systemctl stop tomcat
```

外部から：
- `http://WEB01のグローバルIP/knowledge` が表示され続けること（AP02へ流れる）

AP01を戻す：
```bash
systemctl start tomcat
```

同様にAP02を止めても継続するか確認。

> 注意：Apache側の設定/状態によっては、停止したAPに一時的に投げて失敗→次へ、となることがあります。  
> 演習要件で「落ちたAPを自動で外す」まで求められるなら、ヘルスチェックやfailover設定を追加します。

---

# 6. 仕上げチェック（OK判定）

## 6-1. AP01 / AP02
```bash
systemctl status tomcat --no-pager
ss -lntp | grep 8080
curl -I http://127.0.0.1:8080/knowledge/
```

## 6-2. WEB01
```bash
systemctl status httpd --no-pager
httpd -t
curl -I http://127.0.0.1/knowledge
```

---

# 7. よくあるトラブル

## 502/503が出る（Apache→Tomcat）
- WEB01からAPへ疎通：
  - `curl -I http://10.0.2.10:8080/knowledge/`
  - `curl -I http://10.0.2.11:8080/knowledge/`
- AP側ログ：
  - `journalctl -u tomcat -n 200 --no-pager`
- WEB側ログ：
  - `journalctl -u httpd -n 200 --no-pager`

## 片系停止で落ちる
- APのSGが片方だけ許可されていない
- バランサ設定のIP/ポート間違い
- Tomcatが片方だけ起動していない

## ログインが外れる/状態が保持されない
- スティッキーセッション（`stickysession=JSESSIONID`）を検討
- もしくはセッション共有/ステートレス化（演習要件次第）

