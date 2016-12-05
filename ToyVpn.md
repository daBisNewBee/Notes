# 使用ToyVpn搭建简易VPN

### 什么是ToyVpn
- 基于VpnService的实现
- 封装的隧道包协议为__IP In UDP__
- 包括了最简单的handshake + forwarding
- [ToyVpn Google](https://github.com/android/platform_development/tree/master/samples/ToyVpn)

### 拓扑

![toyVpn](https://github.com/daBisNewBee/Notes/blob/master/pic/toyVpn.jpg)

- VpnServer的eth1为公网地址，eth2接私有地址，可访问多个应用服务器
- VpnClient可通过Wifi或者移动数据访问到VpnServer
- 192.168.1.9为DNS Server，可解析www.myweb.com
- VpnClient可拨号至VpnServer，作为隧道的两端

### 目标
- 手机浏览器通过IP地址192.168.1.8访问Web应用服务器
- 手机浏览器通过域名www.myweb.com访问Web应用服务器

### 步骤
- 配置vpn.sh并运行./vpn.sh：

```                    
#/bin/bash

iptables -t nat -F
ip tuntap del dev tun0 mode tun

# Enable IP forwarding
echo 1 > /proc/sys/net/ipv4/ip_forward

# Pick a range of private addresses and perform NAT over eth2.
iptables -t nat -A POSTROUTING -s 10.0.0.0/8 -o eth2 -j MASQUERADE

# Create a TUN interface.
ip tuntap add dev tun0 mode tun

# Set the addresses and bring up the interface.
ifconfig tun0 10.0.0.1 dstaddr 10.0.0.2 up

# Create a server on port 61195 with shared secret "mySecret".
./ToyVpnServer tun0 61195 mySecret -m 1400 -a 10.0.0.2 32  -r 192.168.1.8 32
```
- 客户端设置Server Address:101.231.201.53  Server Port:61195 Shared Secrert:ySecrect  ，点击connected就可以访问 192.168.1.8了
- 通过www.myweb.com访问服务器，需修改vpn.sh：

```
./ToyVpnServer tun0 61195 mySecret -m 1400 -a 10.0.0.2 32  -d 192.168.1.9  -r 192.168.1.0 24
(假设www.myweb.com为192.168.1.100 ,即也在192.168.1.0/24网段)
```

### 扩展
- 保护个人Android终端上所有外网流量，需要VpnServer也有接入外网的能力。比如通过上述的eth1来接入外网并转发手机流量：

```
iptables -t nat -A POSTROUTING -s 10.0.0.0/8 -o eth1 -j MASQUERADE
./ToyVpnServer tun0 61195 mySecret -m 1400 -a 10.0.0.2 32   -r 0.0.0.0 0
```
这样手机在任意Wifi热点，就可以随意接入，而不用担心被监听了。当然VpnService需要对截获的流量进行加解密，ToyVpn也可以直接替换成现有的实现，比如OpenVPN
- 免root防火墙，在VpnService中对截获的流量进行处理，比如：

```
  private ParcelFileDescriptor mInterface;
 mInterface = builder.setSession(mServerAddress)
                .setConfigureIntent(mConfigureIntent)
                .establish();
FileInputStream in = new FileInputStream(mInterface.getFileDescriptor());

ByteBuffer packet = ByteBuffer.allocate(32767);
 int length = in.read(packet.array());
                if (length > 0) {
                    // Write the outgoing packet to the tunnel.
                   // do write to tunnel or Drop
                   // tunnel.write(packet);
                }

```
- 一个基于VpnService实现的免Root下的开源防火墙[NetGuard](https://github.com/M66B/NetGuard)
