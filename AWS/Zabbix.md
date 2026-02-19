# Zabbix 監視システム

## Zabbix サーバ

### 【全体像】

#### 前提・ゴール
- ##### ゴール
  http://<IPアドレス>/zabbix でZabbix UI表示

- ##### 構成（1台に全部載せるパターン）
  OS：Amazon Linux 2023  
  Web：Apache（httpd）
  PHP実行方式：php-fpm（PHP 8.4）
  DB：MariaDB（例：10.5系）
  Zabbix：
  zabbix-server（監視サーバ本体）
  zabbix-web（Web UI：/zabbix）
  zabbix-agent（このサーバ自身を監視するためのエージェント）

- ##### 通信イメージ
  ブラウザ → Apache(80/443) → Zabbix Web UI（PHP）
  Zabbix Web UI（PHP）→ MariaDB(3306)（同一サーバなら localhost）
  監視対象 → Zabbix Server(10051)（必要な場合）
  Zabbix Server → Zabbix Agent(10050)（必要な場合）

- ##### セキュリティグループ（1台構成）
  SSH   | 22   |   MYIP	 |   SSH
  HTTP  | 80   |   MYIP（または社内NW）   |   Zabbix Web UI
  TCP   | 10051 |	 VPCCIDR |	Zabbix Server
  TCP   | 10050 |	 VPCCIDR |	Zabbix Agent

### 手順書

#### 1. OS 基本設定
   1-1) タイムゾーン
   timedatectl set-timezone Asia/Tokyo
   
   1-2) アップデート
   dnf update -y

1. Zabbix 7.4 リポジトリ追加
   公式HP：<URL>https://www.zabbix.com/download
   ```
   rpm -Uvh https://repo.zabbix.com/zabbix/7.4/release/amazonlinux/2023/noarch/zabbix-release-latest-7.4.amzn2023.noarch.rpm
   
   dnf clean all
   ```

1. Apache / PHP8.4(FPM) / MariaDB の導入
   dnf list 
   
   2-1) インストール（Zabbix UI向けPHP拡張も含む）
   ```
   dnf install -y httpd mariadb105-server php8.4 php8.4-cli php8.4-common php8.4-fpm php8.4-mysqlnd php8.4-gd php8.4-xml php8.4-mbstring php8.4-bcmath php8.4-opcache
   ```

2. Zabbix（Server/Web/Agent）インストール
   ```
   dnf install -y zabbix-server-mysql zabbix-web-mysql zabbix-apache-conf zabbix-sql-scripts zabbix-agent zabbix-get
   ```

3. 動作確認
   <code> php -v </code>
   <code> systemctl status httpd mariadb php-fpm --no-pager </code>

### Apache が conf.d を読み込むことを最初に担保（重要）
1. httpd.conf に conf.d include があるか確認
   ```
   grep -nE 'IncludeOptional\s+conf\.d/\*\.conf|IncludeOptional\s+/etc/httpd/conf\.d/\*\.conf' /etc/httpd/conf/httpd.conf
   
   ※もし何も出なければ追記（修正）
   echo 'IncludeOptional /etc/httpd/conf.d/*.conf' | sudo tee -a /etc/httpd/conf/httpd.conf
   ```

2. 読み込み確認（DUMP_INCLUDESが確実）
   <code> sudo httpd -t -D DUMP_INCLUDES 2>&1 | grep -i zabbix || true </code>

#### サービス起動（ここで /zabbix の公開確認までやる）
1. 起動・自動起動
   <code> systemctl enable --now httpd mariadb php-fpm </code>
   

1. Apache 側の /zabbix が見えるか確認（DB前でも確認できる）
   <code> curl -I http://127.0.0.1/zabbix </code>
   <code> curl -I http://127.0.0.1/zabbix/ </code>
   
   >期待値：
   /zabbix → 301 など
   /zabbix/ → 200 / 302（setup.php等）


#### MariaDB 初期設定＆Zabbix用DB作成

1. 初期設定
   <code> mysql_secure_installation </code>
   
