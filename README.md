# aws_study
こちらは「Amazon Web Services基礎からのネットワーク＆サーバー構築改訂４版 」のまとめ


## Availability Zon (AZ)の意義
リージョンの中に複数のサーバー基地局(AZ)を持ち、
1つの基地局が地震や洪水などで被害を受けても他の基地局が使えるようにする  
=> 対障害性を高めるため  
なので、運用する場合は異なるAZに同じ構成のサーバーを立てたりする

## Amazon Virtial Private Cloud (AWS VPC)
仮想的なネットワーク

全体のAmazon VPCを作成して  
そのネットワークを分割する  
=> 分割したネットワークのことをサブネットという

## Amazon Elastic Compute Cloud (AWS EC2)
仮想的なサーバー

## Hands On
2. ネットワークの構築  
   - IPアドレスの割り当て
   - パブリックサブネットの構築 
3. サーバーの構築
   - EC2を使ったサーバー構築
   - ファイアウォールの設定  
4. Webサーバーのインストール
   - Apacheのインストール
   - DNSの設定
5. HTTPの動きを確認
   - デバッグツールでHTTPの解析
6. プライベートサブネットの構築
   - プラベートサブネットを利用してDBサーバーを配置
7. NATの構築
   - NATを使ってインターネットに接続できるようにする
8. DBを使ったブログの構築
   - DBサーバーをインストール
9. TCP/IP通信の仕組みの理解

### 2. ネットワークの構築  
#### プライベートIPアドレス (自由に利用可)
- 10.0.0.0 ~ 10.255.255.255  
- 172.16.0.0 ~ 172.31.255.255  
- 192.168.0.0 ~ 192.168.255.255

#### CIDR表記とサブネットマスク表記
1. CIDR表記「/24」もしくはサブネットマスク表記「/255.255.255.0」
の場合は256個分の範囲を示す  
例: 「AAA.BBB.CCC.0/24」もしくは「AAA.BBB.CCC.0/255.255.255.0」=> 「AAA.BBB.CCC.0 ~ AAA.BBB.CCC.255」

2. CIDR表記「/16」もしくはサブネットマスク表記「/255.255.0.0」
の場合は65536個分の範囲を示す  
例: 「AAA.BBB.0.0/16」もしくは「AAA.BBB.0.0/255.255.0.0」=> 「AAA.BBB.0.0 ~ AAA.BBB.255.255」

```
192.168.1.0 ~ 192.168.1.255
192.168.1.0/24 (CIDR表記)
192.168.1.0/255.255.255.0 (サブネットマスク表記)
=> これは全部同じIPアドレスの範囲を示す
```

#### サブネットの考え方
通常、VPCにCIDRブロック「10.0.0.0/16」を割り当てた場合、
それをさらに「/24」の大きさできって256分割する
```
10.0.0.0/16
(10.0.0.0 ~ 10.0.255.255)
❘
❘ 256個のサブネット(「/24」)に分割
↓
10.0.0.0/24             | 10.0.1.0/24            
(10.0.0.0 ~ 10.0.0.255) | (10.0.1.0 ~ 10.0.1.255)
...
10.0.254.0/24             | 10.0.255.0/24            
(10.0.254.0 ~ 10.0.0.255) | (10.0.255.0 ~ 10.0.255.255)
↓
65536個のIPアドレスを作成可能
(256個のサブネット * 1サブネット256範囲)
```

理由  
1. 物理的な隔離  
例：社内LANで１階と２階とで別のサブネットに分けたいとか

2. セキュリティ上の理由  
例：経理部のネットワークだけ分離して、他の部署からアクセスできなくするとか


#### VPCをサブネットに分割する
1. パブリックサブネット (10.0.1.0/24)にする  
=> インターネットからアクセス可能なサブネット。
2. プライベートサブネット (10.0.2.0/24)にする
=> インターネットから隔離したサブネット。

#### サブネットのインターネット接続
インターネットに接続するんには「インターネットゲートウェイ (Ingernet Gateway)」を利用する。

```aws cli
# Attach internet-gateway to VPC
aws ec2 attach-internet-gateway --vpc-id "vpc-0addfb681f6a7fe13" --internet-gateway-id "igw-0acdaeee34cb7516a" --region ap-northeast-1
```

#### ネットワークにデータを流すための「ルーティング情報」
AWSでは「ルートテーブル」と呼ばれるもので
「宛先IPアドレス値がいくつのとき、どのネットワークに流すか」という設定を行う。  
=> 疑問：Route53となにが違う  
=> 回答：Route53はDNSサーバーでドメインとインスタンスを紐づける  
=>      ルートテーブルは内部のサブネットとなにかを紐づけるTODO？？

