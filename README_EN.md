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

##  Example: Minimize side effects, only corrupt the TCP checksum, mark = 0xDEA10000. If you need to use a low TTL (e.g., 2) with a correct checksum, set the mark to 0xDEA00002.

IPv4 example: 1.1.1.1

IPv6 example: 2606:4700:4700::1111

Note: This example sends a spoofed packet for every matching packet, which may impact communication efficiency.

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


## Example 2: TTL set to 3, incorrect TCP checksum, spoofed packet sent only once after handshake. To prevent the original packet from being sent prematurely, set jiffies to 1. mark is 0xDEA10103

1. Nftables Configuration Example
```
#!/usr/sbin/nft -f

# Flush all nftables rules (be careful with this line)
flush ruleset

table inet filter {
    chain postrouting {
        type filter hook postrouting priority 0 ; policy accept;
        # Connections already marked will not be marked again
        ct mark 0xDEA10103 return
        # For IPv4: Destination 1.1.1.1, TCP port range 0-65535, packet length > 120,
        # set connection mark and packet mark, then send
        ip daddr {1.1.1.1} tcp dport { 0-65535 } meta length gt 120 ct mark set 0xDEA10103 meta mark set 0xDEA10103 return
        # For IPv6: Destination 2606:4700:4700::1111, TCP port range 0-65535, packet length > 100,
        # set connection mark and packet mark, then send
        ip6 daddr {2606:4700:4700::1111} tcp dport { 0-65535 } meta length gt 100 ct mark set 0xDEA10103 meta mark set 0xDEA10103 return
    }

}
```

2. iptables

```
# For IPv4: Skip marking for connections already marked
iptables  -t mangle -A POSTROUTING -m connmark --mark 0xDEA10103 -j ACCEPT
# For IPv6: Skip marking for connections already marked
ip6tables  -t mangle -A POSTROUTING -m connmark --mark 0xDEA10103 -j ACCEPT
# For IPv4: Destination 1.1.1.1, TCP port range 0-65535, packet length > 120, set connection mark
iptables  -t mangle -A POSTROUTING -d 1.1.1.1 -p tcp --dport 0:65535 -m length --length 121:65535 -j CONNMARK --set-mark 0xDEA10103 
# For IPv4: Set packet mark
iptables  -t mangle -A POSTROUTING -d 1.1.1.1 -p tcp --dport 0:65535 -m length --length 121:65535 -j MARK --set-xmark 0xDEA10103/0xFFFFFFFF
# For IPv4: Accept the marked packet
iptables  -t mangle -A POSTROUTING -d 1.1.1.1 -p tcp --dport 0:65535 -m length --length 121:65535 -j ACCEPT
# For IPv6: Destination 2606:4700:4700::1111, TCP port range 0-65535, packet length > 100, set connection mark
ip6tables -t mangle -A POSTROUTING -d 2606:4700:4700::1111 -p tcp --dport 0:65535 -m length --length 101:65535 -j CONNMARK --set-mark 0xDEA10103
# For IPv6: Set packet mark
ip6tables -t mangle -A POSTROUTING -d 2606:4700:4700::1111 -p tcp --dport 0:65535 -m length --length 101:65535 -j MARK --set-xmark 0xDEA10103/0xFFFFFFFF
# For IPv6: Accept the marked packet
ip6tables -t mangle -A POSTROUTING -d 2606:4700:4700::1111 -p tcp --dport 0:65535 -m length --length 101:65535 -j ACCEPT
```