2. DBとユーザー作成
   <code> mysql -uroot -p </code>
   ```
   MariaDB内で（パスワードは置換）：
   CREATE DATABASE zabbix CHARACTER SET utf8mb4 COLLATE utf8mb4_bin;
   CREATE USER 'zabbix'@'localhost' IDENTIFIED BY 'zabbix';
   GRANT ALL PRIVILEGES ON zabbix.* TO 'zabbix'@'localhost';
   FLUSH PRIVILEGES;
   QUIT;
   ```

1. Zabbix DBスキーマ投入
   ```
   zcat /usr/share/zabbix/sql-scripts/mysql/server.sql.gz | mysql --default-character-set=utf8mb4 -u zabbix -p zabbix  

   ※ここで入力するパスワード：zabbix@localhost のパスワード（＝2で設定した ZBX_DB_PASS）
   ```

#### Zabbix server 設定（DB接続）
   >1台構成なので DBHost は基本 “不要（localhost扱い）”。
   書くなら DBHost=localhost に統一。
1. 設定バックアップ
   <code> cp /etc/zabbix/zabbix_server.conf{,.org} </code>  
   <code> ls /etc/zabbix | grep zabbix_server.conf </code> 

2. DB設定
   <code> vi /etc/zabbix/zabbix_server.conf </code>
   ```
   /etc/zabbix/zabbix_server.conf を編集して以下を設定：
   DBHost=<DBサーバのIP>　
   DBName=zabbix
   DBUser=zabbix
   DBPassword=zabbix
   ```
   確認：
   <code> sudo egrep '^(DBName|DBUser|DBPassword)=' /etc/zabbix/zabbix_server.conf </code>
   >下記のように表示されればOK
   DBName=zabbix
   DBUser=zabbix
   DBPassword=zabbix
   
#### Zabbix agent 設定（必要最低限）
   同一サーバ監視なら、基本デフォルトでも動くことが多い。必要なら以下。
1. 設定バックアップ
   <code> cp /etc/zabbix/zabbix_agentd.conf{,.org} </code>
   <code> ls /etc/zabbix | grep zabbix_agentd.conf </code>  
1. 設定編集
   <code> vi /etc/zabbix/zabbix_agentd.conf </code>
   ```
   以下を設定：
   Server=127.0.0.1
   ServerActive=127.0.0.1
   Hostname=<hostname -s の値>
   ```

1.  Zabbix サービス起動
   <code> sudo systemctl enable --now zabbix-server zabbix-agent </code>
   <code> sudo systemctl status zabbix-server zabbix-agent --no-pager </code>

2.  Web UI 初期セットアップ
    ブラウザで開く：
    http://<サーバのIP>/zabbix/
    
    >セットアップ画面入力（同一サーバなので localhost）：
    DB host：localhost
    DB name：zabbix
    DB user：zabbix
    DB password：ZBX_DB_PASS

---
###### 最終チェック<必要に応じて>
1) サービス起動状態
   sudo systemctl status httpd php-fpm mariadb zabbix-server zabbix-agent --no-pager

2) 自動起動（Enable）
   sudo systemctl is-enabled httpd php-fpm mariadb zabbix-server zabbix-agent

3) ポート確認（ローカル）
   ss -lntp | egrep ':(80|443|3306|10050|10051)\b' || true

4) Zabbixサーバログ（起動直後エラー確認）
   sudo tail -n 50 /var/log/zabbix/zabbix_server.log

- ポイント！
  /zabbix が404の時は DBより先に DUMP_INCLUDES と curl -I /zabbix/ を見る
  → “Apache連携の問題” を最短で切り分けできる


---

## 監視対象サーバの構築

1. OS 基本設定
   1-1) タイムゾーン
   timedatectl set-timezone Asia/Tokyo
   
   1-2) アップデート
   dnf update -y

1. Zabbix 7.4 リポジトリ追加
   公式HP：<URL>https://www.zabbix.com/download
   ```
   rpm -Uvh https://repo.zabbix.com/zabbix/7.4/release/amazonlinux/2023/noarch/zabbix-release-latest-7.4.amzn2023.noarch.rpm
   dnf clean all
   ```

