# 演習1：BIG-IPでの基本負荷分散構成（Node / Pool / Virtual Server）構築手順

## 0. 演習概要
チーム内の各メンバーが作成したWebサーバ（Apache）を **Node** としてBIG-IPに登録し、  
それらを **Pool** としてまとめ、**Virtual Server（EIP）** で外部からアクセスできるようにする。

また、Poolの **Load Balancing Method** を変更し、振り分け方式（分散方法）の違いを確認する。

---

## 1. 前提条件
- チーム人数：**5人**
- **BIG-IPサーバは構築済み**
- VPC構成：**サブネット3つ（Public 2 / Private 1）**
- 各メンバーのWebサーバは **Privateサブネット** に配置する
- PrivateインスタンスにApacheを入れる際は、**AMI（Apache入り）を使用する**
- BIG-IPの設定作業は、**BIG-IP GUI** で実施する

---

## 2. 構成イメージ
- **Node**：各メンバーのWebサーバ（1人1台）
- **Pool**：チーム全員分のNodeを1つのグループとしてまとめる
- **Virtual Server**：チーム用のEIPを割り当て、アクセスの受け口にする

### 例（5人構成）
- web01（メンバー1）
- web02（メンバー2）
- web03（メンバー3）
- web04（メンバー4）
- web05（メンバー5）

---

## 3. 事前準備（各メンバー）
### 3-1. Apache入りAMIからPrivateインスタンスを作成
EC2コンソール → **インスタンスを作成**

#### 設定ポイント
- **名前**：例 `teamA-web01`（各自識別できる名前）
- **AMI**：チームで用意した Apache入りAMI（例：`teamx_AMI`）
- **VPC**：BIG-IPがあるVPC
- **サブネット**：**Privateサブネット**
- **パブリックIP**：無効（Privateなので不要）
- **セキュリティグループ**：
  - SSH(22)：踏み台サーバのSG（または踏み台IP）から許可
  - HTTP(80)：BIG-IPからのアクセスを許可（BIG-IPのSG または BIG-IPのIP）

> **ポイント**  
> ApacheのインストールはAMIで事前に実施済みの前提。  
> Privateインスタンス上で `dnf install httpd` は行わない。

---

### 3-2. Webページの識別表示を設定（各サーバ）
BIG-IP経由でどのサーバに振り分けられたか確認しやすいように、`index.html` に識別文字を入れる。

#### 例（web01）
```bash
echo web01 | sudo tee /var/www/html/index.html
sudo chmod 644 /var/www/html/index.html
curl -i http://127.0.0.1/
```

#### 例（web02）
```bash
echo web02 | sudo tee /var/www/html/index.html
sudo chmod 644 /var/www/html/index.html
curl -i http://127.0.0.1/
```

※ web03 ～ web05 も同様に設定する

---

### 3-3. （任意推奨）ヘルスチェック用ページ `/health` を作成
BIG-IPのヘルスチェック用に、200 OKを返す専用ページを作る。

```bash
echo OK | sudo tee /var/www/html/health >/dev/null
sudo chmod 644 /var/www/html/health
curl -i http://127.0.0.1/health
```

---

## 4. BIG-IP GUIでの設定
> ここからは **BIG-IP GUI** で作業する。

---

## 5. Nodeの作成（各Webサーバを登録）
### 5-1. Node作成画面
**Local Traffic** → **Nodes** → **Node List** → **Create**

### 5-2. 設定（各サーバ分）
各メンバーのWebサーバをNodeとして登録する。

#### 例（web01）
- **Name**：`node_web01`
- **Address**：`<web01のPrivate IP>`

#### 例（web02）
- **Name**：`node_web02`
- **Address**：`<web02のPrivate IP>`

※ web03 ～ web05 も同様

---

## 6. Poolの作成（Nodeをまとめる）
### 6-1. Pool作成画面
**Local Traffic** → **Pools** → **Pool List** → **Create**

### 6-2. 設定例
- **Name**：`pool_team_web`
- **Health Monitors**：後から設定でも可（ここで設定してもOK）
- **Load Balancing Method**：まずは `Round Robin`（初期値のことが多い）

### 6-3. Pool Member追加
**New Members** に、各Nodeを `:80` で追加する

#### 追加例
- `<web01のPrivate IP>:80`
- `<web02のPrivate IP>:80`
- `<web03のPrivate IP>:80`
- `<web04のPrivate IP>:80`
- `<web05のPrivate IP>:80`

---

## 7. ヘルスチェック（Monitor）設定
### 7-1. Monitor作成（HTTP）
**Local Traffic** → **Monitors** → **Create**

