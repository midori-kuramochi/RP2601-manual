# f5 BigIP GUIへのアクセス方法


BigIPのGUIへの接続 
ssh -L 10443:172.31.2.20:443 -i ./.ssh/CL_kuramochi.pem ec2-user@踏み台サーバIP


````markdown
## BIG-IP GUIへ接続（SSHポートフォワード）

### 目的
踏み台サーバを経由して、**BIG-IP（172.31.2.20）のGUI（HTTPS/443）** に自PCからアクセスできるようにする。

---

### 実行コマンド
```bash
ssh -L 10443:172.31.2.20:443 -i ./.ssh/CL_kuramochi.pem ec2-user@踏み台サーバIP
````

---

### コマンドの意味（分解）

* `ssh ec2-user@踏み台サーバIP`
  → 踏み台サーバへ `ec2-user` でSSH接続する

* `-i ./.ssh/CL_kuramochi.pem`
  → SSH接続に使う秘密鍵（キーペア）を指定する

* `-L 10443:172.31.2.20:443`
  → ローカルポートフォワードを設定する

  * **自PCの `localhost:10443`** に来た通信を
  * **踏み台経由で `172.31.2.20:443`（BIG-IP GUI）** へ転送する

通信の流れ：
`自PC (localhost:10443) → SSHトンネル → 踏み台 → BIG-IP (172.31.2.20:443)`

---

### GUIへのアクセス方法

上記コマンドを実行して **SSH接続したまま**、ブラウザで以下にアクセスする：

* `https://localhost:10443`

---

### 注意事項

* SSH接続（このターミナル）を閉じるとトンネルも切れるため、GUIにアクセスできなくなる
* 踏み台サーバからBIG-IPへ **TCP 443** が到達できるように、SG/NACLの許可が必要
* 証明書が自己署名の場合、ブラウザで警告が出ることがある（演習ではよくある）

```
::contentReference[oaicite:0]{index=0}
```

---

## 🎯 目的
```
PublicでApacheをインストール
↓
その状態をAMI化
↓
Private Subnetで起動
```
---
</br>


## 🏗 全体構成イメージ
```
① Public Subnet
   EC2（Apacheインストール済）
        ↓
   AMI作成
        ↓
② Private Subnet
   AMIから新規EC2起動
```
---
</br>


## 🪜 手順（実務レベル）
**① Public SubnetでEC2作成**
- Public IPあり
- IGW接続あり
- SSH可能

**② Apacheインストール**</br>
Amazon Linux 2023 の場合：
```bash
sudo dnf update -y
sudo dnf install httpd -y
sudo systemctl enable httpd
sudo systemctl start httpd
```

動作確認：
```bash
http://<PublicIP>
```

**③ AMIを作成**</br>
AWSコンソール：
```bash
EC2
↓
対象インスタンス選択
↓
アクション
↓
イメージとテンプレート
↓
イメージを作成
```
ポイント：
- 「再起動」推奨（整合性のため）
- 数分待つ

**④ Private SubnetでEC2起動**
```
EC2起動
↓
AMI選択（自分が作成したもの）
↓
サブネット：Privateを指定
↓
Public IP：無効
```

### 💡 重要ポイント（プロ視点）
**セキュリティグループ**</br>
**Private側では：**
- SSHは踏み台からのみ許可
- 80番はALBやAPからのみ許可

**ApacheのListen確認**

念のため：
```bash
sudo vi /etc/httpd/conf/httpd.conf
```
```bash
Listen 80
```
→ OK

---
</br>
</br>


## webサーバ動作確認
踏み台サーバにssh接続
```bash
ssh -i ~/.ssh/entrycl_202601.pem ec2-user@16.144.198.187
```
プライペートサブネットのwebサーバに対して、httpアクセス
```bash
curl http://172.31.1.206/
```
※ webサーバ側のSGに踏み台サーバからのアクセスができるように設定しておく

ヘルスチェック用のファイルを作成
```bash
sudo vi /var/www/html/index.html
sudo chown apache:apache /var/www/html/index.html
```
**SG設定**
| プロトコル | ポート | ソース        |
|----------|-------|--------------|
| ssh      | 22    | マイIP        |
| http     | 80    | 自ネットワーク(172.31.0.0/16) |
| icmp     | 全て   | 自ネットワーク(172.31.0.0/16) |
---
</br>
</br>


## ノード作成
BigIp F5 に接続
```bash
ssh -L 10443:172.31.2.20:443 -i ~/.ssh/entrycl_202601.pem ec2-user@16.144.198.187
```
ブラウザでアクセス（ターミナルは閉じない）
```bash
https://127.0.0.1:10443
```
以下を設定
```
Local Traffic
↓
Nodes
↓
Node list
↓
ボタン「Create」
↓
Name: 任意
Address: プライベートIP
Health: default
↓
Finished
```
---
</br>
</br>



## Pool作成
以下を設定
```
Local Traffic
↓
Pools
↓
Pool list
↓
ボタン「Create」
↓
Name: 任意
Health Monitors: httpを追加
Lord Balancing Method: Roud Robin
New Member: Node listでノードを追加
↓
Finished
```
---
</br>
</br>



## Virtual Server設定
EIP作成 & 関連付け
```
① LBサーバのインターフェース(Pub)に対して、プライベートIPを複製
↓
② ElasticIP作成
↓
③ ElasticIPを①に関連付け
```
f5 で `Virtual Server` を作成
```
Name: 任意
Destination Address/Mask: 172.31.0.100(ElasticIPを①に関連付けたプライベートIP)
HTTP Profile: http
Source Address Translation: AutoMap
```
---
</br>
</br>



## 動作確認
ヘルスチェック確認(webサーバにssh接続)
```bash
sudo tail /var/log/httpd/access_log
```
ブラウザ確認
```bash
# http://<ElasticIP>/
http://52.43.35.56/
or
http://52.43.35.56/index.html
```
---
</br>
</br>
</br>


# TMOSで操作
## ログイン
```bash
#ssh <ユーザ名>@<BigIPの管理IP(BigIPがインストールされているEC2のmngインターフェース)>
ssh admin@172.31.2.20
```
▼ パスワードを入力
```bash
(admin@172.31.2.20) Password: 
```
▼ ログイン後
```
Last login: Tue Feb 24 09:55:50 2026 from 172.31.2.131
admin@(ip-172-31-2-20)(cfg-sync Standalone)(Active)(/Common)(tmos)#
```
</br>

## node作成
▼ 作成
```bash
create ltm node CL_kurosawa2 address 172.31.1.206
```
▼ 確認
```bash
list ltm node CL_kurosawa2
```
▼ 結果
```bash
ltm node CL_kurosawa2 {
    address 172.31.1.206
}
```
</br>

## pool作成
▼ 作成
```bash
create ltm pool CL_teamE2 members add { CL_kurosawa:80 CL_kuramochi:80 CL_maruyama:80 CL_sato_h:80 CL_watanabe_node:80 }
```
▼ 存在確認
```bash
list ltm pool CL_teamE2
```
▼ メンバー確認
```bash
list ltm pool CL_teamE2 members
```
</br>

## VirtualServer作成
▼ 作成
```bash
create ltm virtual teamE destination 172.31.0.101:http ip-protocol tcp
```
▼ 確認
```bash
list ltm virtual teamE
```




