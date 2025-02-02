---
title: Scapy+Tcpreplay+Netmapでトラフィックジェネレーター(改)
tags:
  - Python
  - Network
  - tcpreplay
  - netmap
  - scapy
private: false
updated_at: '2025-02-02T18:48:12+09:00'
id: af67886add149584638b
organization_url_name: null
slide: false
ignorePublish: false
---
# はじめに

[前回の記事](https://qiita.com/Inaba_Hiragi/items/8e38b0119b26b5d78661)を書いている途中で、TcpreplayのコマンドオプションにNetmapがあることに気が付き、更に安定・パフォーマンスが向上したトラフィックジェネレーターにできるのではないかと考えました。(というわけで(改)！！)
本記事では前回作成したScapyによるpcapファイル作成のコードを用いてTcpreplayとNetmapによるスループットパフォーマンステストを行います。

## Scapyについて

pythonで作成されたIPパケット生成ライブラリです。
生成だけでなく、実際にパケットの送信・受信も行うことができます。

https://scapy.net/

https://scapy.readthedocs.io/en/latest/

https://github.com/secdev/scapy

## Tcpreplayについて

tcpdumpやWiresharkによってキャプチャされたネットワークトラフィックを編集および再生するためのツールです。
多数のネットワークベンダーや企業、大学、研究所で使用されているようです。

https://tcpreplay.appneta.com/

https://github.com/appneta/tcpreplay

## Netmapについて

Linux/FreeBSDのカーネルモジュールとして実装されている高速パケットI/Oフレームワークで、Tcpreplayのオプションとして使用することで市販のトラフィックジェネレーターでと同様のフルラインレートを達成できるようになるとのこと。

https://github.com/luigirizzo/netmap

https://tcpreplay.appneta.com/wiki/installation.html

# 検証構成

前回は1Gnicでのテストでしたが、今回は余ってた10Gnicを活用して10G帯域でのテストを実施してみます。
レシーバーvmに10Gnicを割り当ててテストしてみたところkernel dropとinterface dropが多発し計測難易度高なので、今回は対向のルーターへ送信・計測します。

- 送信側
  - Rocky Linux 9
    - 4core 8GB memory
  - KVM host(Proxmox ve)
    - intel X520 10Gb SFP
- 受信側 Fortigate 800C 10G sfp+ port

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2881599/5db13ea5-16a0-595f-6930-75583c311adf.png)

## Guest VMへの物理nicパススルー

