# f5 BigIP


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
