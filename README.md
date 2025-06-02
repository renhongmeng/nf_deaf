Forked from [LGA1150/nf_deaf](https://github.com/LGA1150/nf_deaf)，感谢原作者

[English](./README_EN.md)

# 主要功能

一个在Linux路由器安装的内核模块，在TCP握手成功后抢先发出一个构造好的包，仅供网络测试使用，可以修改源码里的NF_DEAF_BUF_DEFAULT为你喜欢的内容，也可以在加载模块后修改/sys/kernel/debug/nf_deaf/buf的内容。


# 本Fork的修改

1. 删除TCP Option里的魔数0x1312，避免被类似这样的方法检测：tcpdump 'tcp\[20:2\] == 0x1312'，改为原封不动复制原数据包的TCP Option，同时避免了数据包缺失TimeStamp的问题。
2. 补全TCP头缺失的接收窗口。
3. 把mark左移三位，原来只能设置0-31的TTL，现在能设为正常的0-255。
4. 填好了测速的payload，不用自己改文件了。

# 使用方法
## 编译并使用
1. git clone
2. make
3. insmod nf_deaf.ko
4. 确认是否加载成功用lsmod |grep nf_deaf，移除模块用rmmod nf_deaf


## Mark定义
|  说明   | BIT  |
|  ----  | ----  |
| Magic 0xDEA |20-31|
| 使用错误的ACK SEQ |18|
| 使用错误的SEQ|17|
| 使用错误的TCP校验和|16|
| 重复次数|13-15|
| 延迟原包发送jiffies|8-12|
| 设置TTL，全0保持系统默认|0-7|

##  示例1：最大化减小特征，其他不变只使用错误的TCP校验和，mark为0xDEA10000，若需要使用低TTL（以2为例），校验和正常，只需设置mark为0xDEA00002

示例IP: 1.1.1.1
  
示例IPV6：2606:4700:4700::1111

注意：本示例会给每个符合条件的包都发送一次伪装包，会影响通信效率

1. nftables
```
nft insert rule inet filter postrouting ip daddr {1.1.1.1} tcp dport { 0-65535 } meta length gt 120 meta mark set 0xDEA10000 return comment "hahaha"
nft insert rule inet filter postrouting ip6 daddr {2606:4700:4700::1111} tcp dport { 0-65535 } meta length gt 100 meta mark set 0xDEA10000 return comment "hahaha"
```
2. iptables
```
iptables -t mangle -A POSTROUTING -d 1.1.1.1 -p tcp --dport 0:65535 -m length --length 121:65535 -j MARK --set-mark 0xDEA10000 -m comment --comment "hahaha"
ip6tables -t mangle -A POSTROUTING -d 2606:4700:4700::1111 -p tcp --dport 0:65535 -m length --length 101:65535 -j MARK --set-mark 0xDEA10000 -m comment --comment "hahaha"
```

## 示例2：TTL为3，错误的TCP校验和，只在握手成功后发一次伪装包，为防止原始包抢先发出设置jiffies为1，mark是0xDEA10103

1. nftables配置文件示例
```
#!/usr/sbin/nft -f

#清空nftables规则，小心这一条
flush ruleset

table inet filter {
    chain postrouting {
        type filter hook postrouting priority 0 ; policy accept;
        #被标记的连接不再打标记
        ct mark 0xDEA10103 return
        #IPv4，目的地址1.1.1.1，TCP端口0-65535，长度大于120的包，设置连接标记和包标记后发出
        ip daddr {1.1.1.1} tcp dport { 0-65535 } meta length gt 120 ct mark set 0xDEA10103 meta mark set 0xDEA10103 return
        #IPv6，目的地址2606:4700:4700::1111，TCP端口0-65535，长度大于100的包，设置连接标记和包标记后发出
        ip6 daddr {2606:4700:4700::1111} tcp dport { 0-65535 } meta length gt 100 ct mark set 0xDEA10103 meta mark set 0xDEA10103 return
    }

}
```

2. iptables

```
#IPv4被标记的连接不再打标记
iptables  -t mangle -A POSTROUTING -m connmark --mark 0xDEA10103 -j ACCEPT
#IPv6被标记的连接不再打标记
ip6tables  -t mangle -A POSTROUTING -m connmark --mark 0xDEA10103 -j ACCEPT
#IPv4，目的地址1.1.1.1，TCP端口0-65535，长度大于120的包，设置连接标记
iptables  -t mangle -A POSTROUTING -d 1.1.1.1 -p tcp --dport 0:65535 -m length --length 121:65535 -j CONNMARK --set-mark 0xDEA10103 
#IPv4，目的地址1.1.1.1，TCP端口0-65535，长度大于120的包，设置包标记
iptables  -t mangle -A POSTROUTING -d 1.1.1.1 -p tcp --dport 0:65535 -m length --length 121:65535 -j MARK --set-xmark 0xDEA10103/0xFFFFFFFF
#IPv4，目的地址1.1.1.1，TCP端口0-65535，长度大于120的包，打完标记发出
iptables  -t mangle -A POSTROUTING -d 1.1.1.1 -p tcp --dport 0:65535 -m length --length 121:65535 -j ACCEPT
#IPv6，目的地址2606:4700:4700::1111，TCP端口0-65535，长度大于100的包，设置连接标记
ip6tables -t mangle -A POSTROUTING -d 2606:4700:4700::1111 -p tcp --dport 0:65535 -m length --length 101:65535 -j CONNMARK --set-mark 0xDEA10103
#IPv6，目的地址2606:4700:4700::1111，TCP端口0-65535，长度大于100的包，设置包标记
ip6tables -t mangle -A POSTROUTING -d 2606:4700:4700::1111 -p tcp --dport 0:65535 -m length --length 101:65535 -j MARK --set-xmark 0xDEA10103/0xFFFFFFFF
#IPv6，目的地址2606:4700:4700::1111，TCP端口0-65535，长度大于100的包，打完标记发出
ip6tables -t mangle -A POSTROUTING -d 2606:4700:4700::1111 -p tcp --dport 0:65535 -m length --length 101:65535 -j ACCEPT
```

## 示例3：nftables，同示例2，但跳过私有地址，作用到其余所有地址

## 警告！未经测试，可能使你无法连接路由器，自担风险使用

```
#!/usr/sbin/nft -f

#清空nftables规则，小心这一条
flush ruleset

table inet filter {
    chain postrouting {
        type filter hook postrouting priority 0 ; policy accept;
        #跳过IPv4私有地址
        ip daddr {100.64.0.0/10, 0.0.0.0/8, 10.0.0.0/8, 127.0.0.0/8, 169.254.0.0/16, 172.16.0.0/12, 192.0.0.0/24, 192.0.2.0/24, 192.88.99.0/24, 192.168.0.0/16, 198.18.0.0/15, 198.51.100.0/24, 203.0.113.0/24, 224.0.0.0/4, 240.0.0.0/4} return
        #跳过IPv6私有地址
        ip6 daddr {::, ::1, ::ffff:0:0:0/96, 64:ff9b::/96, 100::/64, 2001::/32, 2001:20::/28, 2001:db8::/32, 2002::/16, fc00::/7, fe80::/10, ff00::/8} return
        #被标记的连接不再打标记
        ct mark 0xDEA10103 return
        #IPv4，其余所有地址，端口0-65535，长度大于120的包，设置连接标记和包标记
        ip protocol tcp tcp dport { 0-65535 } meta length gt 120 ct mark set 0xDEA10103 meta mark set 0xDEA10103 return
        #IPv6，其余所有地址，端口0-65535，长度大于100的包，设置连接标记和包标记
        ip6 nexthdr tcp tcp dport { 0-65535 } meta length gt 100 ct mark set 0xDEA10103 meta mark set 0xDEA10103 return

    }

}
```

