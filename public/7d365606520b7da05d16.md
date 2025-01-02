---
title: NetmikoによるNEC IXルータの操作
tags:
  - Python
  - Network
  - NEC
  - Router
  - netmiko
private: false
updated_at: '2024-05-26T18:37:18+09:00'
id: 7d365606520b7da05d16
organization_url_name: null
slide: false
ignorePublish: false
---
# Netmikoについて

様々なメーカーのネットワーク機器に対応し、sshやtelnetによるCLIベースで自動化を手助けしてくれるPythonライブラリです。

https://github.com/ktbyers/netmiko

筆者は普段からNetmikoを用いたログ取得、設定変更を行っておりますが、自宅にてメインで使用していたNEC UNIVERGE IX2215、IX3110がNetmikoに対応していなかったので、対応させることにしました。

# NEC UNIVERGE IXシリーズについて

NECが販売している小～大規模エンタープライズ向けルーターです。
v6プラスやバーチャルコネクト等のVNEサービスに対応しており、個人宅のインターネットアクセスルーター用途にも最適です。

https://jpn.nec.com/univerge/ix/index.html

# NetmikoのNEC IXシリーズ対応状況
現在、以下のpull requestでNEC IXに対応させたソースコードを提出中です。

https://github.com/ktbyers/netmiko/pull/3408

開発に当たっては以下を参考にさせて頂きました。

