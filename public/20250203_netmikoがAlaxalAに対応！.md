---
title: netmikoがAlaxalAに対応！
tags:
  - Python
  - Network
  - alaxala
  - netmiko
private: false
updated_at: ""
id: null
organization_url_name: null
slide: false
ignorePublish: false
---
# はじめに

昨年NEC IXに対応させたpull requestと合わせてAlaxalAスイッチに対応させたpull requestも提出していましたが、先にマージされnetmiko version 4.5.0の一部としてリリースされました。
[Support for AlaxalA AX switch series #3411](https://github.com/ktbyers/netmiko/pull/3411)
[Alaxala 2600s and 3600s driver #3462](https://github.com/ktbyers/netmiko/pull/3462)

本記事では基本的な使い方を紹介します。

# netmiko 4.5.0について

AlaxalA、Cisco APIC等に新規対応したほか、python3.8のサポート終了やバグ修正が含まれます。

https://github.com/ktbyers/netmiko/releases/tag/v4.5.0

# AlaxalAについて

日立製作所とNECの合併により設立された会社で、小規模から大規模まで、広い分野に対応した製品を展開しています。
interop Tokyoでも出展しているのを何度かみたことがあるので、ある程度知名度はあるかと思います。
2021年より、Fortinetと合併事業を開始しているとのこと。

netmiko 4.5.0ではAX3600Sシリーズ、AX2600Sシリーズに対応しています。
※シャーシ型スイッチ(AX4600S、AX8300S、AX8600S)やルーターシリーズには未対応です

https://www.alaxala.com/jp/

https://www.alaxala.com/jp/solutions/#type-business

https://www.alaxala.com/jp/news/press/2021/20211101.html

# テスト環境

- Ubuntu 24.04
- Python 3.11.11
- AX-3640-24TE-A 11.14.S

# 環境準備

## python

```shell
# venv上で作業を行う
python3.11 -m venv .venv
source .venv/bin/activate

# netmikoのインストール
python3.11 -m pip install netmiko

# netmiko 4.5.0がインストールされていることを確認する
python3.11 -m pip list
Package          Version
---------------- -------
...
netmiko          4.5.0
...
```

## AX3640初期設定

ログインユーザー作成とssh有効、管理IPの設定を行います。

```
username: test
password: test
enable password: test
management-ip: 192.168.1.30
```

```shell:環境設定コマンド
adduser test
test
test

password enable-mode
test
test

configure terminal

hostname ax3640-sw

vlan 10

interface vlan 10
  ip address 192.168.1.30 255.255.255.0

interface gigabitethernet 0/1
  media-type rj45
  switchport access vlan 10

username test view-class allcommand

line vty 0 15
  transport input all

ip ssh
ip ssh version 2

```

```log:環境設定
login: operator
Password:
No password is set. Please set password!

Copyright (c) 2005-2020 ALAXALA Networks Corporation. All rights reserved.


> enable
#
# adduser test
User(empty password) add done. Please setting password.

Changing local password for test.
New password:
Please enter a longer password.
New password:
Retype new password:
#
# exit

login: test
Password:

Copyright (c) 2005-2020 ALAXALA Networks Corporation. All rights reserved.


> enable
# password enable-mode
Changing local password for admin.
New password:
Please enter a longer password.
New password:
Retype new password:
#
#
# configure terminal
(config)# hostname ax3640-sw
!(config)#
!ax3640-sw(config)# vlan 10
!ax3640-sw(config-vlan)#
01/20 23:18:51 E3 VLAN 20110002 0700:000000000000 STP (PVST+:VLAN 10) : This bridge becomes the Root Bridge.
!ax3640-sw(config-if)#  ip address 192.168.1.30 255.255.255.0
!ax3640-sw(config-if)#
!ax3640-sw(config-if)#
!ax3640-sw(config-if)# interface gigabitethernet 0/1
!ax3640-sw(config-if)#   media-type rj45
!ax3640-sw(config-if)#   switchport access vlan 10
!ax3640-sw(config-if)#
01/20 23:20:14 E4 VLAN 20110009 0700:1e6380000000 STP (PVST+:VLAN 10) : Port status becomes Blocking on the port(0/1).
!ax3640-sw(config-if)#
!ax3640-sw(config-if)# username test view-class allcommand
!ax3640-sw(config)#
!ax3640-sw(config)# line vty 0 15
!ax3640-sw(config-line)#   transport input all
!ax3640-sw(config-line)# ip ssh
!ax3640-sw(config)# ip ssh version 2
!ax3640-sw(config)#
!ax3640-sw(config)# write # 設定を保存
ax3640-sw(config)#
```

```log:AX3640サンプルコンフィグ
ax3640-sw# show running-config
#Last modified by test at Thu Jan 20 23:22:16 2000 with version 11.14.S
!
hostname "ax3640-sw"
!
vlan 1
  name "VLAN0001"
!
vlan 10
!
spanning-tree mode pvst
!
interface gigabitethernet 0/1
  media-type rj45
  switchport mode access
  switchport access vlan 10
!
interface gigabitethernet 0/2
  switchport mode access
!
interface gigabitethernet 0/3
  switchport mode access
!
interface gigabitethernet 0/4
  switchport mode access
!
interface gigabitethernet 0/5
  switchport mode access
!
interface gigabitethernet 0/6
  switchport mode access
!
interface gigabitethernet 0/7
  switchport mode access
!
interface gigabitethernet 0/8
  switchport mode access
!
interface gigabitethernet 0/9
  switchport mode access
!
interface gigabitethernet 0/10
  switchport mode access
!
interface gigabitethernet 0/11
  switchport mode access
!
interface gigabitethernet 0/12
  switchport mode access
!
interface gigabitethernet 0/13
  switchport mode access
!
interface gigabitethernet 0/14
  switchport mode access
!
interface gigabitethernet 0/15
  switchport mode access
!
interface gigabitethernet 0/16
  switchport mode access
!
interface gigabitethernet 0/17
  switchport mode access
!
interface gigabitethernet 0/18
  switchport mode access
!
interface gigabitethernet 0/19
  switchport mode access
!
interface gigabitethernet 0/20
  switchport mode access
!
interface gigabitethernet 0/21
  switchport mode access
!
interface gigabitethernet 0/22
  switchport mode access
!
interface gigabitethernet 0/23
  switchport mode access
!
interface gigabitethernet 0/24
  switchport mode access
!
interface vlan 1
!
interface vlan 10
  ip address 192.168.1.30 255.255.255.0
!
username test view-class allcommand
!
line vty 0 15
  transport input all
!
ip ssh
ip ssh version 2
!
```

# サンプルコード

## 1. show version、show running-configの取得

`send_command`メソッドを用いてshow versionとshow running-configを取得してみます。

```python:show_commamd.py
from netmiko import ConnectHandler
import logging

logging.basicConfig(filename="netmiko_debug.log", level=logging.DEBUG)
logger = logging.getLogger("netmiko")

remote_device = {
    "ip": "192.168.1.30",
    "username": "test",
    "password": "test",
    "secret": "test",
    "device_type": "alaxala_ax36s",
    "global_delay_factor": 3,
    "session_log": "netmiko.log",
    "fast_cli": False,
}

conn = ConnectHandler(**remote_device) # 接続開始
conn.enable() # enableモードに以降

# showコマンドを実行
print(conn.send_command("show version")) 
print(conn.send_command("show running-config"))

# 切断
conn.disconnect()
```

```log:実行結果
Date 2000/01/21 00:55:35 UTC
Model: AX3640S-24T
S/W: OS-L3A Ver. 11.14.S
H/W: AX-3640-24TE-A [---masked---]
#Last modified by test at Thu Jan 20 23:48:50 2000 with version 11.14.S
!
hostname "ax3640-sw"
!
vlan 1
  name "VLAN0001"
!
vlan 10
!
spanning-tree mode pvst
!
interface gigabitethernet 0/1
  media-type rj45
  switchport mode access
  switchport access vlan 10
!
interface gigabitethernet 0/2
  switchport mode access
!
interface gigabitethernet 0/3
  switchport mode access
!
interface gigabitethernet 0/4
  switchport mode access
!
interface gigabitethernet 0/5
  switchport mode access
!
interface gigabitethernet 0/6
  switchport mode access
!
interface gigabitethernet 0/7
  switchport mode access
!
interface gigabitethernet 0/8
  switchport mode access
!
interface gigabitethernet 0/9
  switchport mode access
!
interface gigabitethernet 0/10
  switchport mode access
!
interface gigabitethernet 0/11
  switchport mode access
!
interface gigabitethernet 0/12
  switchport mode access
!
interface gigabitethernet 0/13
  switchport mode access
!
interface gigabitethernet 0/14
  switchport mode access
!
interface gigabitethernet 0/15
  switchport mode access
!
interface gigabitethernet 0/16
  switchport mode access
!
interface gigabitethernet 0/17
  switchport mode access
!
interface gigabitethernet 0/18
  switchport mode access
!
interface gigabitethernet 0/19
  switchport mode access
!
interface gigabitethernet 0/20
  switchport mode access
!
interface gigabitethernet 0/21
  switchport mode access
!
interface gigabitethernet 0/22
  switchport mode access
!
interface gigabitethernet 0/23
  switchport mode access
!
interface gigabitethernet 0/24
  switchport mode access
!
interface vlan 1
!
interface vlan 10
  ip address 192.168.1.30 255.255.255.0
!
username test view-class allcommand
!
line vty 0 15
  transport input all
!
ip ssh
ip ssh version 2
!
```

## 2. コンフィグファイル投入+設定保存+show startup-config取得

設定内容を記述したコンフィグファイルを用意し、`send_config_from_file`メソッドを用いてコンフィグファイルを投入します。
その後設定を保存してshow startup-configをファイルに保存してみます。

```shell:test.conf
vlan 2

interface vlan 2
  ip address 192.168.2.254 255.255.255.0

interface gigabitethernet 0/2
  no switchport access vlan
  switchport access vlan 2

ip route 0.0.0.0 0.0.0.0 192.168.1.254
```

```python:conf_from_file.py
from netmiko import ConnectHandler
import logging

logging.basicConfig(filename="netmiko_debug.log", level=logging.DEBUG)
logger = logging.getLogger("netmiko")

remote_device = {
    "ip": "192.168.1.30",
    "username": "test",
    "password": "test",
    "secret": "test",
    "device_type": "alaxala_ax36s",
    "global_delay_factor": 3,
    "session_log": "netmiko_.log",
    "fast_cli": False,
}

conf = ('./test.conf') # コンフィグファイル読込

conn = ConnectHandler(**remote_device) # 接続開始
conn.enable() # enableモードに以降

print(conn.send_config_from_file(conf)) # コンフィグファイルを投入
print(conn.save_config()) # 設定を保存

# startup-configをファイルに保存
start_conf = conn.send_command("show startup-config")
print(start_conf)

with open("./startup.conf", "w") as f:
    print(start_conf, file=f)

conn.disconnect()
```

```
configure terminal
ax3640-sw(config)#
ax3640-sw(config)# vlan 2
!ax3640-sw(config-vlan)#
01/21 01:32:35 E3 VLAN 20110002 0700:000000000000 STP (PVST+:VLAN 2) : This bridge becomes the Root Bridge.
!ax3640-sw(config-vlan)#
!ax3640-sw(config-vlan)# interface vlan 2
!ax3640-sw(config-if)#   ip address 192.168.2.254 255.255.255.0       
!ax3640-sw(config-if)#
!ax3640-sw(config-if)# interface gigabitethernet 0/2
!ax3640-sw(config-if)#   no switchport access vlan
!ax3640-sw(config-if)#   switchport access vlan 2
!ax3640-sw(config-if)#
!ax3640-sw(config-if)# ip route 0.0.0.0 0.0.0.0 192.168.1.254
!ax3640-sw(config)# end
Unsaved changes found! Do you exit "configure" without save ? (y/n): y
!ax3640-sw#
write
ax3640-sw(config)#
ax3640-sw(config)#
#Last modified by test at Fri Jan 21 01:32:51 2000 with version 11.14.S
!
hostname "ax3640-sw"
!
vlan 1
...
```

```log:startup.conf
#Last modified by test at Fri Jan 21 01:32:51 2000 with version 11.14.S
!
hostname "ax3640-sw"
!
vlan 1
  name "VLAN0001"
!
vlan 2
!
vlan 10
!
spanning-tree mode pvst
!
interface gigabitethernet 0/1
  media-type rj45
  switchport mode access
  switchport access vlan 10
!
interface gigabitethernet 0/2
  switchport mode access
  switchport access vlan 2
!
interface gigabitethernet 0/3
  switchport mode access
!
interface gigabitethernet 0/4
  switchport mode access
!
interface gigabitethernet 0/5
  switchport mode access
!
interface gigabitethernet 0/6
  switchport mode access
!
interface gigabitethernet 0/7
  switchport mode access
!
interface gigabitethernet 0/8
  switchport mode access
!
interface gigabitethernet 0/9
  switchport mode access
!
interface gigabitethernet 0/10
  switchport mode access
!
interface gigabitethernet 0/11
  switchport mode access
!
interface gigabitethernet 0/12
  switchport mode access
!
interface gigabitethernet 0/13
  switchport mode access
!
interface gigabitethernet 0/14
  switchport mode access
!
interface gigabitethernet 0/15
  switchport mode access
!
interface gigabitethernet 0/16
  switchport mode access
!
interface gigabitethernet 0/17
  switchport mode access
!
interface gigabitethernet 0/18
  switchport mode access
!
interface gigabitethernet 0/19
  switchport mode access
!
interface gigabitethernet 0/20
  switchport mode access
!
interface gigabitethernet 0/21
  switchport mode access
!
interface gigabitethernet 0/22
  switchport mode access
!
interface gigabitethernet 0/23
  switchport mode access
!
interface gigabitethernet 0/24
  switchport mode access
!
interface vlan 1
!
interface vlan 2
  ip address 192.168.2.254 255.255.255.0
!
interface vlan 10
  ip address 192.168.1.30 255.255.255.0
!
ip route 0.0.0.0 0.0.0.0 192.168.1.254
!
username test view-class allcommand
!
line vty 0 15
  transport input all
!
ip ssh
ip ssh version 2
!
```

問題なく設定投入ができました。

# 最後に

過去の自分が(翻訳して)送ったpull requestのメッセージには以下のような記載がありました。

"""
This code is based on the class CiscoBaseConnection because the prompt output from the device resembles Cisco IOS.
However, when modifying the configuration, an "!" is outputted at the beginning of the prompt.
Therefore, the acquisition of base_prompt is adjusted.

example_device_output.log
```
switch(config)# interface gigabitethernet 0/2
switch(config-if)#  shutdown
!switch(config-if)#  no shutdown
!switch(config-if)# 
```
"""

`CiscoBaseConnection`Classをベースに、コンフィグ変更時にホスト名に表示される`!`を取得できるよう調整したとのこと。
私はそこまでPythonの知識があるわけではないので、netmikoの実装を理解するのにかなり時間を要しました。
現在手持ちの機材でしか動作確認ができていないので、AlaxalAスイッチをお持ちの方の動作報告をお待ちしております。