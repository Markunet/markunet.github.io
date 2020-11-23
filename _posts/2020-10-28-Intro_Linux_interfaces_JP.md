---
layout    : post
title     : "[翻訳] 仮想ネットワークのための Linux network interface まとめ"
date      : 2018-01-28
lastupdate: 2020-10-28
categories: linux
---

### 注意

**自分用の勉強メモです。**

この記事は、以下の記事から翻訳されています。

[Introduction to Linux interfaces for virtual networking](https://developers.redhat.com/blog/2018/10/22/introduction-to-linux-interfaces-for-virtual-networking/)

[Macvlan vs Ipvlan](http://hicu.be/macvlan-vs-ipvlan)

[Bridge vs Macvlan](http://hicu.be/bridge-vs-macvlan)

**必要な部分しか翻訳していないため、不明な点がある場合は、元のテキストを参照してください。**

--------

* TOC
{:toc}

以下は翻訳です。

--------

Linuxには、VMとコンテナ、およびクラウド環境をホストするための基盤として使用される豊富な仮想ネットワーク機能があります。この投稿では、一般的に使用されるすべての仮想ネットワークインターフェースタイプについて簡単に紹介します。コードの解説ではなく、Linuxでのインターフェースとその使用法の簡単な紹介だけです。ネットワークのバックグラウンドを持っている人なら誰でもこのブログ投稿に興味があるかもしれません。インターフェースのリストは、`ip link help` コマンドを使用して取得できます 。

この投稿では、次の頻繁に使用されるインターフェースと、互いに簡単に混同される可能性のあるいくつかのインターフェースについて説明します。

この記事を読むと、これらのインターフェースとは何か、それらの違いは何か、いつ使用するか、どのように作成するかがわかります。
トンネルなどの他のインターフェースについては、[An introduction to Linux virtual interfaces: Tunnels](https://developers.redhat.com/blog/2019/05/17/an-introduction-to-linux-virtual-interfaces-tunnels/)を参照してください。

## Bridge

Linuxブリッジは、L2スイッチのように動作します。接続されているインターフェース間でパケットを転送します。これは通常、VMとホスト上のネットワーク名前空間の間でパケットを転送する時などに使用されます。また、STP、VLANフィルタ、およびマルチキャストスヌーピングもサポートしています。VM、コンテナ、およびホスト間の通信チャネルを確立する場合は、ブリッジを使用します。

<p align="center"><img src="/assets/img/2020-10-28-linux-interfaces/bridge.png" ></p>

ブリッジを作成する方法は次のとおりです。

```shell
ip link add br0 type bridge
ip link set eth0 master br0
ip link set tap1 master br0
ip link set tap2 master br0
ip link set veth1 master br0
```

これにより、`br0` という名前のブリッジデバイスが作成され、上の図に示すように、2つのTAPデバイス（tap1、tap2）、VETHデバイス（veth1）、および物理デバイス（eth0）がブリッジデバイスのスレーブとして設定されます。

## Bonded interface

Linux Bonding Driver は、複数のネットワークインターフェースを単一の論理的な「結合(bond)された」インターフェースに集約する方法を提供します。Bonded interface の動作はモードによって異なります。一般的に、モードはホットスタンバイまたは負荷分散サービスのいずれかを提供します。
リンクスピードを上げたり、サーバーでフェイルオーバーを実行したりする場合は、 Bonded interface を使用します。 Bonded interface を作成する方法は次のとおりです。

<p align="center"><img src="/assets/img/2020-10-28-linux-interfaces/bond.png" ></p>

```shell
ip link add bond1 type bond miimon 100 mode active-backup
ip link set eth0 master bond1
ip link set eth1 master bond1
```

これにより、`mode active-backup` の `bond1` が作成されます。その他のモードについては、カーネルのドキュメントを参照してください 。

## Team device

Bonded interface と同様に、Team device の目的は、複数のNIC（ポート）をL2で論理的に1つの（teamdev）にグループ化するメカニズムを提供することです。

<p align="center"><img src="/assets/img/2020-10-28-linux-interfaces/team.png" ></p>

ただし、 Bonded interface と Team の間には機能上の違いもいくつかあります。たとえば、チームは LACP 負荷分散、NS/NA（IPV6）リンク監視、D-Busインターフェースなどをサポートしますが、これらはBonding にはありません。 Bonding では提供されない機能を使用する場合は、チームを使用してください。
Bonding と Team の違いの[詳細](https://github.com/jpirko/libteam/wiki/Bonding-vs.-Team-features)は以下です。

| Feature | Bonding | Team |
| ---- | ---- | ---- |
| broadcast TX policy | Yes | Yes |
| round-robin TX policy	| Yes | Yes |
| active-backup TX policy | Yes | Yes |
| LACP (802.3ad) support | Yes | Yes |
| Hash-based TX policy	| Yes | Yes |
| Highly customizable hash function setup | No | Yes |
| TX load-balancing support (TLB) | Yes | Yes |
| RX load-balancing support (ALB) | Yes | Planned |
| RX load-balancing support (ALB) in bridge or openvswitch	| No | Planned |
| LACP hash port select | Yes | Yes |
| load-balancing for LACP support | No | Yes |
| Ethtool link monitoring | Yes | Yes |
| ARP link monitoring | Yes | Yes |
| NS/NA (IPV6) link monitoring | No | Yes |
| ports up/down delays | Yes | Yes |
| port priorities and stickiness ("primary" option enhancement)	 | No | Yes |
| separate per-port link monitoring setup | No | Yes |
| multiple link monitoring setup | Limited | Yes |
| lockless TX/RX path | No(rwlock) | Yes (RCU) |
| VLAN support | Yes | Yes |
| user-space runtime control | Limited | Full |
| Logic in user-space | No | Yes |
| Extensibility	| Hard | Easy |
| Modular design | No | Yes |
| Performance overhead | Low | Very Low |
| D-Bus interface | No | Yes |
| ØMQ interface | No | Yes |
| multiple device stacking | Yes | Yes |
| zero config using LLDP | No | Planned |

チームを作成する方法は次のとおりです。

```shell
teamd -o -n -U -d -t team0 -c '{"runner"：{"name"： "activebackup"}、 "link_watch"：{"name"： "ethtool"}}'
ip link set eth0 down
ip link set eth1 down
teamdctl team0 port add eth0
teamdctl team0 port add eth1
```

これにより、`team0`という名前の Team Interface が`active-backup`モードで作成され、`eth0` と`eth1`が`team0`のサブインタフェースとして追加されます。

## VLAN (Virtual LAN)

VM、名前空間、またはホストでサブネットを分離する場合は、VLANを使用します。
VLANを作成する方法は次のとおりです。

```shell
ip link add link eth0 name eth0.2 type vlan id 2
ip link add link eth0 name eth0.3 type vlan id 3
```
これはVLAN 2を追加`eth0.2`、VLAN 3を`eth0.3`で追加します。トポロジは次のようになります。

<p align="center"><img src="/assets/img/2020-10-28-linux-interfaces/vlan.png" ></p>

## VXLAN (Virtual eXtensible Local Area Network)

<p align="center"><img src="/assets/img/2020-10-28-linux-interfaces/vxlan.png" ></p>

VXLANの使用方法は次のとおりです。 `dstport 4789` は [IETF RFC7348](https://tools.ietf.org/html/rfc7348)で標準化されたVXLAN UDP portです。

```shell
ip link add name ${interface name} type vxlan \
   id <0-16777215> \
   dev ${source interface}\
   remote ${remote endpoint address} \
   local ${local endpoint address} \
   dstport ${VXLAN destination port}
```

1:1でstaticにトンネルを設定する場合は以下のようにします。

```shell
ip link add name vx0 type vxlan id 100 local 1.1.1.1 remote 2.2.2.2 dev eth0 dstport 4789
```

マルチキャストグループを設定することで1:Nのトンネルを設定できます。

```shell
ip link add name vx0 type vxlan id 100 dev eth0 group 239.0.0.1 dstport 4789
```

情報の表示、削除、転送テーブルの追加は以下です。

```shell
# Delete vxlan device
ip link delete vxlan0

# Show vxlan info
ip -d link show vxlan0

# Create forwarding table entry
bridge fdb add to 00:17:42:8a:b4:05 dst 192.19.0.2 dev vxlan0

# Delete forwarding table entry
bridge fdb delete 00:17:42:8a:b4:05 dev vxlan0

# Show forwarding table
bridge fdb show dev vxlan0
```

## MACVLAN

MACVLANを使用すると、単一の物理インターフェース上に、異なるMACアドレスとIPアドレスを持つ複数のサブインターフェースを作成できます。
これは、VLANを使用して物理インターフェース上にサブインターフェースを作成することとは異なります。vlanサブインターフェースでは、各サブインターフェースはvlanを使用する異なるL2ドメインに属し、すべてのサブインターフェースは同じMACアドレスを持っています。macvlanを使用すると、各サブインターフェースは一意のMACアドレスとIPアドレスを取得し、アンダーレイネットワークに直接公開されます。

<p align="center"><img src="/assets/img/2020-10-28-linux-interfaces/linux-macvlan-1.png" ></p>

MACVLANの前は、以下に示すように、VMまたは名前空間から物理ネットワークに接続する場合は、TAP / VETHデバイスを作成し、片側をブリッジに接続し、同時に物理インターフェースをホスト上のブリッジに接続する必要がありました。

<p align="center"><img src="/assets/img/2020-10-28-linux-interfaces/br_ns.png" ></p>

現在、MACVLANを使用すると、MACVLANに関連付けられている物理インターフェースを、ブリッジを必要とせずに名前空間に直接バインドできます。

MACVLANには5つのタイプがあります。

1. Private:
同じ親インターフェース上のサブインターフェースは、相互に通信できません。サブインターフェースからのすべてのフレームは、親インターフェースを介して転送されます。外部の物理スイッチがヘアピンモードをサポートしている場合でも、同じ物理インターフェース上のMACVLANインスタンス間の通信を許可しません。

<p align="center"><img src="/assets/img/2020-10-28-linux-interfaces/linux-macvlan-private-mode.png" ></p>

2. VEPA(Virtual Ethernet Port Aggregator):
サブインターフェースからのすべてのフレームは、親インターフェースを介して転送されます。VEPAモードには、`IEEE 802.1Qbg`で定義され、VEPAとなる物理スイッチが必要です。VEPA対応スイッチは、送信元と宛先の両方がmacvlanインターフェースに対してローカルであるすべてのフレームを返します。その結果、同じ親インターフェース上のmacvlanサブインターフェースは、物理スイッチを介して相互に通信できます。親インターフェースを介して着信するブロードキャストフレームは、VEPAモードのすべてのmacvlanインターフェースにフラッディングされます。VEPAモードは、物理スイッチにポリシーを適用していて、すべてのVM間トラフィックが物理スイッチを通過するようにする場合に役立ちます。

<p align="center"><img src="/assets/img/2020-10-28-linux-interfaces/linux-macvlan-802.1qbg-vepa-mode.png" ></p>

3. Bridge:
親インターフェース上のすべてのサブインターフェースを単純なブリッジで接続します。あるインターフェースから別のインターフェースへのフレームは直接配信され、外部に送信されません。ブロードキャストフレームは、他のすべてのブリッジポートと外部インターフェースにフラッディングされますが、VEPスイッチから戻ってくると破棄されます。すべてのmacvlanサブインターフェースのMACアドレスがわかっているため、macvlanブリッジモードはMAC学習を必要とせず、STPも必要としません。ブリッジモードはVM間の通信を最速で提供しますが、親インターフェースの性能が低下すると、すべてのmacvlanサブインターフェースの性能も低下します。物理インターフェースが切断されると、VMは相互に通信できなくなります。

<p align="center"><img src="/assets/img/2020-10-28-linux-interfaces/linux-macvlan-bridge-mode.png" ></p>

MACVLANをブリッジモード設定する方法は次のとおりです。

```shell
ip link add macvlan1 link eth0 type macvlan mode bridge
ip link add macvlan2 link eth0 type macvlan mode bridge
ip netns add net1
ip netns add net2
ip link set macvlan1 netns net1
ip link set macvlan2 netns net2
```

これにより、ブリッジモードで2つの新しいMACVLANデバイスが作成され、これら2つのデバイスが2つの異なる名前空間に割り当てられます。

4. Passthru
単一のVMを物理インターフェースに直接接続できます。このモードの利点は、VMがMACアドレスやその他のインターフェースパラメータを変更できることです。

<p align="center"><img src="/assets/img/2020-10-28-linux-interfaces/linux-macvlan-passthru-mode.png" ></p>

5. Source:
送信元モードは、許可された送信元MACアドレスのリストに基づいてトラフィックをフィルタリングし、MACベースのVLANアソシエーションを作成するために使用されます。

```shell
以下の例では、
00:11:11:11:11:11, 00:22:22:22:22:22 のクライアントは macvlan0 のinterfaceでのみ許可されている。
00:44:44:44:44:44 のクライアントは macvlan1 のinterfaceでのみ許可されている。
00:33:33:33:33:33 のクライアントは 両方 のinterfaceで許可されている。

 ip link add link eth0 name macvlan0 type macvlan mode source
 ip link add link eth0 name macvlan1 type macvlan mode source
 ip link set link dev macvlan0 type macvlan macaddr add 00:11:11:11:11:11
 ip link set link dev macvlan0 type macvlan macaddr add 00:22:22:22:22:22
 ip link set link dev macvlan0 type macvlan macaddr add 00:33:33:33:33:33
 ip link set link dev macvlan1 type macvlan macaddr add 00:33:33:33:33:33
 ip link set link dev macvlan1 type macvlan macaddr add 00:44:44:44:44:44
```
タイプはさまざまなニーズに応じて選択されます。ブリッジモードが最も一般的に使用されます。
コンテナから物理ネットワークに直接接続する場合は、MACVLANを使用します。

## IPVLAN

IPVLANはすべてのサブインターフェースは親のインターフェースのMACアドレスを共有しますが、個別のIPアドレスを使用します。

<p align="center"><img src="/assets/img/2020-10-28-linux-interfaces/linux-ipvlan.png" ></p>

単一の親インターフェース上のすべてのVMまたはコンテナが同じMACアドレスを使用するため、ipvlanにはいくつかの欠点もあります。

* 共有MACアドレスはDHCP操作に影響を与える可能性があります。VMまたはコンテナがDHCPを使用してネットワーク設定を取得する場合は、DHCP要求で一意のClientIDを使用し、DHCPサーバーがクライアントのMACアドレスではなくClientIDに基づいてIPアドレスを割り当てるようにします。
* 自動構成されたEUI-64 IPv6アドレスは、MACアドレスに基づいています。同じ親インターフェースを共有するすべてのVMまたはコンテナーは、同じIPv6アドレスを自動生成します。VMまたはコンテナが静的IPv6アドレスまたはIPv6プライバシーアドレスを使用していることを確認し、SLAACを無効にします。

### IPVLAN mode

ipvlanには2つの動作モードがあります。単一の親インターフェースで選択できるのは、2つのモードのうちの1つだけです。すべてのサブインターフェースは、選択したモードで動作します。

#### ipvlan L2 mode
L2モードは、macvlanブリッジモードに類似しています。
親インターフェースは、サブインターフェースと親インターフェースの間のスイッチとして機能します。同じ親Ipvlanインターフェースに接続され、同じサブネット内にあるすべてのVMまたはコンテナーは、親インターフェースを介して相互に直接通信できます。他のサブネット宛てのトラフィックは、親インターフェースを介してデフォルトゲートウェイ（物理ルーター）に送信されます。L2モードのIpvlanは、ブロードキャスト/マルチキャストをすべてのサブインターフェースに配信します。

<p align="center"><img src="/assets/img/2020-10-28-linux-interfaces/linux-ipvlan-l2-mode.png" ></p>

#### ipvlan L3 mode

L3モードは、サブインターフェースと親インターフェースの間のL3デバイス（ルータ）として機能します。
各サブインターフェースは異なるサブネットで構成する必要があります。つまり、両方のインターフェースで10.10.40.0/24を構成することはできません。

<p align="center"><img src="/assets/img/2020-10-28-linux-interfaces/linux-ipvlan-l3-mode-1.png" ></p>

ブロードキャストはL2ドメインに制限されているため、あるサブインターフェースから別のサブインターフェースにブロードキャストすることはできません。L3モードはマルチキャストをサポートしていません。
L3モードはルーティングプロトコルをサポートしていないため、接続先のサブネットを物理ネットワークルーターに通知することはできません。サブインターフェース上のすべてのサブネットに対して、ホストの物理インターフェースを指す物理ルーターに静的ルートを構成する必要があります。

>L3モードはルーターのように動作しますが、通過するパケットのTTL値は減少しません。したがって、tracerouteを実行すると、パスに `ipvlan l3 mode` の hop は表示されません。

#### Macvlan vs Ipvlan
`macvlan` と `ipvlan` は、同じ親インターフェースで同時に使用することはできません。

設定は以下のようになります。l3s mode は `conn-track` を利用した対称ルーティングを行う。FLAGはmacvlanと同様の動作。

```shell
ip link add link <master> name <slave> type ipvlan [ mode MODE ] [ FLAGS ]
   where
     MODE: l3 (default) | l3s | l2
     FLAGS: bridge (default) | private | vepa

e.g.
(a) Following will create IPvlan link with eth0 as master in
    L3 bridge mode
      bash# ip link add link eth0 name ipvl0 type ipvlan
(b) This command will create IPvlan link in L2 bridge mode.
      bash# ip link add link eth0 name ipvl0 type ipvlan mode l2 bridge
(c) This command will create an IPvlan device in L2 private mode.
      bash# ip link add link eth0 name ipvlan type ipvlan mode l2 private
(d) This command will create an IPvlan device in L2 vepa mode.
      bash# ip link add link eth0 name ipvlan type ipvlan mode l2 vepa

# L2 mode & bridge flag on netns
ip netns add ns0
ip link add name ipvl0 link eth0 type ipvlan mode l2
ip link set dev ipvl0 netns ns0
```

次の場合にIpvlanを使用します。

* ハードウェアのMACアドレス数の上限を超え、親インターフェースのパフォーマンス低下が懸念される場合
* 接続された物理スイッチのポートで許可されるMACアドレスの数が制限されている（ポートセキュリティ）
* 同じVMまたはコンテナで実行されているBGPデーモンを使用して、VMまたはコンテナで実行しているサービスをアドバタイズするなどの高度なネットワークシナリオ(CNI pluginを使う場合など)

## MACVTAP/IPVTAP

MACVTAP / IPVTAPは、仮想化されたブリッジネットワークを簡素化することを目的とした新しいデバイスドライバです。MACVTAP / IPVTAPインスタンスが物理インターフェース上に作成されると、カーネルは、KVM / QEMUで直接使用できるTUN / TAPデバイスと同じように使用される文字 device / dev / tapX も作成します。
MACVTAP / IPVTAPを使用すると、TUN / TAPとブリッジドライバーの組み合わせを単一のモジュールに置き換えることができます。
通常、MACVLAN / IPVLANは、ゲストとホストの両方が、ホストが接続されているスイッチに直接表示されるようにするために使用されます。MACVTAPとIPVTAPの違いは、MACVLAN / IPVLANの場合と同じです。

<p align="center"><img src="/assets/img/2020-10-28-linux-interfaces/macvtap.png" ></p>

macvtapの設定は以下のようになります。

```shell
ip link add link eth0 name macvtap0 type macvtap
```

## VETH (Virtual Ethernet)

VETH（仮想イーサネット）デバイスは、ローカルイーサネットトンネルです。次の図に示すようにデバイスはペアで作成されます。
ペアの一方のデバイスで送信されたパケットは、もう一方のデバイスですぐに受信されます。いずれかのデバイスがダウンしている場合、ペアのリンク状態はダウンしています。

それぞれの名前空間がホストの名前空間(global)と通信する必要がある場合、または相互に通信する必要がある場合は、VETH構成を使用します。

VETH構成をセットアップする方法は次のとおりです。

```shell
ip netns add net1
ip netns add net2
ip link add veth1 netns net1 type veth peer name veth2 netns net2
```

<p align="center"><img src="/assets/img/2020-10-28-linux-interfaces/veth.png" ></p>

## NLMON (NetLink MONitor)

NLMONはNetlinkモニターデバイスです。システムのNetlinkメッセージを監視する場合は、NLMONデバイスを使用します。
NLMONデバイスを作成する方法は次のとおりです。

```shell
ip link add nlmon0 type nlmon
ip link set nlmon0 up
tcpdump -i nlmon0 -w nlmsg.pcap
```

これにより、`nlmon0` という名前のNLMONデバイスが作成され、セットアップされます。パケットスニファ（たとえばtcpdump）を使用して、Netlinkメッセージをキャプチャします。Wiresharkの最近のバージョンは、Netlinkメッセージのデコード機能を備えています。

## Dummy interface

ダミーインターフェースは、ループバックインターフェースのように完全に仮想です。ダミーインターフェースの目的は、実際にパケットを送信せずにパケットをルーティングするデバイスを提供することです。
ダミーインターフェースは主にテストとデバッグに使用されています。

ダミーインターフェースを作成する方法は次のとおりです。

```shell
ip link add dummy1 type dummy
ip addr add 1.1.1.1/24 dev dummy1
ip link set dummy1 up
```

## netdevsim

netdevsimは、さまざまなネットワークAPIのテストに使用されるシミュレートされたネットワークデバイスです。現時点では、ハードウェアオフロード、tc / XDPBPFおよびSR-IOVのテストに特に重点を置いています。

netdevsimデバイスは次のように作成できます

```shell
ip link add dev sim0 type netdevsim
ip link set dev sim0 up
```

tcオフロードを有効にするには：

```shell
ethtool -K sim0 hw-tc-offload on
```

XDPBPFまたはtcBPFプログラムをロードするには：

```shell
ip link set dev sim0 xdpoffload obj prog.o
```

