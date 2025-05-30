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

##  示例：最大化减小特征，其他不变只使用错误的TCP校验和，mark为0xDEA10000

示例IP: 1.1.1.1
  
示例IPV6：2606:4700:4700::1111

1. nftables
```
nft insert rule inet filter postrouting ip daddr {1.1.1.1} tcp dport { 0-65535 } meta length gt 120 meta mark set 0xDEA10000 return comment "hahaha"
nft insert rule inet filter postrouting ip6 daddr {2606:4700:4700::1111} tcp dport { 0-65535 } meta length gt 100 meta mark set 0xDEA10000 return comment "hahaha"```
```
2. iptables
```
iptables -t mangle -A POSTROUTING -d 1.1.1.1 -p tcp --dport 0:65535 -m length --length 121:65535 -j MARK --set-mark 0xDEA10000 -m comment --comment "hahaha"
ip6tables -t mangle -A POSTROUTING -d 2606:4700:4700::1111 -p tcp --dport 0:65535 -m length --length 101:65535 -j MARK --set-mark 0xDEA10000 -m comment --comment "hahaha"
```

若需要使用低TTL（以2为例），校验和正常，只需设置mark为0xDEA00002




 



