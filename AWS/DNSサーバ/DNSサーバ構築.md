# DNSサーバ構築演習手順書（BIND / Amazon Linux 2023）

---

## 1. 演習の目的

1人1台のEC2インスタンスを使ってDNSサーバ（BIND）を構築し、  
自分で決めた `.local` ドメインのゾーンを作成する。

その後、ペア同士でお互いのDNSサーバに問い合わせを行い、  
**相手のドメイン名で名前解決できること** を確認する。

---

## 2. 演習内容（課題）

- 用意するもの：**1人1台 EC2インスタンス**
- DNSサーバを構築して、自分で定義したドメインでzoneを作る
- ドメイン名は命名規則に従うこと（`.local` を使う）
  - 例：`kuramochi.local` / `midori.local`
- zone作成後、ペア同士で相手のドメイン名の名前解決ができることを確認する

### 確認コマンド例
```bash
dig @相手のプライベートIPアドレス www.kuramochi.local
```

---

## 3. 先に理解しておくこと（PC①/PC②の役割）

この演習では、**PC①・PC②ともに同じ作業** を行います。  
（2人とも、自分のEC2上にDNSサーバを構築します）

### PC①がやること
- 自分のDNSサーバを構築する
- 自分のドメインを作る
- PC②からの問い合わせに答えられるようにする
- PC②のDNSサーバへ `dig` で問い合わせる

### PC②がやること
- 自分のDNSサーバを構築する
- 自分のドメインを作る
- PC①からの問い合わせに答えられるようにする
- PC①のDNSサーバへ `dig` で問い合わせる

✅ つまり、**2人とも「DNSサーバ構築担当」** です。

---

## 4. ゴール（達成条件）

- `named` サービスが起動している
- 自分のゾーン（例：`midori.local`）が作成されている
- `dig @localhost www.自分のドメイン.local` でAレコードが返る
- ペアの相手から `dig @自分のプライベートIP www.自分のドメイン.local` が成功する
- 自分も相手のDNSサーバへ問い合わせて成功する

---

## 5. 事前準備（重要）

### 5-1. セキュリティグループ（インバウンドルール）

DNSは **53番ポート（UDP/TCP）** を使います。  
ペア相手から問い合わせできるように、以下を許可してください。

- SSH（22/TCP） … 自分のIP
- DNS（53/UDP） … ペアのEC2プライベートIP（または同一VPC）
- DNS（53/TCP） … ペアのEC2プライベートIP（または同一VPC）

> DNSは通常UDPを使うことが多いですが、応答サイズによってはTCPも使うため、**UDP/TCP両方**を開けます。

---

## 6. 作業前のメモ（記入欄）

### 自分（PC① または PC②）
- ドメイン名：`__________________.local`
- プライベートIP：`__________________`

### 相手（ペア）
- ドメイン名：`__________________.local`
- プライベートIP：`__________________`

---

## 7. PC①/PC② 共通手順（DNSサーバ構築）

> ※ 以下はPC①/PC②どちらも同じ手順です  
> ※ `kuramochi.local` や `172.31.27.197` の部分は自分の値に置き換えてください

---

### 7-1. rootユーザーへ切り替え

```bash
sudo su -
```

---

### 7-2. BIND（named）をインストール

```bash
dnf install bind -y
```

#### 確認（インストール済み確認）
```bash
rpm -qa | grep '^bind'
```

---

### 7-3. namedサービスを起動・確認・自動起動設定

```bash
systemctl start named
systemctl status named
systemctl enable named
```

- `Active: active (running)` ならOK

---

### 7-4. 起動直後の状態確認（ポート53の待受確認）

```bash
ps -ef | grep named
netstat -ln | grep 53
```

#### ポイント
初期状態では `127.0.0.1:53` のみで待ち受けしていることがあります。  
このままだと、**相手（別EC2）から問い合わせできません**。

→ 次の手順で `named.conf` を修正して、**自分のプライベートIPでも待ち受け** させます。

---

### 7-5. 設定ファイル `/etc/named.conf` のバックアップ

```bash
cp /etc/named.conf /etc/named.conf.`date +%Y%m%d-%H%M%S`.bak
ls /etc | grep named.conf
```

---

### 7-6. `/etc/named.conf` を編集（listen-on / allow-query）

```bash
vi /etc/named.conf
```

#### 変更ポイント（`options` ブロック内）

- `listen-on port 53` に **自分のプライベートIP** を追加
- `allow-query` で **VPC内からの問い合わせを許可**

#### 設定例（自分のIPに置き換える）
```conf
options {
       # listen-on port 53 { 127.0.0.1; 172.31.27.197; };
       # listen-on-v6 port 53 { ::1; };

        allow-query     { any; };

};
```

#### 用語メモ（初心者向け）
- `listen-on`：どのIPでDNSを受け付けるか
- `allow-query`：どの端末からの問い合わせを許可するか

---

### 7-7. `named.conf` の構文チェック

```bash
named-checkconf
```

- 何も表示されなければOK

---

### 7-8. namedを再起動して反映

```bash
systemctl restart named
systemctl status named
```

---

### 7-9. 自分のプライベートIPで53番を待ち受けしているか確認

```bash
netstat -ln | grep 53
```

#### 確認ポイント
以下のように、**自分のプライベートIP:53** が表示されていればOK

例：
- `172.31.27.197:53`（TCP）
- `172.31.27.197:53`（UDP）

