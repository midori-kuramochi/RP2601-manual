# iRuleとは・使い方まとめ（BIG-IP / F5）

## 1. iRuleとは
iRule（アイルール）は、**F5 BIG-IP 上で動作するカスタム制御ルール**です。  
BIG-IP の標準設定（Pool / Monitor / Virtual Server / Profile など）だけでは実現しにくい要件を、**条件分岐つきのルール**として追加できます。

- ベース言語：**Tcl**
- 適用先：主に **Virtual Server**
- 目的：L4/L7 の通信制御を柔軟にカスタマイズすること

---

## 2. iRuleでできること（代表例）
### 2-1. 振り分け制御
- 特定URLだけ別Poolへ振り分ける
- 特定Hostヘッダで振り分ける
- 特定の拡張子（`.jpg` / `.css` など）だけ別サーバへ送る

### 2-2. アクセス制御
- 特定IPアドレスからのアクセスを拒否
- User-Agent を見て制御
- 特定メソッド（PUT/DELETE）を拒否

### 2-3. 応答の書き換え
- BIG-IPが直接レスポンスを返す（例：503メンテナンスページ）
- HTTPヘッダの追加・削除
- リダイレクト（302）を返す

### 2-4. ログ出力
- 条件に応じてBIG-IP側にログを出す（切り分け用）

---

## 3. まず覚える考え方（超重要）
iRuleは、**「イベントが起きたときに、何をするか」** を書きます。

基本形：
```tcl
when <イベント名> {
    # 実行したい処理
}
```

### よく使うイベント（HTTP系）
- `HTTP_REQUEST`
  - クライアントからHTTPリクエストを受けたとき
- `HTTP_RESPONSE`
  - サーバからHTTPレスポンスが返ってきたとき

---

## 4. 使い方（基本の流れ）
## 4-1. iRuleを作成する（GUI）
1. **Local Traffic** → **iRules** → **iRule List**
2. **Create**
3. 以下を入力
   - **Name**：任意（例：`irule_test`）
   - **Definition**：iRule本体（Tcl）
4. **Finished**

---

## 4-2. Virtual Server に適用する
1. **Local Traffic** → **Virtual Servers** → 対象VS
2. **Resources**（または iRules セクション）
3. 作成したiRuleを **Enabled / Active** に追加
4. **Update**

> iRuleを作るだけでは動かず、**Virtual Serverへ適用して初めて有効**になります。

---

## 5. iRuleの基本サンプル（よく使う）
## 5-1. 動作確認用（アクセスが来たらログを出す）
```tcl
when HTTP_REQUEST {
    log local0. "HTTP request received: [HTTP::method] [HTTP::uri]"
}
```

### 補足
- BIG-IPのログに出力されるため、まずは「iRuleが動いているか」の確認に便利

---

## 5-2. 特定パスだけ別Poolへ振り分ける
```tcl
when HTTP_REQUEST {
    if { [HTTP::uri] starts_with "/api/" } {
        pool pool_api
    } else {
        pool pool_web
    }
}
```

### 例
- `/api/users` → `pool_api`
- `/index.html` → `pool_web`

---

## 5-3. 特定IPを拒否する（例）
```tcl
when HTTP_REQUEST {
    if { [IP::client_addr] eq "203.0.113.10" } {
        reject
    }
}
```

---

## 5-4. BIG-IPが直接メッセージを返す（メンテナンス表示の例）
```tcl
when HTTP_REQUEST {
    HTTP::respond 503 content "<html><body><h1>Maintenance</h1><p>ただいまメンテナンス中です。</p></body></html>" "Content-Type" "text/html; charset=UTF-8"
}
```

### ポイント
- これは **常に 503 を返す** 例
- 実際は条件をつけて使うことが多い（時間帯、URL、メンテフラグなど）

---

## 6. GUIではなくCLI（tmsh）で作成・適用する方法（参考）
## 6-1. iRule作成
```tcl
create ltm rule irule_test {
    when HTTP_REQUEST {
        log local0. "Hello from iRule"
    }
}
```

## 6-2. Virtual Serverへ適用
```tcl
modify ltm virtual vs_http_web rules { irule_test }
```

### 確認
```tcl
list ltm rule irule_test
list ltm virtual vs_http_web
```

---

## 7. iRuleを書くときの注意点（大事）
## 7-1. まずは標準機能を優先
iRuleは強力ですが、複雑になりやすいです。  
まずは以下の標準機能で実現できるか確認するのが基本です。

- Pool
- Health Monitor
- HTTP Profile
- Persistence
- Local Traffic Policy

> **標準機能でできるなら、iRuleを使わない** のが運用しやすいです。

---

## 7-2. 影響範囲を意識する
iRuleは Virtual Server に適用されるため、対象VSに来る通信すべてに影響します。  
意図しない動作にならないよう、条件を絞って書くのがポイントです。

---

## 7-3. ログを入れて切り分けしやすくする
動作確認時は `log local0.` を一時的に入れると、どこまで処理されたか確認しやすくなります。

---

## 7-4. 文字列比較・条件分岐は丁寧に
URLやHostヘッダの条件が曖昧だと誤動作しやすいため、以下を意識します。

- `starts_with` / `contains` / `equals` を使い分ける
- 大文字小文字の違いに注意
- 先に除外条件を書くと読みやすい

---

## 8. よくあるハマりどころ
### 8-1. iRuleを作ったのに動かない
- Virtual Serverに適用していない
- 適用したVSが違う
- イベント名が間違っている（`HTTP_REQUEST` など）

### 8-2. 構文エラーになる
- 波かっこ `{}` の対応ミス
- ダブルクォート `"` の閉じ忘れ
- Tcl構文の書き方ミス

### 8-3. 期待した条件で分岐しない
- `HTTP::uri` と `HTTP::path` の違い
- Hostヘッダの値にポートが含まれている
- 条件の順番（先に広い条件を書いてしまっている）

---

## 9. まずはこれだけ覚える（初心者向け）
1. **iRule = BIG-IPのカスタム制御ルール**
2. **when HTTP_REQUEST** が入口としてよく使う
3. 作っただけでは動かない → **Virtual Serverに適用が必要**
4. 標準機能でできるなら、まずは標準機能を使う
5. 動作確認時は `log local0.` が便利

---

## 10. まとめ
iRuleは、BIG-IPの標準機能では足りない要件を実現するための強力な仕組みです。  
一方で、自由度が高いぶん複雑になりやすいため、以下の順番で考えるのがおすすめです。

1. 標準機能でできるか確認する  
2. できない場合に iRule を使う  
3. 小さく作って、ログで確認しながら適用する  

演習ではまず、`HTTP_REQUEST` でログを出す簡単なiRuleから試すと理解しやすいです。
