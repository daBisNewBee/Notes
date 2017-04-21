### OpenVPN状态原文
NO_RESPONSE:
```
V/com.vpn.sdk(25151): state:WAIT msg:,,
V/com.vpn.sdk(25151): state:RECONNECTING msg:ping-restart,,
V/com.vpn.sdk(25151): state:WAIT msg:,,
V/com.vpn.sdk(25151): state:RECONNECTING msg:ping-restart,,
```

AUTH_FAILED:
```
服务端开启"reject-client"
V/com.vpn.sdk(11995): state:WAIT msg:,,
V/com.vpn.sdk(11995): state:AUTH msg:,,
V/com.vpn.sdk(11995): state:RECONNECTING msg:REJECT_CLIENT:No TLS state for client!,,

服务端未开启"reject-client"
V/com.vpn.sdk(25151): state:WAIT msg:,,
V/com.vpn.sdk(25151): state:AUTH msg:,,
V/com.vpn.sdk(25151): state:RECONNECTING msg:ping-restart,,
```

TUNNEL_DISABLED：
```
V/com.vpn.sdk(11995): state:WAIT msg:,,
V/com.vpn.sdk(11995): state:AUTH msg:,,
V/com.vpn.sdk(11995): state:GET_CONFIG msg:,,
V/com.vpn.sdk(11995): state:RECONNECTING msg:TUNNEL_DISABLED:5,,
```

TIME_OUT：
```
服务端开启"reject-client"
V/com.vpn.sdk(11995): state:WAIT msg:,,
V/com.vpn.sdk(11995): state:AUTH msg:,,
V/com.vpn.sdk(11995): state:GET_CONFIG msg:,,
V/com.vpn.sdk(11995): state:ASSIGN_IP msg:,128.129.0.2,
V/com.vpn.sdk(11995): state:ADD_ROUTES msg:,,
V/com.vpn.sdk(11995): state:CONNECTED msg:SUCCESS,128.129.0.2,192.168.41.155
V/com.vpn.sdk(11995): state:RECONNECTING msg:REJECT_CLIENT:No TLS state for client!,,
V/com.vpn.sdk(11995): state:WAIT msg:,,
V/com.vpn.sdk(11995): state:AUTH msg:,,
V/com.vpn.sdk(11995): state:GET_CONFIG msg:,,
V/com.vpn.sdk(11995): state:ASSIGN_IP msg:,128.129.0.2,
V/com.vpn.sdk(11995): state:ADD_ROUTES msg:,,
V/com.vpn.sdk(11995): state:CONNECTED msg:SUCCESS,128.129.0.2,192.168.41.155

服务端未开启"reject-client"：
V/com.vpn.sdk(11995): state:WAIT msg:,,
V/com.vpn.sdk(11995): state:AUTH msg:,,
V/com.vpn.sdk(11995): state:GET_CONFIG msg:,,
V/com.vpn.sdk(11995): state:ASSIGN_IP msg:,128.129.0.2,
V/com.vpn.sdk(11995): state:ADD_ROUTES msg:,,
V/com.vpn.sdk(11995): state:CONNECTED msg:SUCCESS,128.129.0.2,192.168.41.155
V/com.vpn.sdk(11995): state:RECONNECTING msg:ping-restart,,
V/com.vpn.sdk(11995): state:WAIT msg:,,
V/com.vpn.sdk(11995): state:AUTH msg:,,
V/com.vpn.sdk(11995): state:GET_CONFIG msg:,,
V/com.vpn.sdk(11995): state:ASSIGN_IP msg:,128.129.0.2,
V/com.vpn.sdk(11995): state:ADD_ROUTES msg:,,
V/com.vpn.sdk(11995): state:CONNECTED msg:SUCCESS,128.129.0.2,192.168.41.155
```

正常：
```
V/com.vpn.sdk(11995): state:WAIT msg:,,
V/com.vpn.sdk(11995): state:AUTH msg:,,
V/com.vpn.sdk(11995): state:GET_CONFIG msg:,,
V/com.vpn.sdk(11995): state:ASSIGN_IP msg:,128.129.0.2,
V/com.vpn.sdk(11995): state:ADD_ROUTES msg:,,
V/com.vpn.sdk(11995): state:CONNECTED msg
```
