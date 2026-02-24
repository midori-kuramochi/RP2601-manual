# 演習3：BIG-IP を CLI（TMOS / tmsh）で設定する手順書（踏み台経由・管理IF接続版）

## 0. 目的
これまでGUIで実施した以下の設定を、CLI（TMOS / tmsh）で実施する。

- Node（Webサーバ）登録
- Pool 作成
- Health Monitor（HTTP監視）設定
- Virtual Server 作成
- 負荷分散方式の変更（Round Robin / Ratio）
- （演習2）全Webサーバダウン時の sorry ページ表示（Fallback Host）

---

## 1. 前提

### 1-1. 管理アクセス経路
- 管理者は **踏み台サーバ（Public）** を経由して BIG-IP に接続する
- BIG-IP への管理接続は **admin用インターフェース（management）** を使用する
- BIG-IP には **`admin` ユーザー** で SSH ログインする

### 1-2. 技術前提
- BIG-IP（LTM）にログインできること
- `tmsh` が利用できること
- Webサーバ（Node）のIPがわかっていること
- Webサーバで Apache が起動していること
- `/` または `/health` が **200 OK** を返すこと
- BIG-IP → Webサーバへの疎通（SG / NACL / ルート）が通っていること

### 1-3. 例（今回の変数）
> 実際の値に読み替えて使うこと

- 踏み台サーバ（Public）：`<Bastion-Public-IP>`
- BIG-IP 管理IP（admin interface）：`<BIGIP-MGMT-IP>`
- Web1：`172.31.1.250:80`
- Web2：`172.31.1.251:80`
- VIP（Virtual Server宛先 / BIG-IP側LTM用IP）：`172.31.1.100:80`

### 1-4. 注意
- **管理用IP（SSH/GU I用）** と **Virtual Server用IP（VIP）** は別もの
- CLIで設定変更後は、最後に **`save sys config`** を実行して保存すること

---

## 2. BIG-IPへ接続（踏み台経由）

### 2-1. 管理者PC → 踏み台サーバ
```bash
ssh -i <踏み台用鍵.pem> ec2-user@<踏み台サーバPublicIP>
```

### 2-2. 踏み台サーバ → BIG-IP（管理用インターフェース）
```bash
ssh admin@<BIG-IP管理IP>
```

> 例：`ssh admin@172.31.x.x`  
> 初回はフィンガープリント確認が出る場合があるため `yes` を入力

### 2-3. tmsh を起動
```bash
tmsh
```

---

## 3. 演習1：Node / Pool / Virtual Server を CLI で作成

### 3-1. Node を作成
```tcl
create ltm node web01 address 172.31.1.250
create ltm node web02 address 172.31.1.251
```

#### 確認
```tcl
list ltm node
show ltm node
```

---

### 3-2. HTTP Monitor（ヘルスチェック）を作成

#### パターンA：`/` を監視する（簡易）
```tcl
create ltm monitor http mon_http_root \
    interval 5 timeout 16 \
    send "GET / HTTP/1.1\r\nHost: localhost\r\nConnection: close\r\n\r\n" \
    recv "200 OK"
```

#### パターンB：`/health` を監視する（推奨）
```tcl
create ltm monitor http mon_http_health \
    interval 5 timeout 16 \
    send "GET /health HTTP/1.1\r\nHost: localhost\r\nConnection: close\r\n\r\n" \
    recv "200 OK"
```

> `timeout 16` は `interval 5` に対して安定しやすい値（目安：5×3+1）

#### 確認
```tcl
list ltm monitor http mon_http_root
list ltm monitor http mon_http_health
```

---

### 3-3. Pool を作成（Round Robin）

#### `/` を監視する場合
```tcl
create ltm pool pool_web \
    monitor mon_http_root \
    load-balancing-mode round-robin \
    members add { 172.31.1.250:80 172.31.1.251:80 }
```

#### `/health` を監視する場合
```tcl
create ltm pool pool_web \
    monitor mon_http_health \
    load-balancing-mode round-robin \
    members add { 172.31.1.250:80 172.31.1.251:80 }
```

#### 確認
```tcl
list ltm pool pool_web
show ltm pool pool_web members
```

---

### 3-4. Virtual Server を作成
```tcl
create ltm virtual vs_http_web \
    destination 172.31.1.100:80 \
    ip-protocol tcp \
    mask 255.255.255.255 \
    pool pool_web \
    source-address-translation { type automap } \
    profiles add { tcp http }
```

#### 確認
```tcl
list ltm virtual vs_http_web
show ltm virtual vs_http_web
```

> `source-address-translation { type automap }` は、AWS構成での戻り通信のハマり防止として推奨

---

## 4. 演習2-1：均等分散（Round Robin）

### 4-1. Poolの負荷分散方式を Round Robin に設定（または確認）
```tcl
modify ltm pool pool_web load-balancing-mode round-robin
```

### 4-2. 確認
```tcl
list ltm pool pool_web one-line
```

### 4-3. 動作確認（推奨）
各Webサーバの `index.html` に識別用文字列を設定する。

