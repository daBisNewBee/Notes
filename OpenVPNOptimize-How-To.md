## 从一个简单的优化OpenVPN状态协议过程看如何进行协议优化
### 背景与需求
- VPN（OpenVPN）在移动设备环境的普遍使用
- 移动环境资源受限，包括电量、流量
- VPN在链路出现故障时，未能及时停止连接并通知用户；链路恢复时未能及时恢复服务

### 设计思路
![asd](https://github.com/daBisNewBee/Notes/blob/master/pic/OpenVPN-ExtendProto.png)
其中，OpenVPN将隧道的建立过程分为：WAIT、AUTH、GET_CONFIG、IFCONFIG、ADD_ROUTES、OPENTUN、CONNECTED、EXITING。当错误发生在其中一个阶段，由于出错握手便会无法进入下一阶段。超时后OpenVPN便会发出重连信号，开始重新连接。若不对其进行额外的处理，OpenVPN便会一直无条件的重连下去。这对本来就不具备连接条件的VPN两端，无限的重连对移动设备就是种伤害。
因此，添加的自定义协议如下：

|协议标识|协议名称|含义|处理|
|---|---|---|---|
|NO_RESPONSE|未收到服务端响应|网络不通或者服务端进程停止|停止服务|
|AUTH_FAILED|认证失败|服务端证书链验证客户端提交的证书失败|停止服务|
|TUNNEL_DISABLED|拒绝接入|服务端拒绝该客户端接|停止服务，并输出错误码|
|TIME_OUT|连接超时|隧道建立后，在一定时间内未收到服务端数据|客户端服务完整重启，包括策略的同步等|

备注： 不再单独处理服务端发出的"reject_client"的包，客户端收到该包后会触发SIGUSR1，发出RECONNECTING。仍旧根据收到RECONNECTING所属的阶段来判断服务状态，reject_client只是将RECONNECTING发出的时间提前了。

### 结论
一个协议优化的过程如下:
- 搜集服务出错情形
- 使用扩展协议标识错误情形
- 对抽象出的错误情形进行控制

可以看到，一旦发生错误时，只需简单的停止服务、重启服务即可。对服务的控制很简单，关键在于对问题的**抽象**。具体为：
- 对不具备连接条件的情况进行抽象，并及时停止服务，不再无意义的操作
- 对具备连接条件的情况进行抽象，及时操作恢复状态。

### 参考
- [OpenVPN状态原文](https://github.com/daBisNewBee/Notes/blob/master/OpenVPNStateLog.md)
