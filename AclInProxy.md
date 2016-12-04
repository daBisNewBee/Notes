#  如何根据TCP连接的源端口索引来源APP（即UID）？
### 原理
- /proc/net/tcp文件记录了ipv4下所有tcp连接的情况
- [/proc/net/tcp中各项参数说明](http://img.blog.csdn.net/20140311180218984?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvanVzdGxpbnV4MjAxMA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
- 其中主要用到了对uid项的解析
- uid在Android中的说明

```
作用：Android为单用户设计，uid用于数据共享
常见的uid含义：
    0               rootID
    1000       SystemID
    1001       PhoneID
    >10000  AppID
查找app的uid：“u0_axx对应的uid为10000+xx”
    例如：$ ps |grep koal 
     u0_a263   16798 327   1615480 76420 ffffffff b6f3f970 S koal.ssl
     则 包名为“koal.ssl”的应用uid为10263
```
- 处于不同state的socket，对应的uid不同（接管该socket的对象不同）

```
state: 01 （TCP_ESTABLISHED）     uid：APP的uid
state: 06  （TCP_TIME_WAIT）         uid:   0 （该socket已被root用户接管）
其他
```
- 因此，当tcpClient与tcpServer建立连接后，可以在tcpServer获取到tcpClient的源IP和源端口，然后解析“/proc/net/tcp”得到对应的socket，并确认处于“TCP_ESTABLISHED”，此时就可获得的tcpClient所在的uid，并根据制定的uid规则，对socket对出相应的处理

### 实例
- 一共有两个app应用：tcpServer和tcpClient。tcpServer在lo listen 50001，tcpClient向lo的50001发起connect，成功连接后写入数据。现需要获得在tcpServer获得tcpClient的uid。
- tcpServer在lo listen 50001

```
shell@klte:/ $ cat /proc/net/tcp
  sl  local_address rem_address   st tx_queue rx_queue tr tm->when retrnsmt   uid  timeout inode                                   
   0: 00000000:C351 00000000:0000 0A 00000000:00000000 00:00000000 00000000 10265        0 8096599 1 00000000 100 0 0 10 -1      
```
-  tcpClient向lo的50001发起connect，且tcpServer成功accept

```
shell@klte:/ $ cat /proc/net/tcp
  sl  local_address rem_address   st tx_queue rx_queue tr tm->when retrnsmt   uid  timeout inode                                   
   0: 00000000:C351 00000000:0000 0A 00000000:00000000 00:00000000 00000000 10265        0 8096599 1 00000000 100 0 0 10 -1        
   1: 0100007F:C351 0100007F:9EEB 01 00000000:00000000 00:00000000 00000000 10265        0 8096600 1 00000000 21 4 30 10 -1        
   2: 0100007F:9EEB 0100007F:C351 01 00000000:00000000 00:00000000 00000000 10183        0 8131292 1 00000000 21 0 0 10 -1   
```
可以看到IP为“0100007F”（127.0.0.1），PORT为“9EEB”（40683）的建立起状态为“01”的socket，该tcpClient对应的uid为“10183”

- 多个方法验证tcpClient的uid是否正确

```
根据包名使用ps查看：
root@klte:/proc/3235 # ps|grep tcpclient
u0_a183   3235  315   1592948 50792 ffffffff b6ecd970 S com.example.tcpclient

根据进程号，在进程属性中查看：
root@klte:/proc/3235 # cat cgroup
2:cpu:/apps
1:cpuacct:/uid_10183/pid_3235

根据socket的inode，查找归属uid：
root@klte:/proc/3235/fd # ll|grep 8131292
lrwx------ u0_a183  u0_a183           2016-11-20 23:35 38 -> socket:[8131292]

```

### 在stunnel中的访问控制的应用
- 可以在“accept_connection”中，对accept的new socket做相应处理，类似于超过本地最大连接数“max_clients”的处理。

```
connfd = accept(listenfd, &their_addr, &sin_size);
client_SrcPort = ntohs(their_addr.sin_port);
uid = find( "/proc/net/tcp", "local_address  == 127.0.0.1:client_SrcPort && rem_address == 127.0.0.1:localPort && st == 01"  );
if matchPolicy(uid)
 {
      // doDrop;
      log("Connection rejected: uid:%d",uid);
      close(connfd);
}

```

### 参考
- [Android中如何根据端口号寻找对应的进程](http://blog.csdn.net/myarrow/article/details/8930827)
- [通过执行netstat及cat /proc/net/tcp查看正在运行应用的本地端口号pid和uid以及对方的IP和端口号  ](http://wxmijl.blog.163.com/blog/static/132459282013773122750/)
- [网络监控与/proc/net/tcp文件 ](http://chenlinux.com/2010/07/28/monitor-netflow-by-proc_net_tcp/)
- [Exploring the /proc/net/ Directory](http://www.linuxdevcenter.com/pub/a/linux/2000/11/16/LinuxAdmin.html)
- [图解TCP状态机](http://blog.csdn.net/jedihy/article/details/17043839)