#### 例（各Webサーバで）
```bash
echo web01 | sudo tee /var/www/html/index.html
# 別サーバでは web02 にする
```

LBのEIP（またはVIP）に複数回アクセスし、表示が交互（均等）に切り替わることを確認する。

---

## 5. 演習2-2：全Webサーバダウン時の sorry ページ表示（Fallback Host）

### 5-1. sorryページを作成（Webサーバ側）
任意のWebサーバ（またはsorry専用サーバ）に配置する。

```bash
echo '<h1>Sorry... ただいまメンテナンス中です</h1>' | sudo tee /var/www/html/sorry.html
```

---

### 5-2. Fallback Host 用 HTTP Profile を作成
```tcl
create ltm profile http http_fallback_sorry \
    defaults-from http \
    fallback-host "http://172.31.1.250/sorry.html"
```

> `fallback-host` は、全Poolメンバー利用不可時にリダイレクトされるURL（302）

#### 確認
```tcl
list ltm profile http http_fallback_sorry
```

---

### 5-3. Virtual Server に HTTP Profile を適用
既存の `http` profile を、作成した `http_fallback_sorry` に置き換える。

```tcl
modify ltm virtual vs_http_web profiles replace-all-with { tcp http_fallback_sorry }
```

#### 確認
```tcl
list ltm virtual vs_http_web profiles
```

> `tcp` と `http_fallback_sorry` の両方が表示されることを確認

---

### 5-4. 動作確認（Pool memberを止める）
#### Pool member を Forced Offline（または同等状態）にする例
```tcl
modify ltm pool pool_web members modify { 172.31.1.250:80 { session user-disabled state user-down } }
modify ltm pool pool_web members modify { 172.31.1.251:80 { session user-disabled state user-down } }
```

#### 確認
```tcl
show ltm pool pool_web members
```

→ 全メンバーが利用不可の状態で、LBのEIPへアクセスし、  
`sorry.html` にリダイレクトされることを確認する。

---

### 5-5. 復旧（Pool member を有効化）
```tcl
modify ltm pool pool_web members modify { 172.31.1.250:80 { session user-enabled state user-up } }
modify ltm pool pool_web members modify { 172.31.1.251:80 { session user-enabled state user-up } }
```

---

## 6. 応用：Ratio で重み付け分散する

### 6-1. Poolの負荷分散方式を `ratio-member` に変更
```tcl
modify ltm pool pool_web load-balancing-mode ratio-member
```

### 6-2. 各Memberの Ratio を設定
例：web01=3、web02=1（3:1で配分）

```tcl
modify ltm pool pool_web members modify { 172.31.1.250:80 { ratio 3 } }
modify ltm pool pool_web members modify { 172.31.1.251:80 { ratio 1 } }
```

### 6-3. 確認
```tcl
list ltm pool pool_web members
show ltm pool pool_web members
```

---

## 7. よく使う確認コマンド（CLI）

### 7-1. Node
```tcl
show ltm node
list ltm node
```

### 7-2. Pool / Member
```tcl
show ltm pool
show ltm pool pool_web members
list ltm pool pool_web
```

### 7-3. Monitor
```tcl
list ltm monitor http
```

### 7-4. Virtual Server
```tcl
show ltm virtual
show ltm virtual vs_http_web
list ltm virtual vs_http_web
```

---

## 8. 設定保存（重要）
CLIで設定変更後は必ず保存する。

```tcl
save sys config
```

> 保存しないと再起動後に設定が失われる可能性がある

---

## 9. トラブルシュート

### 9-1. Pool member が Down のまま
- Monitor の `send` / `recv` 設定を確認
- Webサーバで以下を実行し、200が返るか確認
  ```bash
  curl -i http://127.0.0.1/
  curl -i http://127.0.0.1/health
  ```
- Security Group で BIG-IP → Webサーバ:80 を許可しているか確認

### 9-2. Virtual Server が Offline
- `show ltm virtual vs_http_web` で状態確認
- `pool pool_web` が正しく紐づいているか確認
- `destination`（VIP）が正しいか確認
- `source-address-translation { type automap }` が設定されているか確認
- 必要に応じて GUI で VLAN 制限（All VLANs and Tunnels）も確認

### 9-3. LBのEIPで表示されない
- BIG-IPのSecurity Groupで 80 / 443 が開いているか確認
- EIP が正しいENI（外向き側）に紐づいているか確認
- VSの宛先IP（VIP）とEIPが紐づくBIG-IPのPrivate IPの関係を確認
- Apacheログ（Node側）で BIG-IP からのアクセスが来ているか確認
  ```bash
  sudo tail -f /var/log/httpd/access_log
  ```

---

## 10. まとめ
この演習3では、GUIで実施したLTM設定を CLI（tmsh）で再現した。  
`tmsh` で設定することで、Node / Pool / Monitor / Virtual Server / Profile の構造が理解しやすくなり、障害切り分けや運用変更時の対応力向上につながる。

また、今回は **踏み台サーバ経由で BIG-IP の管理用インターフェースに admin ユーザーでSSH接続**する構成を前提とし、実務に近い管理手順で操作を行った。