---

### 7-10. ゾーン定義を `/etc/named.conf` に追加

```bash
vi /etc/named.conf
```

#### ファイル末尾に追加（例：`kuramochi.local` の場合）
```conf
zone "kuramochi.local" IN {
        type master;
        file "/var/named/kuramochi.local.zone";
};
```

> `kuramochi.local` は自分のドメイン名に置き換えてください

---

### 7-11. ゾーンファイルを作成

```bash
vi /var/named/kuramochi.local.zone
```

#### 設定例（自分のドメイン/IPに置き換える）
```zone
$TTL 3600
@   IN SOA  ns.kuramochi.local. root.kuramochi.local. (
        2026021901 ; serial
        3600       ; refresh
        900        ; retry
        604800     ; expire
        86400      ; minimum
)

    IN NS   ns.kuramochi.local.

ns  IN A    172.31.27.197
www IN A    172.31.27.197
```

#### レコードの意味
- `SOA`：ゾーンの基本情報
- `NS`：このゾーンを管理するDNSサーバ
- `A`：ホスト名とIPアドレスの対応

---

### 7-12. ゾーンファイルの構文チェック

```bash
named-checkzone kuramochi.local /var/named/kuramochi.local.zone
```

- `OK` と表示されればOK

---

### 7-13. namedを再起動

```bash
systemctl restart named
systemctl status named
```

---

### 7-14. 自分のサーバ内で動作確認（localhost向け）

#### `www` レコード確認
```bash
dig @localhost www.kuramochi.local
```

#### `ns` レコード確認
```bash
dig @localhost ns.kuramochi.local
```

#### 成功時の見方
`ANSWER SECTION` にAレコードが返っていればOK

例：
```text
www.kuramochi.local.    3600    IN  A   172.31.27.197
```

---

## 8. 相互確認手順（PC① / PC②）

ここが演習のゴールです。  
お互いに **プライベートIP** と **ドメイン名** を共有して確認します。

---

### 8-1. PC1 → PC2へ問い合わせ

#### PC1で実行
```bash
dig @<PC②のプライベートIP> www.<PC②のドメイン名>.local
```

#### 成功条件
`ANSWER SECTION` に PC②のIP が返る

---

### 8-2. PC2 → PC1へ問い合わせ

#### PC2で実行
```bash
dig @<PC1のプライベートIP> www.<PC1のドメイン名>.local
```

#### 成功条件
`ANSWER SECTION` に PC①のIP が返る

---

## 9. よくあるエラーと確認ポイント（トラブルシュート）

---

### 9-1. `dig` がタイムアウトする

#### 主な原因
- セキュリティグループで `53/UDP` または `53/TCP` が未許可
- `listen-on` に自分のプライベートIPが入っていない
- `allow-query` が狭すぎる

#### 確認コマンド
```bash
netstat -ln | grep 53
```

→ 自分のプライベートIP:53 が出ているか確認

---

### 9-2. `named-checkconf` はOKだが起動しない

#### 主な原因
- ゾーンファイルの文法ミス
- ゾーンファイルの権限ミス

#### 確認コマンド
```bash
named-checkzone 自分のドメイン.local /var/named/自分のドメイン.local.zone
journalctl -u named -n 50 --no-pager
```

---

### 9-3. `.local` の警告が出る

`dig` 実行時に次の警告が出ることがあります：

```text
WARNING: .local is reserved for Multicast DNS
```

#### 理由
`.local` は本来 mDNS 用の予約ドメインのため

#### 今回の演習では？
✅ **警告は出てもOK**（回答が返っていれば成功）

---

### 9-4. ゾーンファイルを直したのに反映されない

ゾーンファイルを変更したら、`SOA` の `serial` を増やすのが基本です。

例：
- 修正前：`2026021901`
- 修正後：`2026021902`

---

## 10. 参考コマンド集（よく使う確認用）

```bash
# named設定の構文チェック
named-checkconf

# ゾーンファイルの構文チェック
named-checkzone 自分のドメイン.local /var/named/自分のドメイン.local.zone

# サービス状態確認
systemctl status named

# ポート53の待受確認
netstat -ln | grep 53

# ログ確認（エラー調査）
journalctl -u named -n 100 --no-pager

# localhost向け名前解決確認
dig @localhost www.自分のドメイン.local

# 相手DNSサーバへ問い合わせ
dig @相手のプライベートIP www.相手のドメイン.local
```

---

## 11. 完成チェック（最終確認シート）

### 自分のDNSサーバ
- [ ] `bind` インストール済み
- [ ] `named` が起動している
- [ ] `listen-on` に自分のプライベートIPを設定した
- [ ] ゾーン定義を追加した
- [ ] ゾーンファイルを作成した
- [ ] `named-checkconf` がOK
- [ ] `named-checkzone` がOK
- [ ] `dig @localhost` で名前解決できる

### ペア確認
- [ ] PC① → PC② の名前解決が成功
- [ ] PC② → PC① の名前解決が成功

---

## 12. 補足（学習ポイント）

この演習で学べること：

- DNSサーバ（BIND）の基本構築
- `named.conf` の役割（待ち受け・問い合わせ許可）
- ゾーンファイルの書き方（SOA / NS / A）
- `dig` を使った名前解決テスト
- セキュリティグループとネットワーク疎通の関係

---

以上