#### 回線引き込み
1. インターネットゲートウェイ (Internet Gateway)の作成
2. Internet GatewayとVPCを紐づける
3. ルートテーブルを作成
4. ルートテーブルとVPCを紐づける
5. ルートテーブルをパブリックサブネットに割り当てる
6. デフォルトゲートウェイをInternet Gatewayに設定する
   - 「0.0.0.0/0」を追加

### 3. サーバーの構築  
#### インスタンスの作成
1. 新しくEC2を作成
2. AMI (Amazon Machine Image)を指定  
   => 「Amazon Linux 2023 AMI」を選択
3. インスタンスタイプを指定  
   =>「t2.micro」を選択
4. キーペアを作成、ダウンロード  
   => インスタンスへのログインに必要、「RSA」、「.pem」を選択
   => my-key.pemをダウンロード
5. VPCとサブネットを指定  
   => VPC: 作成した「VPC領域  
   => サブネット: 分割した「パブリックサブネット」を指定
   => パブリックIPの自動割り当て: 「有効化」を指定  
6. セキュリティグルーブを作成  
   => インバウンドセキュリティグルーブでsshを許可
7. プライベートIPアドレスの手動設定  
   => 「パブリックサブネット」は「10.0.1.0/24」と定義してるので　 
   => 「10.0.1.0 ~ 10.0.1.255」の範囲内で指定可能。  
   =>  今回は「10.0.1.10」を指定
8. ストレージの指定  
   => デフォルト 

#### インスタンスにSSH接続
- IP Address: 35.78.205.180  
=> EC2の「Public IPv4 address
- User: ec2-user  
=> Amazon Linux 2023のデフォルトユーザー
```
ssh -i /Users/shoichisato/.ssh/my-key.pem ec2-user@35.78.205.180
# => 
#    ,     #_
#    ~\_  ####_        Amazon Linux 2023
#   ~~  \_#####\
#   ~~     \###|
#   ~~       \#/ ___   https://aws.amazon.com/linux/amazon-linux-2023
#    ~~       V~' '->
#     ~~~         /
#       ~~._.   _/
#          _/ _/
#        _/m/'
# [ec2-user@ip-10-0-1-10 ~]$
```

#### ルーティングプロトコル
インターネットでは、ルーター同士が通信してルートテーブルの情報をやりとりし、必要に応じで自動的に更新するようにしている。  
=> ルーティングプロトコルと呼ばれる仕組みで実現していて、「EGP」と「IGP」という２つの仕組みで成り立っている。

1. EGP (Exterior Gateway Protocol)  
   ISP (インターネットサービスプロバイダー)やAWSなどの「大きなネットワーク」は、そのネットワークを管理する「AS番号 (Autonomous System)」という番号を持っている。  
   EGPでは、このAS番号をやりとりして、「どのネットワークの先に、どのネットワークが接続されているか」をやりとりしてる。

2. IGP (Interior Gateway Protocol)  
   AS番号が割り当ててあるネットワークの内部ルーター同士で、ルートテーブルのやり取りをすること。  
   プロバイダーやAWSの内部での詳細なやりとりに使われる。

#### TCP/IP通信でのポートの種類
TCP (Transmission Control Protocol)  
=> 相手にデータが届いたことを保証するプロトコル

UDP (User Datagram Protocol) 
=> 相手にデータが届いたことを確認せずに送信する (その代わり高速) 

#### 待ち受けポート番号とプログラムの確認
```
sudo lsof -i -n -P
#=>
# COMMAND    PID            USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
# systemd-n 1957 systemd-network   17u  IPv4  24549      0t0  UDP 10.0.1.10:68
# systemd-n 1957 systemd-network   19u  IPv6  16523      0t0  UDP # [fe80::8b2:24ff:fe66:51d7]:546
# sshd      2154            root    3u  IPv4  17259      0t0  TCP *:22 (LISTEN)
# sshd      2154            root    4u  IPv6  17275      0t0  TCP *:22 (LISTEN)
# chronyd   2183          chrony    5u  IPv4  17435      0t0  UDP 127.0.0.1:323
# chronyd   2183          chrony    6u  IPv6  17436      0t0  UDP [::1]:323
# sshd      3143            root    4u  IPv4  21402      0t0  TCP 10.0.1.10:22->103.# 5.140.134:52305 (ESTABLISHED)
# sshd      3162        ec2-user    4u  IPv4  21402      0t0  TCP 10.0.1.10:22->103.# 5.140.134:52305 (ESTABLISHED)
```
- 「LISTEN」=> 他のコンピュータからの待ち受けをしているポート
- 「ESTABLISHED」 => 相手と現在通信中のポート

