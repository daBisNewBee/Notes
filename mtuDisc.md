### 一次使用ping探测链路最小mtu的过程
#### 步骤
- 设置ping携带数据大小（-s），设置不分片（-D）

**sh-3.2# ping 8.8.8.8 -s 1465 -D**

```
PING 8.8.8.8 (8.8.8.8): 1465 data bytes
36 bytes from 192.168.0.1: frag needed and DF set (MTU 1492)
Vr HL TOS  Len   ID Flg  off TTL Pro  cks      Src      Dst
 4  5  00 d505 bfdf   0 0000  40  01 a422 192.168.0.110  8.8.8.8 
```

- 反复调整size，直到ping通。

**sh-3.2# ping 8.8.8.8 -s 1464 -D**

```
PING 8.8.8.8 (8.8.8.8): 1464 data bytes
1472 bytes from 8.8.8.8: icmp_seq=0 ttl=60 time=5.594 ms
```
- 另。当找到了正确的size，也可能仍旧ping不通。因为对端已经禁ping，但是报错与size过大是不同的。

```
userdeMacBook-Pro:git user$ ping 101.231.201.62 -s 1464 -D
PING 101.231.201.62 (101.231.201.62): 1464 data bytes
Request timeout for icmp_seq 0
Request timeout for icmp_seq 1

userdeMacBook-Pro:git user$ ping 101.231.201.62 -s 1465 -D
PING 101.231.201.62 (101.231.201.62): 1465 data bytes
36 bytes from 192.168.0.1: frag needed and DF set (MTU 1492)
Vr HL TOS  Len   ID Flg  off TTL Pro  cks      Src      Dst
 4  5  00 d505 0246   0 0000  40  01 42a6 192.168.0.110  101.231.201.62

```

#### 相关坑
- 必须设置`DF（dont fragment）`。否则ping在路由器上会被自动分片。
- 若预测MTU为1500，则`packetsize`设置为1472可通过，1473失败，预留28差值。因为ip头长度为20byte，icmp头长度为8byte，MTU = iphl + icmphl + icmpdatal。
- `packetsize `为icmp数据段长度，不包括icmp头。因此若`packetsize `为1，则实际发送包的长度为43。（在WS上抓包看到的长度，1 + 8 + 20 + 14 = 43）
- 当`packetsize `设置>=8时，比如9，WS上看到icmp数据段长度只有1? 正常现象。由于icmp自动判断当载荷长度>=8时，默认填充前8个字节为timestamp，实际datal并没有减少;载荷长度<8时，就不设置timestamp字段。

```
 If the data space is at least eight bytes large, ping
 uses the first eight bytes of this space to include a
 timestamp which it uses in the computation of round
 trip times.  If less than eight bytes of pad are 
 specified, no round trip times are given.

```
#### 其他用法
> 当存在VPN时，虚拟设备使用tun，可使用ping估算单次可携带数据长度，即实际有效MTU。避免应用层单个发包过大，导致的VPN加密访问应用服务端失败。

- 比如，当在环境最小MTU为1492时，抓包可看到长度应为1492 + 14 = 1506.
- 首次尝试`packetsize `数值为1390。

```
sh-3.2# ping 192.168.1.8 -s 1390 -D
PING 192.168.1.8 (192.168.1.8): 1390 data bytes
1398 bytes from 192.168.1.8: icmp_seq=0 ttl=63 time=138.478 ms
```
WS上可看到实际包长为1503（此时已逼近可接受长度1506），openvpn的datal为1460.可估算 1460-（1390 + 8 + 20）= 42byte，为加密产生的附加长度。

- 增大`packetsize `为1391，再次尝试

```
sh-3.2# ping 192.168.1.8 -s 1391 -D
PING 192.168.1.8 (192.168.1.8): 1391 data bytes
Request timeout for icmp_seq 0
```
WS上此时openvpn包已被IP层分片，已无完整openvpn包。

- 结论：tun实际有效MTU长度为 1390 + 8 = 1398（根据icmp估算，其他协议结论相同，只不过协议头长度不同，数据段长度不同），即应用层产生的数据包长度不超过1398。
