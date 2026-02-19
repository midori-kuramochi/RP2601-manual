

### クライアント側の設定
1. fstabファイルのバックアップ作成
<code> # cp /etc/fstab /etc/fstab.`date "+%Y%m
%d_%H%M%S"`.bak </code>

1. fstabの設定編集  
   <code> # vi /etc/fstab </code>  
   >[NFSサーバのプライベートIPアドレス]:/share /yakiniku nfs4 defaults 0 0

1. マウント  
   <code> # mount /yakiniku </code>

1. NFS領域にマウントされているか確認する
   <code> # df </code>  
   >Filesystem           1K-blocks    Used Available Use% Mounted on
devtmpfs                  4096       0      4096   0% /dev  
tmpfs                   469420       0    469420   0% /dev/shm  
tmpfs                   187768     440    187328   1% /run  
/dev/nvme0n1p1         8310764 1578560   6732204  19% /  
tmpfs                   469420       0    469420   0% /tmp  
/dev/nvme0n1p128         10202    1318      8884  13% /boot/efi  
tmpfs                    93884       0     93884   0% /run/user/1000  
172.31.21.155:/share   8310784 1578496   6732288  19% /yakiniku  

1. ファイルを作成し、NFSサーバ側でファイルが作成されているか確認  
   <code> # touch /yakiniku/test.txt </code>


## 演習
### NFS-1


### NFS-2
### 演習全体の流れ（概要）
- 最終目標：Outlookからメールサーバにメールを飛ばす
- 流れ
  - DNSサーバ担当  
     1. 講師からサブドメインを発行してもらう
     1. BINDでDNS構築
     1. NFSサーバも同じEC2で構築
     1. メールサーバ用サブドメインの委譲やNS/A/MXレコード設定
     1. 名前解決（localhost→グローバル）
         <code> # dig @localhost teamg.entrycl.net MX </code>  
          <code> # dip teamg.entrycl.net MX </code>
         >
  -  メールサーバ担当（4台）
     1. Postfix構築（SMTP受信）
     2. Dovecot構築（IMAP/POP）
     3. メールスプールディレクトリをNFSにマウント
     4. 動作確認（telnet＋Outlook）

### 前提条件
- OS：Amazon Linux 2023
- VPC内でプライベートIPを使う
- ドメイン：teamg.entrycl.net
- DNSサーバIP：172.31.31.179
- メールサーバ名：  
  mail-1.teamg.entrycl.net → 35.11.11.11  
  mail-2.teamg.entrycl.net → 35.22.22.22  
  mail-3.teamg.entrycl.net → 35.33.33.33  
  mail-4.teamg.entrycl.net → 35.44.44.44  
- NFS共有ディレクトリ：/share_teamg  
  
## ステップ1：DNS + NFSサーバ構築

### セキュリティグループ(DNSサーバ)
- ssh        22      MYIP
- DNS(UDP)   53      0.0.0.0/0
- SMTP       25      0.0.0.0/0
- NFS        2049    172.31.0.0/16(VPCCIDR)
- ICMP               172.31.0.0/16(VPCCIDR)

### 1-1. BINDインストール

1. bindインストール  
   <code> # dnf install bind </code>  

### 1-2. BIND設定ファイル編集