### 7-2. 設定例（/health を使用）
- **Name**：`mon_http_health`
- **Type**：`HTTP`
- **Interval**：`5`
- **Timeout**：`16`
- **Send String**：
  ```text
  GET /health HTTP/1.1\r\nHost: localhost\r\nConnection: close\r\n\r\n
  ```
- **Receive String**：
  ```text
  200 OK
  ```

> `/health` を使わない場合は `GET /` でも可。ただし本番運用では `/health` 推奨。

---

### 7-3. PoolへMonitorを適用
**Local Traffic** → **Pools** → `pool_team_web` → **Health Monitors**

- `Available` から `mon_http_health` を選択し、`Active` に移動
- **Update**

---

## 8. Virtual Serverの作成（EIPを受け口にする）
### 8-1. 事前準備（AWS側）
- チーム用の **EIP** を払い出し
- BIG-IPの外向きNIC（ENI）に関連付ける

> EIPはAWS上の公開IP。  
> BIG-IP内のVirtual Serverは通常、**BIG-IPのPrivate IP（VIP）** を宛先に設定する。

---

### 8-2. Virtual Server作成画面
**Local Traffic** → **Virtual Servers** → **Virtual Server List** → **Create**

### 8-3. 設定例
- **Name**：`vs_team_web_http`
- **Type**：`Standard`
- **Destination Address/Mask**：`<BIG-IPの受け口にするPrivate IP>`
- **Service Port**：`80`（HTTP）
- **Default Pool**：`pool_team_web`
- **Source Address Translation (SNAT)**：`Auto Map`
- **VLAN and Tunnel Traffic**：切り分け中は `All VLANs and Tunnels` 推奨

> **重要**  
> AWS上では、ブラウザからは `http://<EIP>` でアクセスするが、BIG-IPのVS宛先は通常Private IP（VIP）を指定する。

---

## 9. 動作確認
### 9-1. Pool Memberが正常（green）であることを確認
**Local Traffic** → **Pools** → `pool_team_web` → **Members**

- 各Memberが **Available (green)** になっていること

### 9-2. Virtual Serverが正常（green）であることを確認
**Local Traffic** → **Virtual Servers**

- `vs_team_web_http` が **Available (green)** であること

### 9-3. ブラウザ / curl でアクセス確認
ブラウザでアクセス：
```text
http://<チームのEIP>
```

または `curl`：
```bash
curl http://<チームのEIP>
curl http://<チームのEIP>
curl http://<チームのEIP>
```

#### 期待結果
- `web01`, `web02`, `web03` ... のように、各サーバの表示が切り替わる
- Poolの負荷分散により、各Nodeへ振り分けられていることを確認できる

---

## 10. 分散方法の違いを確認する（Load Balancing Method変更）
Poolの **Load Balancing Method** を変更し、振り分け方の違いを確認する。

### 設定場所
**Local Traffic** → **Pools** → `pool_team_web`

### 確認する例
- **Round Robin**（均等に順番）
- **Least Connections (Member)**（接続数が少ないメンバー優先）
- **Ratio (Member)**（重み付け配分）

> `Ratio (Member)` を使う場合は、各Pool Memberの **Ratio値** を設定する必要あり。  
> 設定場所：**Pools → 対象Pool → Members → 対象Member → Ratio**

---

## 11. よくあるハマりどころ
### 11-1. Pool Memberが赤/青のまま
- Apacheが起動していない
- Node側SGで **BIG-IP → 80/tcp** が許可されていない
- Monitorの `Send/Receive String` が不正
- `/health` が 200 を返していない

---

### 11-2. Virtual ServerがOffline
- VSの **Default Pool** が未設定
- VSの **Destination / Service Port** が誤り
- VSが **Disabled**
- **SNAT** 未設定（AWS構成では `Auto Map` 推奨）

---

### 11-3. EIPでアクセスできない
- BIG-IPのSGで **80/tcp** が未許可
- EIPの関連付け先ENIが誤り
- VSのDestination（Private VIP）が、EIPに紐づく経路と一致していない

---

## 12. 演習チェックリスト
- [ ] 5台分のPrivate Webサーバ（Apache入りAMI）を作成した
- [ ] 各Webサーバの `index.html` に識別用文字列を設定した
- [ ] （任意推奨）`/health` を作成した
- [ ] BIG-IPで5台分のNodeを作成した
- [ ] Pool（`pool_team_web`）を作成し、5台をMember追加した
- [ ] HTTP Monitor（`mon_http_health`）を作成し、Poolに適用した
- [ ] EIPをBIG-IPに関連付けた
- [ ] Virtual Server（`vs_team_web_http`）を作成した
- [ ] Pool Member / Virtual Server が green であることを確認した
- [ ] EIPアクセスで各Webサーバへ振り分けられることを確認した
- [ ] Load Balancing Method を変更し、分散方法の違いを確認した
