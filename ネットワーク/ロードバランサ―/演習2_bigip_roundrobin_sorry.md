# 演習2：BIG-IPで均等分散（Round Robin）＋ sorryページ表示 設定手順

## 0. 前提
- 演習1（Node / Pool / Virtual Server の作成）が完了していること
- 各Webサーバ（Pool Member）で Apache が稼働していること
- BIG-IP から各Webサーバへのヘルスチェックが **Healthy（green）** であること
- Virtual Server（VIP / EIP経由）が作成済みであること

---

## 1. 各Webサーバへ均等に振り分ける（Round Robin）

### 1-1. 各Webサーバの表示内容を分ける（確認しやすくする）
各Webサーバで、`index.html` にサーバ名などを入れておく。

### Webサーバ1
```bash
echo web01 | sudo tee /var/www/html/index.html
sudo chmod 644 /var/www/html/index.html
```

### Webサーバ2
```bash
echo web02 | sudo tee /var/www/html/index.html
sudo chmod 644 /var/www/html/index.html
```

> 台数が3台以上ある場合も同様に、`web03` など識別できる内容にする。

---

### 1-2. Pool の負荷分散方式を Round Robin に設定
#### BIG-IP GUI
`Local Traffic` → `Pools` → 対象Pool を選択

#### 設定項目
- **Load Balancing Method**：`Round Robin`

> Round Robin は、Pool Member に順番にリクエストを振り分ける方式。

---

### 1-3. 均等分散の確認
ブラウザで LB の EIP にアクセスする、または `curl` を複数回実行して確認する。

```bash
curl http://<LBのEIP>
curl http://<LBのEIP>
curl http://<LBのEIP>
```

#### 期待結果
- `web01`
- `web02`
- `web01`
- `web02`
のように、交互（または均等に近い形）で表示される。

> **注意**  
> ブラウザのキャッシュ / Keep-Alive / Persistence の影響で偏って見えることがある。  
> 確認時は `curl` の利用や、ブラウザのシークレットモードを推奨。

---

## 2. 全Webサーバダウン時に sorryページを表示する

### 2-1. sorryページを用意する
どこか1台（または専用サーバ）に `sorry.html` を作成する。

```bash
echo '<h1>Sorry... ただいまメンテナンス中です</h1>' | sudo tee /var/www/html/sorry.html
sudo chmod 644 /var/www/html/sorry.html
```

#### 動作確認（作成したサーバ上）
```bash
curl -i http://127.0.0.1/sorry.html
```

---

### 2-2. HTTP Profile（Fallback Host用）を作成
#### BIG-IP GUI
`Local Traffic` → `Profiles` → `Services` → `HTTP` → `Create`

#### 設定例
- **Name**：`http_fallback_sorry`
- **Parent Profile**：`http`
- **Fallback Host**：`http://<sorryページを置いたサーバのIP>/sorry.html`

> 例：`http://172.31.1.250/sorry.html`

---

### 2-3. Virtual Server に HTTP Profile を適用
#### BIG-IP GUI
`Local Traffic` → `Virtual Servers` → 対象Virtual Server を選択

#### 設定項目
- **HTTP Profile**（または Client HTTP Profile）：`http_fallback_sorry`

保存（`Update` / `Finished`）

---

## 3. 動作確認（正常時 / 障害時）

### 3-1. 正常時（Pool Member が Up）
- LB の EIP にアクセス
- 各Webサーバのページ（`web01`, `web02` など）が均等に表示されることを確認

---

### 3-2. 障害時（全Memberを停止して sorry表示確認）
Poolの全メンバーを `Disabled` または `Forced Offline` にする。

#### BIG-IP GUI
`Local Traffic` → `Pools` → 対象Pool → `Members`

各Memberを選択して以下を実施：
- **Session**：`Disabled`（または）
- **State**：`Forced Offline`

#### 確認
ブラウザで `http://<LBのEIP>` にアクセス

#### 期待結果
- `sorry.html` に **302 リダイレクト**される
- `Sorry... ただいまメンテナンス中です` が表示される

---

## 4. 切り分けポイント（うまく動かないとき）

### 4-1. 均等分散されない
- Pool の **Load Balancing Method** が `Round Robin` になっているか
- Virtual Server に **Persistence** が設定されていないか
- ブラウザキャッシュの影響がないか（`curl` で確認）

---

### 4-2. sorryページが表示されない
- Virtual Server に **HTTP Profile（http_fallback_sorry）** が適用されているか
- Fallback Host の URL が正しいか
- sorryページを置いたサーバに BIG-IP から到達できるか
- `sorry.html` が 200 で返るか

---

### 4-3. そもそもLBのEIPで表示できない
- BIG-IP の Security Group で **80/tcp** が許可されているか
- Virtual Server の **Destination / Service Port** が正しいか
- Virtual Server の **SNAT** が `Auto Map` になっているか
- Pool Member が `green (Available)` か

---

## 5. 補足
- 今回の sorryページ表示は **HTTP Profile の Fallback Host** を使用しているため、動作は **リダイレクト（302）** になる
- URLを変えずに sorry メッセージを返したい場合は、別途 **iRule** による `HTTP::respond` を使う方法がある

---

## 6. 演習チェックリスト
- [ ] Pool の Load Balancing Method を `Round Robin` に設定した
- [ ] 各Webサーバの `index.html` に識別用文字列を設定した
- [ ] LBのEIPへアクセスし、表示が均等に切り替わることを確認した
- [ ] `sorry.html` を作成した
- [ ] `http_fallback_sorry`（HTTP Profile）を作成した
- [ ] Virtual Server に `http_fallback_sorry` を適用した
- [ ] 全Pool Memberを停止し、sorryページが表示されることを確認した