#### ファイアウォール
ファイアウォールとは「通してよいデータだけを通して、それ以外を遮断する機能」の総称。

#### パケットフィルタリング
ファイアウォールの機能の１つで、  
流れるパケットを見て、通過の可否を決める仕組み。  
パケットには「IPアドレス」や「ポート番号」も含まれており、  
IPアドレスを判定して、「特定のIPアドレスを送信元とするパケット以外を除外する」
構成にするれば、接続元を制限できる。
=> EC2ではセキュリティグルーブとして設定する


### 4. Webサーバーのインストール
#### IAMユーザーの作成
1. IAMコンソール
2. "Users"をクリック
3. ユーザー作成
4. 作成完了後、access keyの作成
5. 作成したユーザーをGroupsに追加 
   => ここで"Attach policies directly"
   => 今回は"AdministratorAccess"(Provides full access to AWS services and resources.)を追加

#### AWS CLIの設定
```
# サーバーにログイン
$ ssh -i /Users/shoichisato/.ssh/my-key.pem ec2-user@35.78.205.180

# ↓をIAMのコンソールで確認
# - Access Key ID
# - Secret Access Key ID
$ aws configure
# AWS Access Key ID [None]: AKIA2UC3AO7FPC3YUR4H
# AWS Secret Access Key [None]: ***********
# Default region name [None]: ap-northeast-1
# Default output format [None]: json
```

#### Apacheのインストール
```
$ ssh -i /Users/shoichisato/.ssh/my-key.pem ec2-user@35.78.205.180

# -y: Automatically answer yes for all questions.
$ sudo dnf -y install httpd

$ systemctl status httpd.service

# 起動
$ sudo systemctl start httpd.service

# 自動起動するように設定 (サーバーが再起動したときも)
$ sudo systemctl enable httpd.service

# サーバ起動時の自動起動ON OFF確認
$ sudo systemctl list-unit-files -t service | grep httpd
#=> 
# PRESETはデフォルトの値
#
# UNIT FILE                              STATE           PRESET
# httpd.service                          enabled         disabled

# プロセスの確認
$ ps -ax | grep httpd | grep -v grep
# =>
# PID TTY      STAT   TIME COMMAND
# 174197 ?        Ss     0:00 /usr/sbin/httpd -DFOREGROUND
# => 
# STAT: Ssの意味
# - S: 割り込み可能な待ち状態
# - s: セッションリーダー

# ネットワークの待ち受け状態を確認する
$ sudo lsof -i -n -P
# =>
# COMMAND      PID            USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
# httpd     174197            root    4u  IPv6 654417      0t0  TCP *:80 (LISTEN)
# httpd     174225          apache    4u  IPv6 654417      0t0  TCP *:80 (LISTEN)
# httpd     174226          apache    4u  IPv6 654417      0t0  TCP *:80 (LISTEN)
# httpd     174227          apache    4u  IPv6 654417      0t0  TCP *:80 (LISTEN)
# => ポート80番で待ち受けしてる
```
#### ファイアウォールの設定で80番ポート開放
```
$ curl --connect-timeout 5 http://35.78.205.180
# curl: (28) Failed to connect to 35.78.205.180 port 80 after 5002 ms: Timeout was reached
# => 80番ポートが開いてないのでアクセスできない

$ aws ec2 describe-security-groups --group-ids sg-0ec58b0a16405861a
# {
#     "SecurityGroups": [
#         {
#             "Description": "launch-wizard-1 created 2024-04-29T07:12:37.932Z",
#             "GroupName": "WEB-SG",
#             "IpPermissions": [
#                 {
#                     "FromPort": 22,
#                     "IpProtocol": "tcp",
#                     "IpRanges": [
#                         {
#                             "CidrIp": "0.0.0.0/0"
#                         }
#                     ],
#                     "Ipv6Ranges": [],
#                     "PrefixListIds": [],
#                     "ToPort": 22,
#                     "UserIdGroupPairs": []
#                 }
#             ],
#             "OwnerId": "730335311818",
#             "GroupId": "sg-0ec58b0a16405861a",
#             "IpPermissionsEgress": [
#                 {
#                     "IpProtocol": "-1",
#                     "IpRanges": [
#                         {
#                             "CidrIp": "0.0.0.0/0"
#                         }
#                     ],
#                     "Ipv6Ranges": [],
#                     "PrefixListIds": [],
#                     "UserIdGroupPairs": []
#                 }
#             ],
#             "VpcId": "vpc-0addfb681f6a7fe13"
#         }
#     ]
# }

$ aws ec2 authorize-security-group-ingress --group-id sg-0ec58b0a16405861a --ip-permissions "IpProtocol=tcp,FromPort=80,ToPort=80,IpRanges=[{CidrIp=0.0.0.0/0}]"
# {
#     "Return": true,
#     "SecurityGroupRules": [
#         {
#             "SecurityGroupRuleId": "sgr-0239262ec875996f0",
#             "GroupId": "sg-0ec58b0a16405861a",
#             "GroupOwnerId": "730335311818",
#             "IsEgress": false,
#             "IpProtocol": "tcp",
#             "FromPort": 80,
#             "ToPort": 80,
#             "CidrIpv4": "0.0.0.0/0"
#         }
#     ]
# }

$ aws ec2 describe-security-groups --group-ids sg-0ec58b0a16405861a
# ...
#            "IpPermissions": [
#                 {
#                     "FromPort": 80,
#                     "IpProtocol": "tcp",
#                     "IpRanges": [
#                         {
#                             "CidrIp": "0.0.0.0/0"
#                         }
#                     ],
#                     "Ipv6Ranges": [],
#                     "PrefixListIds": [],
#                     "ToPort": 80,
#                     "UserIdGroupPairs": []
#                 },
# ...

$ curl --connect-timeout 5 http://35.78.205.180
# <html><body><h1>It works!</h1></body></html>
#=> アクセスできるようになった
```

