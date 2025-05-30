Forked from [LGA1150/nf_deaf](https://github.com/LGA1150/nf_deaf), thanks to the original author.

[Chinese](./README.md)

# Main Features

This is a Linux kernel module installed on a router that, immediately after a successful TCP handshake, sends out a pre-constructed packet for network testing purposes. You can modify the default payload by changing NF_DEAF_BUF_DEFAULT in the source code, or update the content at /sys/kernel/debug/nf_deaf/buf after loading the module.


# Changes in This Fork

1. Removed the magic number 0x1312 from the TCP Option, preventing detection via filters like: tcpdump 'tcp\[20:2\] == 0x1312'. Instead, TCP Options are now copied verbatim from the original packet, avoiding issues with missing Timestamps.
2. Restored the missing receive window field in the TCP header.
3. Shifted the mark value left by 3 bits, allowing TTL values from 0 to 255 instead of only 0–31.
4. Included a ready-to-use speedtest payload—no need to modify the file yourself.

# Usage
## Build and Load
1. git clone
2. make
3. insmod nf_deaf.ko
4. Verify with lsmod | grep nf_deaf; remove with rmmod nf_deaf


## Definitions
|  Description   | BIT  |
|  ----  | ----  |
| Magic 0xDEA |20-31|
| Corrupt ACK sequence number |18|
| Corrupt sequence number |17|
| Corrupt TCP checksum |16|
| Repeat count |13-15|
| Delay original packet (in jiffies) |8-12|
| Set TTL (0 for system default) |0-7|

##  Example: Minimize side effects, only corrupt the TCP checksum, mark = 0xDEA10000

IPv4 example: 1.1.1.1

IPv6 example: 2606:4700:4700::1111

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


If you need to use a low TTL (e.g., 2) with a correct checksum, set the mark to 0xDEA00002.