1. Zabbix（Agent）インストール
   <code> dnf install -y zabbix-agent zabbix-get </code>

1. Zabbix agent 設定（必要最低限）
   同一サーバ監視なら、基本デフォルトでも動くことが多い。必要なら以下。
   <code> cp /etc/zabbix/zabbix_agentd.conf{,.org} </code> 
   <code> ls /etc/zabbix | grep zabbix_agentd.conf </code>  
   <code> vi /etc/zabbix/zabbix_agentd.conf </code>
   ```
   以下を設定：
   Server=<ZabbixServerのプライベートIP>
   ServerActive=<ZabbixServerのプライベートIP>
   Hostname=<監視対象の名前>
   ```

1.  Zabbix サービス起動
   <code> sudo systemctl enable --now zabbix-agent </code>
   <code> sudo systemctl status zabbix-agent --no-pager </code>

1.  Zabbix UIからホスト追加
    データ収集 → ホスト 
    → [ホストの作成]
    - ホスト名 :zabbix_agentd.confのHostname=と一致
    - 表示名：わかりやすい名前を指定
    - テンプレート：Linux by Zabbix agent（適当に設定）
    - ホストグループ：Linux servers（適当に設定）
    - インターフェース：エージェント | 監視対象サーバのプライベートIP | IP | 10050
    →[更新]
    
    エージェントの状態[ZBX]が緑になっていることを確認  
   
#### トリガー設定・テスト

1. 手順（UI）
   データ収集 → ホスト
   監視対象ホストを開く（ホスト名の横の・・・を押す）
   トリガー →右上の[トリガーの作成]

1. 基本項目
   入力例：こんな内容で作る（まずはテスト用に低めでOK）
   名前：CPU使用率が高い（avg5m > 80%）
   深刻度：Warning など
   有効：ON のままでOK

1. いちばん重要：条件（式）の作り方
   トリガー画面に 「式」(Expression) の入力欄があるはず。
   
   1. “追加” ボタンからアイテムを選ぶ（おすすめ）
   式のところに 「追加」/「選択」/「挿入」 みたいなボタン（虫眼鏡や「…」）があるので押す
   （UI表記は環境で少し違うけど、必ず「式にアイテムを入れる」ためのボタンがある）
   
   すると アイテム選択画面が出るので：
   ホスト：KannshiTaisho1（またはすでに固定されてる）
   検索欄に CPU と入力
   だいたい次のようなアイテムを選ぶ：
   CPU utilization / CPU utilization (average) など
   （Zabbix agent2だと system.cpu.util 系の名前のこともある）
   選ぶと、式に アイテム参照が挿入されます。
   
   2. “avg(5m)” と “>80” を付ける
   挿入されたアイテムの後ろに、関数を付けます。
   考え方はこれだけ：
   直近5分の平均 → avg(5m)
   80%より大きい → >80
   
   >最終的に式がこういう意味になればOK：
   CPU使用率の 5分平均が 80 を超えたら発火
   もし「すぐテストしたい」なら >80 を一時的に >20 とかにすると、負荷を少し上げるだけで発火確認できます。

1. CPU負荷を“テスト的にかける方法”
   一番おすすめ：stress-ng（安全・止めやすい）
   監視対象サーバで：
   sudo dnf install -y stress-ng
   CPUを上げる（例：2分だけ、CPUコア数ぶん負荷）：
   
   CORES=$(nproc)
   stress-ng --cpu $CORES --cpu-method matrixprod --timeout 120s
   
   --timeout 120s なので 2分で自動停止（安全）
   止めたい時は Ctrl+C

1. アラート確認
   負荷をかけたら Zabbix UI で：
   監視 → 障害（トリガーが発火してるか）
   監視 → 最新データ → CPU（グラフ/値が上がってるか）

---

## 障害通知設定（自分の社用outlookにメール送信）

- 全体像：登場人物は3つだけ
  Zabbix → 127.0.0.1:25 → Postfix → 社用Outlook
  - Zabbix（通知を作る人）：「トリガーが障害になった！メール送って！」を実行する
  - Postfix（郵便局）：Zabbixから受け取ったメールを、外部の宛先（Outlook）へ配送する
  - Outlook（受信箱）：メールを受け取る側

