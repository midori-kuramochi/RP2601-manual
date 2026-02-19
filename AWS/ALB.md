# ALB

0. 演習の概要
   - 人数：2人
   - 準備：各自 EC2 を1台作成（計2台：サーバA / サーバB）
   - 目的：
     ALB を作成し、2台へ均等に負荷分散できることを確認する
     パスによって転送先を分ける（/saba → サーバA、/miso → サーバB）
     ヘルスチェック専用ページ /health を作り、ALB がそこをチェックする
     /health のアクセスログを通常ログと分けて保存する

---

### EC2側（サーバA / サーバB）のHTTPサーバ構築

##### 1. Apacheインストール＆起動（両サーバ共通）
   ```
   sudo dnf install -y httpd
   sudo systemctl enable --now httpd
   curl -I http://127.0.0.1/
   ```

##### 2. Step1：ALBで「2台へ均等に負荷分散」する
   ここは ターゲットグループ1つに2台登録してOK。
   
1. ターゲットグループ（TG-rr）作成
   - ターゲットの種類：インスタンス
   - プロトコル：HTTP
   -  ポート：80
   -  ヘルスチェック：あとで /health に変更する（今は仮で / でもOK）
   -  作成後、サーバA・サーバBの2台を登録する。

1. ALB作成
   - 種類：Application Load Balancer
   - スキーム：インターネット向け
   - VPC：default
   - マッピング：EC2が存在するAZを2つ以上選択（重要）
   - リスナー：HTTP 80
   - 転送先：TG-rr

1. 動作確認(均等分散)
   - ALBのDNSにアクセスする：
     ブラウザ：http://<ALB_DNS>/
   - もしくはコマンド：
      ```
      ↓ALBに10回アクセスして、毎回のHTTPステータスコードだけを表示する   
      for i in {1..10}; do curl -s -o /dev/null -w "%{http_code}\n" http://<ALB_DNS>/; done   
      
      ↓でもOK
      curl -s http://<ALB_DNS>/
      ```

   - 両サーバでアクセスログ確認：
      ```
      tail -f /var/log/httpd/access_log
      ```
      両方のサーバに同じ時間帯のアクセスが入っていればOK。
      ☆成功していれば、「ELB-HealthChecker/2.0」からアクセスが来ているはず！
      (例)
      >[root@ip-172-31-18-123 ~]# tail -f /var/log/httpd/access_log
      172.31.35.54 - - [12/Feb/2026:06:52:50 +0000] "GET / HTTP/1.1" 403 191 "-" "ELB-HealthChecker/2.0"



### 演習2：パスで転送先を分ける（/saba → A、/miso → B）

ここは ターゲットグループを2つに分ける必要がある（超重要）。

1. サーバAにコンテンツ作成
   ```
   sudo mkdir -p /var/www/html/saba
   echo "server A (saba)" | sudo tee /var/www/html/saba/saba.html >/dev/null
   sudo chmod 755 /var/www/html/saba
   sudo chmod 644 /var/www/html/saba/saba.html
   curl -i http://127.0.0.1/saba/saba.html
   ```

2. サーバBにコンテンツ作成
   ```
   sudo mkdir -p /var/www/html/miso
   echo "server B (miso)" | sudo tee /var/www/html/miso/miso.html >/dev/null
   sudo chmod 755 /var/www/html/miso
   sudo chmod 644 /var/www/html/miso/miso.html
   curl -i http://127.0.0.1/miso/miso.html
   ```

3. ターゲットグループを2つ作る
   - TG-saba：サーバAだけ登録（HTTP:80）
   - TG-miso：サーバBだけ登録（HTTP:80）

4. ALBのリスナー（HTTP:80）にルール追加
   - Listener rules に以下を追加：
   - ルール1：Path /saba と /saba/* → Forward to TG-saba
   - ルール2：Path /miso と /miso/* → Forward to TG-miso
   - Default：それ以外 → 任意（どちらかへ）

5. 動作確認（パス分岐）
   ```
   curl -i http://<ALB_DNS>/saba/saba.html
   curl -i http://<ALB_DNS>/miso/miso.html
   ```
   - /saba/... で server A の文字が返る
   - /miso/... で server B の文字が返る

### 演習3.ヘルスチェック専用ページの作成（/health）

両サーバで /health を作成する（200 OKを返すこと）
```
echo OK | sudo tee /var/www/html/health >/dev/null
sudo chmod 644 /var/www/html/health
curl -i http://127.0.0.1/health
```

#### /health のアクセスログを通常ログと分離（Apache設定）

両サーバで設定する。

1. 設定ファイル作成
   ```
   sudo vi /etc/httpd/conf.d/healthcheck-log.conf
   ```
   中身：
   ```
   #/health に来たアクセスを「ヘルスチェック」として印をつける
   SetEnvIf Request_URI "^/health$" is_health
   
   #通常アクセスログ：/health は書かない
   CustomLog /var/log/httpd/access_log combined env=!is_health
   
   #ヘルスチェック専用ログ：/health だけ書く
   CustomLog /var/log/httpd/healthcheck_access_log combined env=is_health
   ```
   反映：
   <code> sudo httpd -t </code>
   <code> sudo systemctl restart httpd </code>

2. ルーティング先TGのヘルスチェック設定を /health に変更
   Path：/health
   Success codes：200
   Port：traffic port 推奨
   （TG-rr / TG-saba / TG-miso それぞれで設定するのが安全）
   

3. ログ確認
   ```
   sudo tail -f /var/log/httpd/healthcheck_access_log
   ```
   >ALBのヘルスチェックが来ると /health のアクセスが定期的に出る。
   
   (例)
   [root@ip-172-31-18-123 ~]# tail -f /var/log/httpd/healthcheck_access_log
   
   118.237.255.201 - - [12/Feb/2026:07:37:59 +0000] "GET /health HTTP/1.1" 200 3 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/144.0.0.0 Safari/537.36"

ヘルスチェック
ターゲット
Lヘルスチェック
Lパス：/health
L間隔：5秒
Lタイムアウト：2秒
L閾値：2ヘルスチェック

ログ確認

→ターゲットからヘルスチェックが緑になっていることを確認する


3-2. ヘルスチェック専用ログの確認（各EC2で）
```
sudo tail -f /var/log/httpd/healthcheck_access_log
```
例：
172.31.xx.xx - - [12/Feb/2026:07:41:54 +0000] "GET /health HTTP/1.1" 200 3 "-" "..."

3-3. コンソールでターゲットが Healthy になっていることを確認

Target group → Targets で Status が green (healthy) になっていること。

（目安）

Interval 5秒 / Healthy threshold 2 なら、だいたい 10秒前後で healthy になりやすい