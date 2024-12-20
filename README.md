# HomeLab-Renew
最新の自宅サーバー事情
* 2024年11月現在の自宅サーバー事情についてです！
![home](./home.drawio.png)

## それぞれの紹介

### 全体
自宅サーバーにはProxmoxハイパーバイザ上で仮想マシンを運用しています。（詳細は後で）
それらのサーバーに外出先からアクセスしたいが、自宅サーバーを直接公開するのは怖いのでさくらのクラウドを経由させます。

また自宅サーバーではカバーできない領域はIaaSやSaaSでカバーしています。
* 独自ドメインの運用
* CDNとWAF
* メールサーバー
* グローバルIPアドレスの使い捨て
* シングルサインオン

自宅サーバーやさくらのクラウドで運用しているものはZabbixサーバーで監視しています。

### 自宅サーバー
#### ルーター YAMAHA RTX1200
![page2](./img/page2-001.png)
ツイッター上で投げ売りされていたのを譲っていただきました。

ルーターとしての機能はもちろん、VLANの払い出しや拠点間VPNに対応しているので重宝しています。
LANポートが全面にあるため、点滅が非常に美しいです。

上流にはOPTAGEによるeo光ネットが繋がっています。

NTTとは異なり電力系ISPなので（動的ですが）IPv4アドレスが割り振られます。シェアードではないので、NATセッション数がRTX1200の上限である20000も有効です。神

設定すればv6アドレスも降ってくるようですが現状はしません。

外部からの通信はさくらのクラウドを経由させます。
HTTPならCloudflare ZeroTrustでトンネリングし、それ以外はIP層でプロキシを挟みます。
外部からのリクエストを直接は受け取りません。
* 原則、外国からの接続は自宅サーバーに流さない
* 攻撃されやすいので、対策が整っているネットワーク事業者様側でなんとかしてもらう
* HTTPに関連した攻撃はCloudflareのWAFがすごい

個人でできるセキュリティには限度があります。攻撃にあった場合のリスクは計り知れないので、お金を払ってでも専門サービスを利用するべきと考えます。<br>
また、<br>自宅サーバーを標的にされた攻撃で家族に迷惑をかけることはあってはなりません。

![page2-002](./img/page2-002.png)
(IPsec VPNはインターネットVPNです)

さくらのクラウドは通信量に対しての課金が発生しません。好き

このように業務用ルーターは可能性が無限大なので非常に便利で勉強への意欲もマシマシになります。

#### Zabbix Server
[Zabbix](https://www.zabbix.com/jp)

サーバーやネットワーク機器を監視するために利用しています。
![sc-img1](./img/sc-image1.png)
![image2](./img/sc-image2.png)
![image3](./img/sc-image3.png)
(ネットワークの監視をしているダッシュボードです。サーバー側は汚くてお見せできません)

過去はGrafanaを使っていたこともありましたが、機能的にはZabbixのほうが使いやすかったのでこっちにしました。

Proxmoxは定期的に再起動やメンテナンスを実施しないといけないため、RaspberryPi3で動いています。

計測している項目
* Zabbix Agent
  * ハイパーバイザ
  * 各仮想マシン
  * Zabbix サーバー本体
* SNMP
  * RTX1200のシステムリソースやトラフィック
    [SNMP MIBリファレンス - YAMAHA](https://www.rtpro.yamaha.co.jp/SW/docs/swx3200/Rev_4_00_13/other/other_snmp_mib.html?rev=4.00.13)
    ```config
    snmp host 192.168.100.2 public
    snmp trap host 192.168.100.2 public
    snmp trap community GW0YahamaRTX1200
    ```
    ```bash
    #LAN1のインバウンドを習得
    $ snmpwalk -v 1 -c public XXXX .1.3.6.1.2.1.2.2.1.10.1
    ```
* シンプルチェック
  * さくらのクラウドVPCルータ
  * インターネット側(1.1.1.1)の計測
    * JPIXのと接続アドレスと迷いました
* TELNETエージェント
  * RTX1200のNATセッション数の推移
  * ログイン周りの仕様により試行錯誤中

ネットワークを監視していると、PeeringDBに記されていた
`非常に大量なインバウンド`の意味が理解できます。<br>
下流のネットワーク機器はブラウジングや動画視聴など、アウトバウンドはHTTPリクエストなどシンプルですが、それに対するレスポンスが大量に降り注いでくる目視することができます。
https://www.peeringdb.com/net/2753

#### Proxmox VE
自宅マシンのほとんどが稼働しているハイパーバイザです。

https://www.proxmox.com/en/

無料な上、非常に多機能です。
使い方によっては分散ストレージやVXLANなどを構築できるようですが、現在勉強中です。

ただのハイパーバイザなので特筆することはありません。
将来的にはクラスタリングで可用性を向上させてみたいですね。

#### NextCloud
NextCloudはオープンソースで利用できるオンラインストレージです。<br>Webブラウザやクライアントソフトウェアも公開されていて、自分用のクラウドストレージを構築できます。
https://nextcloud.com/

Proxmoxの仮想マシンにSSDをアタッチし、Cloudflare ZeroTrust越しに公開しています。
![page3](./img/page3-001.png)

Cloudflare越しにアクセスすると次の恩恵を受けられます
* SSL証明書の自動設定
* プロキシによるグローバルアドレスの隠匿
* IP層での接続制限やWAF

ストレージの接続から公開まで自分で行えるので、インフラの勉強には最適です。

#### DNS兼VPNサーバー
インターネット側から自宅サーバーのリソースにアクセスするためのOpenVPNサーバーと、ローカルDNSサーバーです。<br>諸事情により同じ仮想マシンでホストしています。

#### DNSサーバー
Pi-Hole(https://pi-hole.net/)というオープンソースのDNSサーバーを運用しています。<br>ローカルDNSサーバーでありながら、DNSレベルの広告ブロッカーとして機能します。<br>広告とされるドメインのDNSクエリに対し、`0.0.0.0`を返却することで、Webサイトに埋め込まれた広告が表示できなくなります。<br>

また、任意のドメインをブラックリストに登録することが可能です。自分が踏んでしまった偽警告サイトのドメインを登録しておくことで、家族が引っかかってしまうことを防止することが可能です。

#### VPNサーバー
自宅内のリソースに外部から接続するときに使います。
* 各サーバーへのSSH接続
* ZabbixやProxmoxのWebGUI
* 前述の広告ブロッカー
* その他検閲を気にするとき

##### 工夫１ - HTTPSで待ち受け
OpenVPNはデフォルトでUDPを利用しますが、このシステムはあえてTCP 443(HTTPS)で待ち受けています。<br>
理由は大学などのアウトバウンドが規制されている環境下だとVPNサーバーに繋がらないからです。
![image4](./img/page4-001.png)

TCP通信だとプロトコルの仕様でスループットが低下するなどの懸念事項はありますが、FWを突破するにはHTTPS通信に見せかける必要があります。

後で判明したことですが、大学が提供している学内ネットワーク内だと、ゲームのマルチプレイができないことが判明しました。
WireSharkでパケットをキャプチャしたところ、ゲームサーバーとUDPで通信していたためFWで遮断されたものだと考えます。