前回はホスト側で作成していたネットワークブリッジをvmに割り当てていましたが、Netmapを使用するため物理nicをパススルーします。
Proxmoxでのやり方は以下を参考にしました。(iommuの有効化に苦労した...)
[Proxmoxによる仮想マシンへGPUパススルー(SR-IOV)](https://www.technicalife.net/proxmox-sr-iov-gpu-passthrough/#toc13)

# 環境準備

前回Tcpreplayはepelよりdnfでインストールしていましたが、Netmapを使用する際はソースからビルドをする必要があるようなので、まずはNetmapからインストール、その後Tcpreplayをインストールします。
その他python関連の環境は[前回の記事](https://qiita.com/Inaba_Hiragi/items/8e38b0119b26b5d78661)のものを引き続き使用します。

### Rocky Linux 9の開発ツールインストール

```shell
dnf groupinstall -y base "Development tools"
```

### netmap

```shell
cd /usr/src
git clone https://github.com/luigirizzo/netmap.git

# ./configure 
# 自分の環境ではエラーが出たのでオプション指定
./configure --no-drivers=virtio_net.c --kernel-dir=/usr/src/kernels/5.14.0-503.22.1.el9_5.x86_64
make
make install
```

### Tcpreplay 

最新版の4.5.1だと実行時Netmap間のエラーが出るので4.4.2をダウンロード
```shell
wget https://github.com/appneta/tcpreplay/releases/download/v4.4.2/tcpreplay-4.4.2.tar.gz
tar -zxvf tcpreplay-4.4.2.tar.gz
cd tcpreplay-4.4.2
./configure
make
make install
```

Netmapのインストールが完了していると以下のようにTcpreplay内のLinux/BSD Netmapが有効化され、パスが表示されます
```shell
# Linux/BSD netmap:yesとなりnetmapのパスが記載される
./configure
...
##########################################################################
             TCPREPLAY Suite Configuration Results (4.4.2)
##########################################################################
libpcap:                    /usr (>= 0.9.6)
PF_RING libpcap             no   
libdnet:                    no   
autogen:                     (unknown - man pages will not be built)
Use libopts tearoff:        yes
64bit counter support:      yes
tcpdump binary path:        /sbin/tcpdump
fragroute support:          no
tcpbridge support:          yes
tcpliveplay support:        yes

Supported Packet Injection Methods (*):
Linux TX_RING:              no
Linux PF_PACKET:            yes
BSD BPF:                    no
libdnet:                    no
pcap_inject:                yes
pcap_sendpacket:            yes **
pcap_netmap                 no
Linux/BSD netmap:           yes /usr/src/netmap
Tuntap device support:      yes

* In order of preference; see configure --help to override
** Required for tcpbridge
```

# Tcpreplayの実行

pcapファイルは[前回の記事](https://qiita.com/Inaba_Hiragi/items/8e38b0119b26b5d78661)を参考に64byteと1500byteのデータサイズで作成します。



Tcpreplay実行時のオプションは--mbpsを付けることが推奨されていそうです。

[Q: Why doesn't Tcpreplay send traffic as fast as I told it to?](https://tcpreplay.appneta.com/wiki/faq.html#why-doesnt-tcpreplay-send-traffic-as-fast-as-i-told-it-to)

```shell
# netmapを有効化
sudo su - 
insmod /usr/src/netmap/netmap.ko

# tcpreplay実行こまんど
tcpreplay -i ens17 -K --loop 5000 --netmap --mbps 1000 tcp_port_scan_64.pcap
tcpreplay -i ens17 -K --loop 5000 --netmap --mbps 1000 udp_port_scan_64.pcap
tcpreplay -i ens17 -K --loop 5000 --netmap --mbps 10000 tcp_port_scan_1500.pcap
tcpreplay -i ens17 -K --loop 5000 --netmap --mbps 10000 udp_port_scan_1500.pcap
```

### TCP 1Gbps 64byte

```shell
[root@localhost traffgen]# tcpreplay -i ens17 -K --loop 5000 --netmap --mbps 1000 tcp_port_scan_64.pcap
Switching network driver for ens17 to netmap bypass mode... done!
File Cache is enabled
Actual: 322555000 packets (25159290000 bytes) sent in 201.27 seconds
Rated: 125000198.7 Bps, 1000.00 Mbps, 1602566.65 pps
Flows: 64511 flows, 320.51 fps, 322555000 flow packets, 0 non-flow
Statistics for network device: ens17
        Successful packets:        322555000
        Failed packets:            0
        Truncated packets:         0
        Retried packets (ENOBUFS): 0
        Retried packets (EAGAIN):  357180
```

受信側
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2881599/256f568a-70c1-b2d8-6854-8c972f8072a9.png)

### UDP 1Gbps 64byte

```shell
[root@localhost traffgen]# tcpreplay -i ens17 -K --loop 5000 --netmap --mbps 1000 udp_port_scan_64.pcap
Switching network driver for ens17 to netmap bypass mode... done!
File Cache is enabled
Actual: 322555000 packets (25159290000 bytes) sent in 201.27 seconds
Rated: 125000198.7 Bps, 1000.00 Mbps, 1602566.65 pps
Flows: 64511 flows, 320.51 fps, 322555000 flow packets, 0 non-flow
Statistics for network device: ens17
        Successful packets:        322555000
        Failed packets:            0
        Truncated packets:         0
        Retried packets (ENOBUFS): 0
        Retried packets (EAGAIN):  254035
```

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2881599/f52cf8ca-7cd6-23c7-b22b-f6bc43d37282.png)


1.6Mpps程のレートで1Gbpsの送信ができており、受信側も多少の上下が見られるものの問題なく狙いのスループットが出ていることがわかります。

### TCP 10Gbps 1500byte

※適当に決めたループ値が長すぎるため5分程でinterruptさせています

```shell
[root@localhost traffgen]# tcpreplay -i ens17 -K --loop 5000 --netmap --mbps 10000  tcp_port_scan_1500.pcap
Switching network driver for ens17 to netmap bypass mode... done!
File Cache is enabled
^C User interrupt...
sendpacket_abort
Warning: Unable to send packet: User abort
Actual: 266741810 packets (403847100340 bytes) sent in 332.84 seconds
Rated: 1213326103.7 Bps, 9706.60 Mbps, 801404.29 pps
Flows: 64511 flows, 193.81 fps, 266752985 flow packets, 0 non-flow
Statistics for network device: ens17
        Successful packets:        266741810
        Failed packets:            0
        Truncated packets:         0
        Retried packets (ENOBUFS): 0
        Retried packets (EAGAIN):  33074858
```

受信側
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2881599/420bb149-a0b9-b420-2904-84e6d5adc1e6.png)

### TCP 10Gbps 1500byte

```shell
[root@localhost traffgen]# tcpreplay -i ens17 -K --loop 5000 --netmap --mbps 10000  udp_port_scan_1500.pcap
Switching network driver for ens17 to netmap bypass mode... done!
File Cache is enabled
^C User interrupt...
sendpacket_abort
Warning: Unable to send packet: User abort
Actual: 261650806 packets (396139320284 bytes) sent in 324.19 seconds
Rated: 1221913041.1 Bps, 9775.30 Mbps, 807075.98 pps
Flows: 64511 flows, 198.98 fps, 261656616 flow packets, 0 non-flow
Statistics for network device: ens17
        Successful packets:        261650806
        Failed packets:            0
        Truncated packets:         0
        Retried packets (ENOBUFS): 0
        Retried packets (EAGAIN):  44570026
```

受信側
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2881599/9c127ef6-3d45-da5f-1830-03be01098ae8.png)