1. BIND設定ファイル編集<named.conf.>    
   1. バックアップ作成    
      <code># cp /etc/named.conf /etc/named.conf.`date "+%Y%m%d_%H%M%S"`.bak </code>  
    1. bindの設定ファイルの編集  
   <code> # vi /etc/named.conf </code>  
   ▼ 下記参照    
     >options {  
        #listen-on port 53 { 127.0.0.1; };  <コメントアウト>  
        #listen-on-v6 port 53 { ::1; };  <コメントアウト>  
        ...  
        allow-query     { any; };  
        #allow-query     { localhost; };  <コメントアウト>  
        ---  
        ↓末尾に追記  
        zone "teamg.entrycl.net" IN {  
        type master;  
        file "/var/named/teamg.entrycl.net.zone";  
        };  
        
    2. 間違いがないか確認  
       <code> # named-checkconf </code>  

2. ゾーンファイル作成 </var/named/teamg.entrycl.net.zone>  
   <code> # vi /var/named/teamg.entrycl.net.zone </code>
   >$TTL 3600  
   @ IN SOA ns.teamg.entrycl.net. test.teamg.entrycl.net. (  
   20210401 ; serial  
   3600 ; refresh  
   3600 ; retry  
   3600 ; expire  
   3600 ) ; minimum  

        IN  NS   ns.teamg.entrycl.net.
     ns IN  A    [VPCサーバのパブリックIP]  
     ; MXレコード  
     @   IN  MX 10 mail-1.teamg.entrycl.net.  
     @   IN  MX 10 mail-2.teamg.entrycl.net.  
     @   IN  MX 10 mail-3.teamg.entrycl.net.  
     @   IN  MX 10 mail-4.teamg.entrycl.net.  
     ; Aレコード  
     mail-1  IN  A  35.11.11.11  
     mail-2  IN  A  35.22.22.22  
     mail-3  IN  A  35.33.33.33  
     mail-4  IN  A  35.44.44.44  
  >MXの数字は「優先度（小さいほど優先）」

1. 間違いがないか確認    
   <code> # named-checkzone teamg.entrycl.net /var/named/teamg.entrycl.net.zone
 </code>  

1. DNS参照先サーバーの設定   
   <code> # vi /etc/systemd/resolved.conf </code>  
   ▼下記修正    
   >#DNS=   
   ↓コメントアウトを外す  
   DNS=[DNSサーバーのプライベートIPアドレス]
2. 設定の反映（サービスの再起動）  
   <code> # systemctl restart systemd-resolved.service</code>  

3. named起動  
   <code> # systemctl start named </code>    
   <code> # systemctl status named </code> 

4. 動作確認  
   1. localhostに向けて名前解決  
   <code> # dig @[DNSサーバIP] teamg.entrycl.net MX</code>   
   <code> # dig @[DNSサーバIP] mail-1.teamg.entrycl.net A</code> 
   2. グローバルに向けて名前解決  
   <code> # dig teamg.entrycl.net MX</code>   
   <code> # dig mail-1.teamg.entrycl.net A</code>  
     >最終的にkuramochi@teamg.entrycl.netに向けてメール送りたいので、@より右側のteamx.entrycl.netのＭＸレコードがグローバルに名前解決できる必要があるため。
     具体的なゾーンファイルは以下のようになる  
     teamx.entrycl.net　＝　mail.teamx.entrycl.net  
     mail.teamx.entrycl.net = メールサーバのグローバルＩＰアドレス  
     ◎なぜグローバルＩＰアドレスなのか、というと、このゾーンファイルをoutlookが見たうえでメールをサーバに送信するから.  
     outlookはＶＰＣ外にあるのでグローバルＩＰでないと特定できません。
     <!-- 
     💻[あなたのPC (Outlook)]（社内LAN / Wi-Fi）
        │  
        ▼
     🏠[社内ルータ / FW / NAT]
        │
        ▼
     🌍 インターネット（WAN）
        │
        ▼
     　[AWSのパブリックIP]
        │
        ▼
     📩[mail-*.teamg.entrycl.net (EC2)] 
     -->


### 1-3. NFSサーバ構築  
1. 共有ディレクトリ作成  
   <code> # mkdir /share_teamg </code>

2. exportsの設定  
   <code> # cp /etc/exports /etc/exports.`date "+%Y%m%d_%H%M%S"`.bak </code>  
   <code> # vi /etc/exports </code>  
   下記を入力  
   /share_teamg 172.31.0.0/16(rw,sync,no_root_squash)

3. NFS設定反映  
   <code> # systemctl restart nfs-server</code>  
   <code> # systemctl enable nfs-server</code>  
   <code> # exportfs -a</code>  



----


## ステップ2：メールサーバ構築（Postfix）
※メールサーバ側

### セキュリティグループ(メールサーバ)
- SSH       22     MYIP
- DNS(UDP)  53     0.0.0.0/0
- SMTP      25     0.0.0.0/0
- POP3      110    172.31.0.0/16(VPCCIDR)
- NFS       2049 　172.31.0.0/16(VPCCIDR)
- ICMP             172.31.0.0/16(VPCCIDR)

### 2-1. postfix, mailxのインストール
1. インストール  
<code> $ sudo su - </code>  
<code> # dnf install postfix </code>

2. 起動  
   <code> # systemctl start postfix </code>  
   <code> # systemctl status postfix </code>  
   <code> # ps -ef | grep postfix </code>   
   <code> # netstat -ln | grep 25 </code>   

2. mailxインストール  
   <code> # dnf install mailx </code>  
   mailx:コマンドライン上でメールを送るシステム

### 2-2Postfixの設定
postfixの基本的な設定ファイル：/etc/postfix/main.cf  

1. バックアップ作成  
   <code># cp /etc/postfix/main.cf /etc/postfix/main.cf`date "+%Y%m%d_%H%M%S"`.bak </code>
2. 確認   
   <code> # ll /etc/postfix/ | grep main.cf </code>  

3. /etc/postfix/main.cfのコメントアウト行削除→/tmp/main.cfに上書き  
   <code> # grep -v ^# /etc/postfix/main.cf | cat -s > /tmp/main.cf </code>  
   <code> # cp /tmp/main.cf /etc/postfix/main.cf </code>  
   <code> # less /etc/postfix/main.cf </code>  

4. postfixの設定ファイル編集  
   <code> # vi /etc/postfix/main.cf </code>  
   ▼追記  
   myhostname = mail-1.teamg.entrycl.net   # 各サーバで変更
   mydomain = teamg.entrycl.net
   myorigin = $mydomain
   mynetworks = 172.31.0.0/16, 127.0.0.1
   mail_spool_directory = /var/spool/mail/ 
   ▼編集
   inet_interfaces = all
   mydestination = $mydomain, $myhostname

5. 再起動  
   <code> # systemctl restart postfix </code>  
   <code> # systemctl enable postfix </code>  

### 2-3 メールサーバ構築（Dovecot）
1. Dovecotのインストール  
   <code> # dnf install dovecot </code>
1. dovecot.confのバックアップ作成  
   <code> # cp /etc/dovecot/dovecot.conf /etc/dovecot/dovecot.conf.`date "+%Y%m%d_%H%M%S"`.bak </code>  
1. dovecotの設定  
   <code> # vi /etc/dovecot/dovecot.conf </code>  
   >1. protocols = imap pop3 lmtp submission  
   ↓  
   protocols = pop3  に変更
   
   >2. 末尾に追加  
   mail_location = maildir:/var/spool/mail/%u

2. 10-ssl.conf | バックアップ  
   <code> # cp /etc/dovecot/conf.d/10-ssl.conf /etc/dovecot/conf.d/10-ssl.conf.`date "+%Y%m%d_%H%M%S"`.bak </code>  
3. 10-ssl.confの設定編集  
   <code> # vi /etc/dovecot/conf.d/10-ssl.conf </conf>
   >ssl = required だけをコメントアウト

4. 10-auth.conf | バックアップ  
   <code> # cp /etc/dovecot/conf.d/10-auth.conf /etc/dovecot/conf.d/10-auth.conf.`date "+%Y%m%d_%H%M%S"`.bak </code>  
5. 10-auth.confの設定編集  
   <code> # vi /etc/dovecot/conf.d/10-auth.conf </conf>  
   > #disable_plaintext_auth = yes だけ   
   →  
   disable_plaintext_auth = no [※コメントアウトを外す]

6. dovecot起動  
   <code> # systemctl start dovecot </code>  
   <code> # systemctl status dovecot </code>

7. dovecot自動起動設定  
   <code> # systemctl enable dovecot </code> 

1. DNS参照先サーバーの設定   
   <code> # vi /etc/systemd/resolved.conf </code>  
   ▼下記修正    
   >#DNS=   
   ↓コメントアウトを外す  
   DNS=[DNSサーバーのプライベートIPアドレス]

2. 設定の反映（サービスの再起動）  
   <code> # systemctl restart systemd-resolved.service</code>  

4. 動作確認  
   1. localhostに向けて名前解決  
   <code> dig @[DNSサーバIP] teamg.entrycl.net MX</code>     
   <code> dig @[DNSサーバIP] mail-1.teamg.entrycl.net A</code>   
   2. グローバルに向けて名前解決  
   <code> dig teamg.entrycl.net MX</code>     
   <code> dig mail-1.teamg.entrycl.net A</code>    
   

1. ユーザー追加  
   <code> # useradd midori -g mail -M -K MAIL_DIR=/dev/null -s /sbin/nologin </code>   
   <code> # passwd midori </code>
   >useradd で追加した場合、1001番で番号が振られる
    zone番号のMXで優先順位を付けた場合、全員同じ番号にならないとダメ

2. telnetインストール  
   <code> # dnf install telnet </code>  
   <code> # telnet localhost 110 </code>  
   user midori  
   +OK  
   pass 12345  

## メールが送れるか確認
1. 自分のrootユーザー宛てに送信  
   <code> # mail -s kennmei root@mail-kuramochi.teamg.entrycl.net </code>
   <code> # mail </code>
2. 自分のmidoriユーザー宛てに送信
   <code> # mail -s kennmei midori@mail-kuramochi.teamg.entrycl.net </code>
3. telnetでメール確認
   <code> telnet localhost 110 </code>
   user midori
   pass 12345
   list で見る
   retr 番号
   * <code> mail </code>でもOK！

## 3. メールスプールをNFSへ変更
目的:各サーバのメール保存先を /share_teamg に統一

1. fstabの設定編集  
   <code> # vi /etc/fstab </code>  
   >[NFSサーバのプライベートIPアドレス]:/share_teamg /var/spool/mail nfs4 defaults 0 0

2. マウント  
   <code> # mount /var/spool/mail </code>

3. NFS領域にマウントされているか確認する
   <code> # df </code> 

4. メールを送ると自動的にユーザーのディレクトリが作成される
   <code> # ls /var/spool/mail　</code>  
   (例) root  test.text  yoshie midori  ryotaro yuki 

## 4. メールが送れるか確認  
Outlookのアドレスからメールを送信    
midori@mail-kuramochi.teamg.entrycl.net  
宛てにメールを送信！  
<code> # ls /var/spool/mail　</code>  
 (例) root  test.text  yoshie midori  ryotaro yuki   
<code> # ls /var/spool/mail/midori　</code>  
new cur tmp  
<code> # ls /var/spool/mail/new　</code>  
届いているメールの情報  
<code> # cat /var/spool/mail/new/届いているメールの情報　</code>
で見れます！