#### DNSホスト名の有効化 (Enable DNS hostnames)
```
$ aws ec2 describe-vpc-attribute --vpc-id vpc-0addfb681f6a7fe13 --attribute enableDnsHostnames
# {
#     "VpcId": "vpc-0addfb681f6a7fe13",
#     "EnableDnsHostnames": {
#         "Value": false
#     }
# }

# DNSホスト名を有効化
# - 無効化は"--no-enable-dns-hostnames"
$ aws ec2 modify-vpc-attribute --vpc-id vpc-0addfb681f6a7fe13 --enable-dns-hostnames

# 確認
$ aws ec2 describe-vpc-attribute --vpc-id vpc-0addfb681f6a7fe13 --attribute enableDnsHostnames
# {
#     "VpcId": "vpc-0addfb681f6a7fe13",
#     "EnableDnsHostnames": {
#         "Value": true
#     }
# }
```

#### EC2の「パブリックDNSの確認」
```
# DNSホスト名有効化まえ
$ aws ec2 describe-instances --instance-ids i-06ee364df7beeb004 | grep Public
#                     "PublicDnsName": "",
#                     "PublicIpAddress": "35.78.205.180",
#                                 "PublicDnsName": "",
#                                 "PublicIp": "35.78.205.180"
#                                         "PublicDnsName": "",
#                                         "PublicIp": "35.78.205.180"

# DNSホスト名有効化あと
$ aws ec2 describe-instances --instance-ids i-06ee364df7beeb004 | grep Public
# // "Instances"
#                     "PublicDnsName": "ec2-35-78-205-180.ap-northeast-1.compute.amazonaws.com",
#                     "PublicIpAddress": "35.78.205.180",
# // "NetworkInterfaces"
#                                 "PublicDnsName": "ec2-35-78-205-180.ap-northeast-1.compute.amazonaws.com",
#                                 "PublicIp": "35.78.205.180"
# // "PrivateIpAddresses"
#                                         "PublicDnsName": "ec2-35-78-205-180.ap-northeast-1.compute.amazonaws.com",
#                                         "PublicIp": "35.78.205.180"
# => PublicDnsNameの値が追加
```

#### nslookupでDNSサーバーの動きを確認
(より詳細な確認はdigコマンドで可能)
```
## ローカルから実行
# DNS => IP 検索
$ nslookup ec2-35-78-205-180.ap-northeast-1.compute.amazonaws.com
#=>
# Server:		103.5.140.1 # Wifi接続のため、これはルーターのIPアドレス
# Address:	103.5.140.1#53
# 
# Non-authoritative answer:
# Name:	ec2-35-78-205-180.ap-northeast-1.compute.amazonaws.com
# Address: 35.78.205.180

# IP => DNS 検索
$ nslookup 35.78.205.180
# => 
# Server:		103.5.140.1
# Address:	103.5.140.1#53
# 
# Non-authoritative answer:
# 180.205.78.35.in-addr.arpa	name = ec2-35-78-205-180.ap-northeast-1.compute.amazonaws.com.
# 
# Authoritative answers can be found from:


## AWSから実行
$ nslookup google.com
#=>
# Server:		10.0.0.2 # AWS VPCによって自動提供されたDNSサーバー
# Address:	10.0.0.2#53
# 
# Non-authoritative answer:
# Name:	google.com
# Address: 142.250.207.46
# Name:	google.com
# Address: 2404:6800:4004:821::200e
```