TCPとUDPどちらも10Gbpsに近いスループットが出ています。

# Netmapのパフォーマンスについて

次は`--netmap`オプションを使用せずにトラフィックを送信してみます。

### TCP 10Gbps 1500byte

```shell
[root@localhost traffgen]# tcpreplay -i ens17 -K --loop 5000 --mbps 10000  tcp_port_scan_1500.pcap
File Cache is enabled
^C User interrupt...
sendpacket_abort
Actual: 139794081 packets (211648238634 bytes) sent in 301.04 seconds
Rated: 703056864.9 Bps, 5624.45 Mbps, 464370.45 pps
Flows: 64511 flows, 214.29 fps, 139795337 flow packets, 0 non-flow
Statistics for network device: ens17
        Successful packets:        139794080
        Failed packets:            0
        Truncated packets:         0
        Retried packets (ENOBUFS): 0
        Retried packets (EAGAIN):  0
```
受信側
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2881599/ea47aec5-9fb3-070c-efc8-80ca0320b90d.png)

### UDP 10Gbps 1500byte

```shell
[root@localhost traffgen]# tcpreplay -i ens17 -K --loop 5000 --mbps 10000  udp_p
ort_scan_1500.pcap
File Cache is enabled
^C User interrupt...
sendpacket_abort
Actual: 159914469 packets (242110506066 bytes) sent in 308.45 seconds
Rated: 784916084.9 Bps, 6279.32 Mbps, 518438.62 pps
Flows: 64511 flows, 209.14 fps, 159922769 flow packets, 0 non-flow
Statistics for network device: ens17
        Successful packets:        159914468
        Failed packets:            0
        Truncated packets:         0
        Retried packets (ENOBUFS): 0
        Retried packets (EAGAIN):  0
```

受信側
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2881599/dc6800f3-4ae8-c543-d7fb-cba942fda1b5.png)

オプションを有効にしたときの結果と比べ、20-30%程スループットが低下し、波形も大幅な上下が見られます。
Netmapがスループット向上に大きく貢献していることがわかりました。

## その他

- Tcpreplayはマルチスレッド処理に対応しておらず、実行時はTcpreplayのプロセスを1coreのみで処理している
- hostサーバーからnicをパススルーしてguestにてNetmapを使用しているが、ptnetmapという方法があり本来はこちらの手法が正しいのかもしれない
  - [Netmap passthrough howto](https://github.com/luigirizzo/netmap/blob/master/README.ptnetmap.md)
- スループットテスト用のpcapファイルを作成した方が
- ベアメタル環境では更なる安定性向上が期待される
- Netmapはmodprobeでの有効化ができなかったためinsmodでの有効化を行っています
  - ただし起動する都度実行する必要があるため、改善する必要あり

## 最後に

前回の記事を書いた勢いで環境構築と検証を行いましたが、想定した通りNetmapによるTcpreplayの性能向上を確認できました。
今後もソフトウェアトラフィックジェネレーターの可能性について模索していきたいと思います。
また、今回久々に10G環境を触れてみて思ったのが、自宅ラボの10G機器が少なく(本検証で出てきた装置が全て)、満足に検証ができなかったので近いうちに10G sfp+搭載の小型PCやスイッチを仕入れて検証を行う予定です。
