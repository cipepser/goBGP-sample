# GoBGPでEVPN/VXLANを試す

[GoBGPでEVPN/VXLANを試す](http://skjune12.hatenadiary.com/entry/2017/12/09/235550)を自分でも手を動かしてみる。

[vagrant-evpn-vxlan](https://github.com/skjune12/vagrant-evpn-vxlan/blob/master/README.md)のVagrantfileで環境構築。
ただし、このVagrantfileだと`ubuntu/xenial64`が`box add`できないので
Vagrantfileに以下を追記

```
  Vagrant::DEFAULT_SERVER_URL.replace('https://vagrantcloud.com')
```

`vagrant up`は結構時間が掛かる(手元のMacで30分弱)

## 手順

### I/Fの設定、goplaneの起動

```
> vagrant ssh gobgp1
> sudo -i
> ~/config/config-interface.sh

> goplane -f ~/config/multiple-sites.conf
```

`gobgp1`を`gobgp2`,`gobgp1`に変えて3つとも設定するする


[vagrant-evpn-vxlan](https://github.com/skjune12/vagrant-evpn-vxlan/blob/master/README.md)の手順にある`ip netns exec vxlan ping -c 5 192.168.1.4`をやる前に、VTEPの事前状態を確認する。

### 各ノードのI/F情報

```
root@gobgp1:~# ifconfig
br10      Link encap:Ethernet  HWaddr 12:3c:fd:37:39:bc
          inet6 addr: fe80::7c68:66ff:fe7b:6af6/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1450  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:9 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:0 (0.0 B)  TX bytes:738 (738.0 B)

enp0s3    Link encap:Ethernet  HWaddr 02:24:89:81:87:2f
          inet addr:10.0.2.15  Bcast:10.0.2.255  Mask:255.255.255.0
          inet6 addr: fe80::24:89ff:fe81:872f/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:296273 errors:0 dropped:0 overruns:0 frame:0
          TX packets:88024 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:413312058 (413.3 MB)  TX bytes:5598554 (5.5 MB)

enp0s8    Link encap:Ethernet  HWaddr 08:00:27:59:3c:5e
          inet addr:10.0.12.10  Bcast:10.0.12.255  Mask:255.255.255.0
          inet6 addr: fe80::a00:27ff:fe59:3c5e/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:300 errors:0 dropped:0 overruns:0 frame:0
          TX packets:167 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:44470 (44.4 KB)  TX bytes:13065 (13.0 KB)

lo        Link encap:Local Loopback
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:276 errors:0 dropped:0 overruns:0 frame:0
          TX packets:276 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1
          RX bytes:23236 (23.2 KB)  TX bytes:23236 (23.2 KB)

veth1     Link encap:Ethernet  HWaddr ae:65:ac:15:bf:35
          inet addr:192.168.1.2  Bcast:192.168.1.255  Mask:255.255.255.0
          inet6 addr: fe80::ac65:acff:fe15:bf35/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1450  Metric:1
          RX packets:8 errors:0 dropped:0 overruns:0 frame:0
          TX packets:20 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:648 (648.0 B)  TX bytes:1656 (1.6 KB)

vtep10    Link encap:Ethernet  HWaddr 12:3c:fd:37:39:bc
          inet6 addr: fe80::103c:fdff:fe37:39bc/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:16 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)

```

```
root@gobgp2:~# ifconfig
br10      Link encap:Ethernet  HWaddr 56:ae:19:f8:8b:65
          inet6 addr: fe80::30dd:a4ff:fee1:2eaf/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1450  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:10 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:0 (0.0 B)  TX bytes:828 (828.0 B)

enp0s3    Link encap:Ethernet  HWaddr 02:24:89:81:87:2f
          inet addr:10.0.2.15  Bcast:10.0.2.255  Mask:255.255.255.0
          inet6 addr: fe80::24:89ff:fe81:872f/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:295481 errors:0 dropped:0 overruns:0 frame:0
          TX packets:85246 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:413268119 (413.2 MB)  TX bytes:5410389 (5.4 MB)

enp0s8    Link encap:Ethernet  HWaddr 08:00:27:a5:e6:6e
          inet addr:10.0.12.20  Bcast:10.0.12.255  Mask:255.255.255.0
          inet6 addr: fe80::a00:27ff:fea5:e66e/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:278 errors:0 dropped:0 overruns:0 frame:0
          TX packets:174 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:39794 (39.7 KB)  TX bytes:13464 (13.4 KB)

enp0s9    Link encap:Ethernet  HWaddr 08:00:27:71:66:10
          inet addr:10.0.23.10  Bcast:10.0.23.255  Mask:255.255.255.0
          inet6 addr: fe80::a00:27ff:fe71:6610/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:260 errors:0 dropped:0 overruns:0 frame:0
          TX packets:178 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:37595 (37.5 KB)  TX bytes:13728 (13.7 KB)

lo        Link encap:Local Loopback
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:200 errors:0 dropped:0 overruns:0 frame:0
          TX packets:200 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1
          RX bytes:16090 (16.0 KB)  TX bytes:16090 (16.0 KB)

veth1     Link encap:Ethernet  HWaddr 5e:c1:2d:87:d0:ec
          inet addr:192.168.1.3  Bcast:192.168.1.255  Mask:255.255.255.0
          inet6 addr: fe80::5cc1:2dff:fe87:d0ec/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1450  Metric:1
          RX packets:8 errors:0 dropped:0 overruns:0 frame:0
          TX packets:32 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:648 (648.0 B)  TX bytes:2664 (2.6 KB)

vtep10    Link encap:Ethernet  HWaddr 56:ae:19:f8:8b:65
          inet6 addr: fe80::54ae:19ff:fef8:8b65/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:18 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)

```

```
root@gobgp3:~# ifconfig
br10      Link encap:Ethernet  HWaddr 42:d0:89:0b:76:f1
          inet6 addr: fe80::b499:e3ff:feae:45b0/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1450  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:9 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:0 (0.0 B)  TX bytes:738 (738.0 B)

enp0s3    Link encap:Ethernet  HWaddr 02:24:89:81:87:2f
          inet addr:10.0.2.15  Bcast:10.0.2.255  Mask:255.255.255.0
          inet6 addr: fe80::24:89ff:fe81:872f/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:294672 errors:0 dropped:0 overruns:0 frame:0
          TX packets:78563 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:413302082 (413.3 MB)  TX bytes:5001963 (5.0 MB)

enp0s8    Link encap:Ethernet  HWaddr 08:00:27:f7:11:88
          inet addr:10.0.23.20  Bcast:10.0.23.255  Mask:255.255.255.0
          inet6 addr: fe80::a00:27ff:fef7:1188/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:280 errors:0 dropped:0 overruns:0 frame:0
          TX packets:172 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:36671 (36.6 KB)  TX bytes:13430 (13.4 KB)

lo        Link encap:Local Loopback
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:105 errors:0 dropped:0 overruns:0 frame:0
          TX packets:105 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1
          RX bytes:9033 (9.0 KB)  TX bytes:9033 (9.0 KB)

veth1     Link encap:Ethernet  HWaddr 42:d0:89:0b:76:f1
          inet addr:192.168.1.5  Bcast:192.168.1.255  Mask:255.255.255.0
          inet6 addr: fe80::40d0:89ff:fe0b:76f1/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1450  Metric:1
          RX packets:8 errors:0 dropped:0 overruns:0 frame:0
          TX packets:18 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:648 (648.0 B)  TX bytes:1476 (1.4 KB)

vtep10    Link encap:Ethernet  HWaddr 8e:a1:16:65:0a:ed
          inet6 addr: fe80::8ca1:16ff:fe65:aed/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:16 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)

```

### BGP neighborsの確認

```
root@gobgp1:~# gobgp neigh
Peer          AS  Up/Down State       |#Received  Accepted
10.0.12.20 65000 00:01:54 Establ      |        2         2
10.0.23.20 65000 00:16:27 Establ      |        2         2
```

```
root@gobgp2:~# gobgp neigh
Peer          AS  Up/Down State       |#Received  Accepted
10.0.12.10 65000 00:01:59 Establ      |        2         2
10.0.23.20 65000 00:01:57 Establ      |        2         2
```

```
root@gobgp3:~# gobgp neigh
Peer          AS  Up/Down State       |#Received  Accepted
10.0.12.10 65000 00:18:57 Establ      |        2         2
10.0.23.10 65000 00:04:22 Establ      |        2         2
```

gobgp1, 2, 3で`Establ`になっており、peerを張れている。

### VTEPのstatus確認(事前)

```
root@gobgp1:~# gobgp global rib -a evpn
    Network                                           Next Hop             AS_PATH              Age        Attrs
*>  [type:multicast][rd:65000:10][etag:10][ip:1.1.1.1]0.0.0.0                                   00:33:26   [{Origin: i} {Pmsi: type: ingress-repl, label: 0, tunnel-id: 1.1.1.1} {Extcomms: [65000:10]}]
*>  [type:multicast][rd:65000:10][etag:10][ip:2.2.2.2]10.0.12.20                                00:15:52   [{Origin: i} {LocalPref: 100} {Extcomms: [65000:10]} {Pmsi: type: ingress-repl, label: 0, tunnel-id: 2.2.2.2}]
*>  [type:multicast][rd:65000:10][etag:10][ip:3.3.3.3]10.0.23.20                                00:30:25   [{Origin: i} {LocalPref: 100} {Extcomms: [65000:10]} {Pmsi: type: ingress-repl, label: 0, tunnel-id: 3.3.3.3}]
```

```
root@gobgp2:~#  gobgp global rib -a evpn
    Network                                           Next Hop             AS_PATH              Age        Attrs
*>  [type:multicast][rd:65000:10][etag:10][ip:1.1.1.1]10.0.12.10                                00:15:58   [{Origin: i} {LocalPref: 100} {Extcomms: [65000:10]} {Pmsi: type: ingress-repl, label: 0, tunnel-id: 1.1.1.1}]
*>  [type:multicast][rd:65000:10][etag:10][ip:2.2.2.2]0.0.0.0                                   00:16:10   [{Origin: i} {Pmsi: type: ingress-repl, label: 0, tunnel-id: 2.2.2.2} {Extcomms: [65000:10]}]
*>  [type:multicast][rd:65000:10][etag:10][ip:3.3.3.3]10.0.23.20                                00:15:56   [{Origin: i} {LocalPref: 100} {Extcomms: [65000:10]} {Pmsi: type: ingress-repl, label: 0, tunnel-id: 3.3.3.3}]
```

```
root@gobgp3:~#  gobgp global rib -a evpn
    Network                                           Next Hop             AS_PATH              Age        Attrs
*>  [type:multicast][rd:65000:10][etag:10][ip:1.1.1.1]10.0.12.10                                00:30:33   [{Origin: i} {LocalPref: 100} {Extcomms: [65000:10]} {Pmsi: type: ingress-repl, label: 0, tunnel-id: 1.1.1.1}]
*>  [type:multicast][rd:65000:10][etag:10][ip:2.2.2.2]10.0.23.10                                00:15:58   [{Origin: i} {LocalPref: 100} {Extcomms: [65000:10]} {Pmsi: type: ingress-repl, label: 0, tunnel-id: 2.2.2.2}]
*>  [type:multicast][rd:65000:10][etag:10][ip:3.3.3.3]0.0.0.0                                   00:30:52   [{Origin: i} {Pmsi: type: ingress-repl, label: 0, tunnel-id: 3.3.3.3} {Extcomms: [65000:10]}]
```

### ping

#### gobgp1 -> gobgp2

`ifconfig`だとgobgp2は`192.168.1.3`では？

```
root@gobgp1:~# ip netns exec vxlan ping -c 5 192.168.1.4
PING 192.168.1.4 (192.168.1.4) 56(84) bytes of data.
64 bytes from 192.168.1.4: icmp_seq=1 ttl=64 time=1002 ms
64 bytes from 192.168.1.4: icmp_seq=2 ttl=64 time=2.01 ms
64 bytes from 192.168.1.4: icmp_seq=3 ttl=64 time=0.974 ms
64 bytes from 192.168.1.4: icmp_seq=4 ttl=64 time=0.505 ms
64 bytes from 192.168.1.4: icmp_seq=5 ttl=64 time=0.812 ms

--- 192.168.1.4 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 4006ms
rtt min/avg/max/mdev = 0.505/201.409/1002.745/400.668 ms, pipe 2
```

#### gobgp1 -> gobgp3

`ifconfig`だとgobgp2は`192.168.1.5`では？

`config-interface.sh`を見ると以下のように設定されているのでよさそう。`netns`の勉強が必要。
```
ip netns exec vxlan ifconfig veth0 192.168.1.4 up
```


```
root@gobgp1:~# ip netns exec vxlan ping -c 5 192.168.1.6
PING 192.168.1.6 (192.168.1.6) 56(84) bytes of data.
64 bytes from 192.168.1.6: icmp_seq=2 ttl=64 time=0.799 ms
64 bytes from 192.168.1.6: icmp_seq=3 ttl=64 time=0.883 ms
64 bytes from 192.168.1.6: icmp_seq=4 ttl=64 time=0.866 ms
64 bytes from 192.168.1.6: icmp_seq=5 ttl=64 time=1.13 ms

--- 192.168.1.6 ping statistics ---
5 packets transmitted, 4 received, 20% packet loss, time 4011ms
rtt min/avg/max/mdev = 0.799/0.920/1.135/0.132 ms
```

### VTEPのstatus確認(事後)

```
root@gobgp1:~# gobgp global rib -a evpn
    Network                                                                                       Next Hop             AS_PATH              Age        Attrs
*>  [type:macadv][rd:0:0][esi:single-homed][etag:10][mac:ae:93:09:e9:20:f6][ip:<nil>][labels:[10]]10.0.12.20                                00:00:16   [{Origin: i} {LocalPref: 100} {Extcomms: [VXLAN]}]
*>  [type:macadv][rd:0:0][esi:single-homed][etag:10][mac:ca:5d:43:b6:0c:a7][ip:<nil>][labels:[10]]0.0.0.0                                   00:01:44   [{Origin: i} {Extcomms: [VXLAN]}]
*>  [type:macadv][rd:0:0][esi:single-homed][etag:10][mac:ce:a1:43:5b:f7:fc][ip:<nil>][labels:[10]]10.0.23.20                                00:00:10   [{Origin: i} {LocalPref: 100} {Extcomms: [VXLAN]}]
*>  [type:multicast][rd:65000:10][etag:10][ip:1.1.1.1]0.0.0.0                                   00:52:17   [{Origin: i} {Pmsi: type: ingress-repl, label: 0, tunnel-id: 1.1.1.1} {Extcomms: [65000:10]}]
*>  [type:multicast][rd:65000:10][etag:10][ip:2.2.2.2]10.0.12.20                                00:34:43   [{Origin: i} {LocalPref: 100} {Extcomms: [65000:10]} {Pmsi: type: ingress-repl, label: 0, tunnel-id: 2.2.2.2}]
*>  [type:multicast][rd:65000:10][etag:10][ip:3.3.3.3]10.0.23.20                                00:49:16   [{Origin: i} {LocalPref: 100} {Extcomms: [65000:10]} {Pmsi: type: ingress-repl, label: 0, tunnel-id: 3.3.3.3}]
```

```
oot@gobgp2:~# gobgp global rib -a evpn
    Network                                                                                       Next Hop             AS_PATH              Age        Attrs
*>  [type:macadv][rd:0:0][esi:single-homed][etag:10][mac:ae:93:09:e9:20:f6][ip:<nil>][labels:[10]]0.0.0.0                                   00:00:33   [{Origin: i} {Extcomms: [VXLAN]}]
*>  [type:macadv][rd:0:0][esi:single-homed][etag:10][mac:ca:5d:43:b6:0c:a7][ip:<nil>][labels:[10]]10.0.12.10                                00:02:01   [{Origin: i} {LocalPref: 100} {Extcomms: [VXLAN]}]
*>  [type:macadv][rd:0:0][esi:single-homed][etag:10][mac:ce:a1:43:5b:f7:fc][ip:<nil>][labels:[10]]10.0.23.20                                00:00:27   [{Origin: i} {LocalPref: 100} {Extcomms: [VXLAN]}]
*>  [type:multicast][rd:65000:10][etag:10][ip:1.1.1.1]10.0.12.10                                00:35:00   [{Origin: i} {LocalPref: 100} {Extcomms: [65000:10]} {Pmsi: type: ingress-repl, label: 0, tunnel-id: 1.1.1.1}]
*>  [type:multicast][rd:65000:10][etag:10][ip:2.2.2.2]0.0.0.0                                   00:35:12   [{Origin: i} {Pmsi: type: ingress-repl, label: 0, tunnel-id: 2.2.2.2} {Extcomms: [65000:10]}]
*>  [type:multicast][rd:65000:10][etag:10][ip:3.3.3.3]10.0.23.20                                00:34:58   [{Origin: i} {LocalPref: 100} {Extcomms: [65000:10]} {Pmsi: type: ingress-repl, label: 0, tunnel-id: 3.3.3.3}]
```

```
root@gobgp3:~# gobgp global rib -a evpn
    Network                                                                                       Next Hop             AS_PATH              Age        Attrs
*>  [type:macadv][rd:0:0][esi:single-homed][etag:10][mac:ae:93:09:e9:20:f6][ip:<nil>][labels:[10]]10.0.23.10                                00:00:53   [{Origin: i} {LocalPref: 100} {Extcomms: [VXLAN]}]
*>  [type:macadv][rd:0:0][esi:single-homed][etag:10][mac:ca:5d:43:b6:0c:a7][ip:<nil>][labels:[10]]10.0.12.10                                00:02:21   [{Origin: i} {LocalPref: 100} {Extcomms: [VXLAN]}]
*>  [type:macadv][rd:0:0][esi:single-homed][etag:10][mac:ce:a1:43:5b:f7:fc][ip:<nil>][labels:[10]]0.0.0.0                                   00:00:46   [{Origin: i} {Extcomms: [VXLAN]}]
*>  [type:multicast][rd:65000:10][etag:10][ip:1.1.1.1]10.0.12.10                                00:49:54   [{Origin: i} {LocalPref: 100} {Extcomms: [65000:10]} {Pmsi: type: ingress-repl, label: 0, tunnel-id: 1.1.1.1}]
*>  [type:multicast][rd:65000:10][etag:10][ip:2.2.2.2]10.0.23.10                                00:35:19   [{Origin: i} {LocalPref: 100} {Extcomms: [65000:10]} {Pmsi: type: ingress-repl, label: 0, tunnel-id: 2.2.2.2}]
*>  [type:multicast][rd:65000:10][etag:10][ip:3.3.3.3]0.0.0.0                                   00:50:13   [{Origin: i} {Pmsi: type: ingress-repl, label: 0, tunnel-id: 3.3.3.3} {Extcomms: [65000:10]}]
```

## References
* [GoBGPでEVPN/VXLANを試す](http://skjune12.hatenadiary.com/entry/2017/12/09/235550)
* [vagrant-evpn-vxlan](https://github.com/skjune12/vagrant-evpn-vxlan/blob/master/README.md)
* [vagrant box update - Fails with 404 Not Found error #9442](https://github.com/hashicorp/vagrant/issues/9442)