### Zabbixサーバ側の設定  

1. Postfix を入れて起動（Zabbixサーバ上）
   sudo dnf install -y postfix mailx
   sudo systemctl enable --now postfix
   sudo systemctl status postfix --no-pager

1. Postfix を “インターネット配送” 用の最小設定にする
   Amazon Linux 2023 はデフォルトだとローカル配送寄りなので、まず最小で外に出せる形に寄せる。
   
   設定編集
   sudo cp -a /etc/postfix/main.cf /etc/postfix/main.cf.bak.$(date +%F)
   sudo vi /etc/postfix/main.cf
   
   main.cf に以下を 追記 or 修正（そのままコピペでOK）：
   #--- minimal outbound settings ---
   myhostname = zbx.local
   myorigin = $myhostname
   mydestination = $myhostname, localhost.$mydomain, localhost
   
   inet_interfaces = loopback-only
   inet_protocols = ipv4
   
   #outbound: use public DNS
   relayhost =
   
   #(optional) avoid slow IPv6 resolution issues
   smtp_address_preference = ipv4

   ※inet_interfaces = loopback-only にしてるのは、演習で「外部からSMTPを受けない」安全寄り構成にするため（Zabbixからlocalhostで投げるだけだからこれでOK）。

1. 反映：
   sudo postfix check
   sudo systemctl restart postfix
   sudo systemctl status postfix --no-pager

2. OS側の疎通テスト（Zabbixより先にここを必ず）
   まず Postfix単体で社用メールに届くことを確認。
   <code> echo "postfix test" | mail -s "Postfix test from Zabbix server" midori.kuramochi@rakus-partners.co.jp </code>
   
   届かない場合に見るログ（ここ超重要）
   <code> sudo journalctl -u postfix -n 200 --no-pager </code>
   
   ☆よくある詰まり：
   25番アウトバウンドが塞がれている（AWSや社内NWで多い）
   outlook.com 側が「認証なしの直送」を拒否（最近かなり多い）
   DNS引けない/逆引き/HELO などで弾かれる
   この場合は「Postfixから 中継（relayhost） を使う」方式に切り替えるのが正解（社内SMTPリレー or M365 SMTP リレー）。
   ※ここは環境依存なので、ログを見れば次の一手が即決できる。

### Zabbix UI 側の設定（127.0.0.1:25 へ投げる）

1. メディアタイプ（Email）
   Zabbix UI → アラート → メディアタイプ → Email
   SMTP server：127.0.0.1
   SMTP server port：25
   email（From）：zabbix@zbx.local
   SMTP helo：zbx.local
   （メッセージテンプレートはデフォルトでもOK。必要なら後で整える）


1. ユーザーに受信先を紐付け
   Zabbix UI → ユーザー →（自分）→ メディア → 追加
   タイプ：Email
   送信先：midori.kuramochi@rakus-partners.co.jp
   有効：ON

1. アクション（トリガー発生で送る）
   Zabbix UI → アラート → アクション → トリガーアクション → 作成
   条件：Severity >= 警告（まずはこれでOK）
   実行内容：自分へ Email 送信

1. CPU負荷を“テスト的にかける方法”
   一番おすすめ：stress-ng（安全・止めやすい）
   監視対象サーバで：
   <code> sudo dnf install -y stress-ng </code>
   ```
   CPUを上げる（例：2分だけ、CPUコア数ぶん負荷）：
   
   CORES=$(nproc)
   stress-ng --cpu $CORES --cpu-method matrixprod --timeout 120s
   
   --timeout 120s なので 2分で自動停止（安全）
   止めたい時は Ctrl+C
   ```

1. アラート・メール確認
   負荷をかけたら Zabbix UI で：
   監視 → 障害（トリガーが発火してるか）
   監視 → 最新データ → CPU（グラフ/値が上がってるか）
   負荷をかけたら 社用outlook で：
   届いているか確認
   ない場合、迷惑メールに入っている可能性あり