[Contributing to Netmiko](https://github.com/ktbyers/netmiko/blob/develop/CONTRIBUTING.md)
[Steps for adding a new vendor](https://github.com/ktbyers/netmiko/blob/develop/VENDOR.md)
[NetmikoでヤマハのNW機器を操作する](https://qiita.com/kitara/items/b2496f74c5a4e0aab174)

また、このpull request内に詳細のテストシナリオを記載していますが、本記事ではサンプルコードによる操作を行います。

# テスト環境
- Ubuntu 24.04
- Python 3.11.9

Netmikoは以下のコマンドでNEC IX用のdevelopブランチをインストールします。
※venv等仮想環境でのインストールをおすすめします
```
pip3 install git+https://github.com/inaba-vdom-0/netmiko.git@develop
```

テストに使用する機器は自宅に余っていたIX2215を使用します。
今回のテストではtelnet、sshによる接続を行うので、ログインユーザー作成、telnetとssh有効、管理IPの設定を行います。

```
username: test-ix
password: testnecix123
management-ip: 192.168.1.30
```

```log:環境設定コマンド
username test-ix password plain testnecix123 administrator
telnet-server ip enable
ssh-server ip enable
interface GigaEthernet0.0
  ip address 192.168.1.30/24
  no shutdown
```

```log:IX2215サンプルコンフィグ
Router(config)# show running-config
! NEC Portable Internetwork Core Operating System Software
! IX Series IX2215 (magellan-sec) Software, Version 10.7.18, RELEASE SOFTWARE
! Compiled Oct 25-Tue-2022 12:37:13 JST #2
! Current time May 26-Sun-2024 16:08:00 JST
!
timezone +09 00
!
logging buffered 8192
logging subsystem tels debug
logging timestamp datetime
!
username test-ix password hash [ --- hashed passoword --- ] administrator
!
!
!
!
!
!
!
!
!
!
!
!
!
telnet-server ip enable
!
ssh-server ip enable
!
!
!
device GigaEthernet0
!
device GigaEthernet1
!
device GigaEthernet2
!
device BRI0
  isdn switch-type hsd128k
!
device USB0
  shutdown
!
interface GigaEthernet0.0
  ip address 192.168.1.30/24
  no shutdown
!
interface GigaEthernet1.0
  no ip address
  shutdown
!
interface GigaEthernet2.0
  no ip address
  shutdown
!
interface BRI0.0
  encapsulation ppp
  no auto-connect
  no ip address
  shutdown
!
interface USB-Serial0.0
  encapsulation ppp
  no auto-connect
  no ip address
  shutdown
!
interface Loopback0.0
  no ip address
!
interface Null0.0
  no ip address
!
```


# サンプルコード

## 1. show versionの取得

```send_command```メソッドを用いてshowコマンドを送信し、結果を取得します。

```python:show_version.py
from netmiko import ConnectHandler

remote_device = {
    'ip': '192.168.1.30',
    'username': 'test-ix',
    'password': 'testnecix123',
    'device_type': 'nec_ix',
    'global_delay_factor': 3,
    'fast_cli' : False
    }

connection = ConnectHandler(**remote_device)
connection.enable()
print(connection.send_command('show version'))

connection.disconnect()
```

```shell:実行結果
NEC Portable Internetwork Core Operating System Software
IX Series IX2215 (magellan-sec) Software, Version 10.7.18, RELEASE SOFTWARE
Compiled Oct 25-Tue-2022 12:37:13 JST #2 by sw-build, coregen-10.7(18)  

ROM: System Bootstrap, Version 22.1
System Diagnostic, Version 22.1
Initialization Program, Version 3.1

System uptime is 6 days 21 hours 45 minutes
System woke up by reload, caused by power-on
System started at May 19-Sun-2024 19:37:55 JST
System image file is "ix2215-ms-10.7.18.ldc"

Processor board ID <0>
IX2215 (P1010E) processor with 262144K bytes of memory.
3 GigaEthernet/IEEE 802.3 interfaces
1 ISDN Basic Rate interface
1 USB interface
1024K bytes of non-volatile configuration memory.
32768K bytes of processor board System flash (Read/Write)

Current configuration is based on "startup-configuration"
```

## 2. InterfaceのIPアドレス設定

設定内容を記述したコンフィグファイルを用意し、```send_config_from_file```メソッドを用いてコンフィグファイルを投入します。

```configfile.txt
interface GigaEthernet2.0
  ip address 10.0.0.1/24
  no shutdown
```

このブランチではtelnetにも対応させているので、今回はtelnetにて接続してみます。
telnetで接続するには、device_typeフィールドの最後に```_telnet```を追加します。

```python:send_config_file_telnet.py
from netmiko import ConnectHandler

remote_device = {
    'ip': '192.168.1.30',
    'username': 'test-ix',
    'password': 'testnecix123',
    'device_type': 'nec_ix_telnet',
    'global_delay_factor': 3,
    'fast_cli' : False
    }

connection = ConnectHandler(**remote_device)
connection.enable()
print(connection.send_command('show ip interface GigaEthernet2.0'))
print(connection.send_config_from_file('configfile.txt'))
print(connection.send_command('show ip interface GigaEthernet2.0'))

connection.disconnect()
```

```log:実行結果
Interface GigaEthernet2.0 is administratively down, line protocol is down
  Internet protocol processing disabled
interface GigaEthernet2.0
Router(config-GigaEthernet2.0)#   ip address 10.0.0.1/24
Router(config-GigaEthernet2.0)#   no shutdown
Router(config-GigaEthernet2.0)#
Interface GigaEthernet2.0 is down, line protocol is down
  Internet address is 10.0.0.1/24
  Broadcast address is 255.255.255.255
  Address determined by config
  MTU is 1500 octets
  Directed broadcast forwarding is disabled
  Proxy ARP is disabled
  Local proxy ARP is disabled
  ICMP redirects are always sent
  IGMP is disabled
  TCP MSS adjustment is disabled
```

## 3. device typeの自動検知

Netmikoでは操作対象のdevice typeを事前に指定して接続を行いますが、autodetect機能を使用すると、自動でdevice typeを判別してくれます。
autodetect機能を使用するには、device_typeフィールドに```'autodetect'```を記述します。
※実行結果内のシリアルナンバーはマスクしています
```python:devicetype_autodetect.py
from netmiko.ssh_autodetect import SSHDetect
from netmiko.ssh_dispatcher import ConnectHandler

remote_device = {
    'ip': '192.168.1.30',
    'username': 'test-ix',
    'password': 'testnecix123',
    'device_type': 'autodetect',
    'global_delay_factor': 3,
    'fast_cli' : False
    }

guesser = SSHDetect(**remote_device)
best_match = guesser.autodetect()
print(best_match)
print(guesser.potential_matches) 

remote_device['device_type'] = best_match
connection = ConnectHandler(**remote_device)

print(connection.send_command('show hardware'))

connection.disconnect()
```

```shell:実行結果
nec_ix
{'nec_ix': 99}
IX Series IX2215 Hardware Platform

S/N: XXXXXXXXXX

ZTP<0>:
  ZTP ID         : XXXXXXXXXXXX#IX#IX2215#XXXXXXXXXX
  Status         : stopped
  Next boot mode : normal mode

Processor board:
  Processor board ID <0>
  CPU/DDR3/CSB/LBUS clock frequencies are 792/792/396/99 MHz
  P1010E processor (revision 0x80f90010)
  262144K bytes of main memory
  1024K bytes of non-volatile configuration memory
  32768K bytes of processor flash memory <0>

IPsec accelerator:
  on board security engine(SEC4.4), revision 0x100

Onboard interface unit GigaEthernet0:
  GigaEthernet Transceiver is VSC8552
Onboard interface unit GigaEthernet1:
  GigaEthernet Transceiver is VSC8552
Onboard interface unit GigaEthernet2:
  GigaEthernet Switch with Transceivers is VSC7424
Onboard interface unit BRI0:
  BRI Transceiver is YTD439

LED information:
  PWR: On     ALM: Off    BSY: Off
  VPN: Off    PPP: Off    BAK: Off
```

# 最後に

本記事ではtelnet接続での操作も実施していますが、現代ではセキュリティ面で問題があるためtelnetでの接続はおすすめしません。
また、NetmikoはPythonの標準ライブラリであるtelnetlibを使用してtelnetを行いますが、Python 3.11からこのライブラリは非推奨となっており、3.13での削除予定が予告されているため、基本的にはsshでの接続をおすすめします。
