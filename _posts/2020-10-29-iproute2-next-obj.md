---
layout    : post
title     : "iproute2のnexthop objectでIPv4経路のネクストホップをIPv6アドレスに設定する"
date      : 2020-10-29
lastupdate: 2020-10-29
categories: linux
---

**注意: これは [RFC5549](https://tools.ietf.org/html/rfc5549) ではありません。全く異なるソフトウェアスタックの実装です。**

--------

* TOC
{:toc}

--------

Linuxのネットワークスタックの設定や確認で頻繁に利用するツールに `iproute2` があります。最近iproute2に `nexthop object` というものが実装され、経路の設定とネクストホップの設定を分離できるようになりました。
これによりネクストホップの設定の柔軟性が増したので、簡単に使い方をまとめておきます。

この機能の非常に便利なところは、IPv4経路にIPv6ネクストホップを設定できることです(その逆も可能)。
これにより、BGPやOSPFv3などのデュアルスタックなAddress Familyを持つルーティングプロトコルを、シングルスタックのトランスポートネットワークに実装するのが容易になります。

iproute2内部の実装の詳細などは、cumulusのエンジニアが解説している[資料](https://linuxplumbersconf.org/event/4/contributions/434/attachments/251/436/nexthop-objects-talk.pdf)があるので、そちらを参考にしてください。

コマンドの詳しい使い方は [man](https://man7.org/linux/man-pages/man8/ip-nexthop.8.html) を読んでください。

### 環境

```shell
ubuntu% uname -r; ip -V
5.4.0-48-generic
ip utility, iproute2-v5.7.0-77-gb687d1067169
```

### 構成

<p align="center"><img src="/assets/img/2020-10-29-iproute2-next-obj/topo.png" ></p>

### 手順

設定手順は以下のようになります。

1. 2つのnetnsを作成
1. 作成したnetnsをvethで接続
1. netnsの内部にdummy deviceを作成してIPv4アドレスを設定
1. 対向のdummy device宛のルーティングとネクストホップを設定する
1. 疎通確認

```shell
# 手順1~2
ubuntu% sudo ip netns add ns1
ubuntu% sudo ip netns add ns2
ubuntu% sudo ip link add veth1 netns ns1 type veth peer name veth2 netns ns2

# 手順3
ubuntu% sudo ip netns exec ns1 ip link add dummy1 type dummy
ubuntu% sudo ip netns exec ns2 ip link add dummy2 type dummy
ubuntu% sudo ip netns exec ns1 ip addr add 1.1.1.1/32 dev dummy1
ubuntu% sudo ip netns exec ns2 ip addr add 2.2.2.2/32 dev dummy2
ubuntu% sudo ip netns exec ns1 ip link set dummy1 up
ubuntu% sudo ip netns exec ns2 ip link set dummy2 up
```

deviceと作成しupさせた時点で、自動的に `fe80::/64`のIPv6 LLA(Link Local Address)がdeviceに付与されます。vethの場合、ホスト部はランダムなIPが生成されるようです。
明示的にわかりやすいアドレスを付与できますが、今回はこのまま使います。

```shell
# ns1
ubuntu% sudo ip netns exec ns1 ip a s
(snip)
3: veth1@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether f2:c8:c6:66:5e:ed brd ff:ff:ff:ff:ff:ff link-netns ns2
    inet6 fe80::f0c8:c6ff:fe66:5eed/64 scope link
       valid_lft forever preferred_lft forever
4: dummy1: <BROADCAST,NOARP,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN group default qlen 1000
    link/ether f2:1a:0a:e4:f5:20 brd ff:ff:ff:ff:ff:ff
    inet 1.1.1.1/32 scope global dummy1
       valid_lft forever preferred_lft forever
    inet6 fe80::f01a:aff:fee4:f520/64 scope link
       valid_lft forever preferred_lft forever

# ns2
ubuntu% sudo ip netns exec ns2 ip a s
(snip)
3: veth2@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether ce:e8:2b:42:57:1f brd ff:ff:ff:ff:ff:ff link-netns ns1
    inet6 fe80::cce8:2bff:fe42:571f/64 scope link
       valid_lft forever preferred_lft forever
4: dummy2: <BROADCAST,NOARP,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN group default qlen 1000
    link/ether ea:b7:99:54:ad:d0 brd ff:ff:ff:ff:ff:ff
    inet 2.2.2.2/32 scope global dummy2
       valid_lft forever preferred_lft forever
    inet6 fe80::e8b7:99ff:fe54:add0/64 scope link
       valid_lft forever preferred_lft forever
```

以下のIPv6 LLAにIPv4のネクストホップを向けることになります。

* ns1-veth1 : `fe80::f0c8:c6ff:fe66:5eed`
* ns2-veth2 : `fe80::cce8:2bff:fe42:571f`

`ip nexthop ` に続くコマンドでネクストホップオブジェクトを作成し、ルーティングと紐付けます。

以下の設定では、 `id 46` のネクストホップオブジェクトを作成し、IPv4のデフォルトルートがこのオブジェクトを参照するルーティングを設定しています。
ネクストホップには対向のvethのIPv6 LLAを指定します。

```shell
# 手順4 ns1
ubuntu% sudo ip netns exec ns1 ip nexthop add id 46 via fe80::cce8:2bff:fe42:571f dev veth1
ubuntu% sudo ip netns exec ns1 ip route add 0/0 nhid 46

# 手順4 ns2
ubuntu% sudo ip netns exec ns2 ip nexthop add id 46 via fe80::f0c8:c6ff:fe66:5eed dev veth2
ubuntu% sudo ip netns exec ns2 ip route add 0/0 nhid 46
```

正しく設定されると、以下のようになります。
IPv4経路のルーティングネクストホップがIPv6になっていることが確認できます。

```shell
# ns1
ubuntu% sudo ip netns exec ns1 ip nexthop show
id 46 via fe80::cce8:2bff:fe42:571f dev veth1 scope link

ubuntu% sudo ip netns exec ns1 ip route show
default nhid 46 via inet6 fe80::cce8:2bff:fe42:571f dev veth1

ubuntu% sudo ip netns exec ns1 ip route get 2.2.2.2
2.2.2.2 via inet6 fe80::cce8:2bff:fe42:571f dev veth1 src 1.1.1.1 uid 0
    cache

# ns2
ubuntu% sudo ip netns exec ns2 ip nexthop show
id 46 via fe80::f0c8:c6ff:fe66:5eed dev veth2 scope link

ubuntu% sudo ip netns exec ns2 ip route show
default nhid 46 via inet6 fe80::f0c8:c6ff:fe66:5eed dev veth2

ubuntu% sudo ip netns exec ns2 ip route get 1.1.1.1
1.1.1.1 via inet6 fe80::f0c8:c6ff:fe66:5eed dev veth2 src 2.2.2.2 uid 0
    cache
```

### 疎通確認

簡単に疎通確認します。

```shell
ubuntu% sudo ip netns exec ns1 ping 2.2.2.2 -c 4
PING 2.2.2.2 (2.2.2.2) 56(84) バイトのデータ
64 バイト応答 送信元 2.2.2.2: icmp_seq=1 ttl=64 時間=0.043ミリ秒
64 バイト応答 送信元 2.2.2.2: icmp_seq=2 ttl=64 時間=0.114ミリ秒
64 バイト応答 送信元 2.2.2.2: icmp_seq=3 ttl=64 時間=0.108ミリ秒
64 バイト応答 送信元 2.2.2.2: icmp_seq=4 ttl=64 時間=0.122ミリ秒

--- 2.2.2.2 ping 統計 ---
送信パケット数 4, 受信パケット数 4, パケット損失 0%, 時間 3077ミリ秒
rtt 最小/平均/最大/mdev = 0.043/0.096/0.122/0.031ミリ秒
```

そのままnetns内でtermsharkに流し込んで確認してみます。

```shell
sudo ip netns exec ns2 termshark -i veth2
```

<p align="center"><img src="/assets/img/2020-10-29-iproute2-next-obj/termshark1.png" ></p>

あくまでネクストホップがIPv6になるだけなので、パケットのカプセル化などは当然行われていないことが確認できます。

### まとめ

簡単に設定できました。
その他にも `Blackhole nexthop` や、ネクストホップをグループ化して `Multipath nexthop` などもできるようです(未検証)

```shell
# Blackhole nexthop
ip nexthop add id 3 blackhole

# Multipath nexthop
ip nexthop add id 1 via 172.16.1.1 dev eth1
ip nexthop add id 2 via 172.16.2.1 dev eth2
ip nexthop add id 101 group 1/2
```

おわり。
