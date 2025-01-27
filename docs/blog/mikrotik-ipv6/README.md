# 解决了一次mikrotik ipv6连不上的问题

## 问题现象
前几天自己家的IPv6又可以拿到了，于是按照网上的攻略配置上了。但是隔了几天发现个奇怪的现象，可以ping通IPv6地址，可以curl http的网页，但就是curl不通https的网页

* http

``` bash
tianyf@docker:~$ curl test.ipw.cn -v
* Host test.ipw.cn:80 was resolved.
* IPv6: 240e:f7:4f00:2510:5e::16, 240e:f7:4d0f:301:64::c, 240e:f7:4f00:2510:5e::1c, 240e:f7:4f00:2510:5e::1d, 240e:f7:4f00:2510:5e::1b, 240e:f7:4d0f:301:64::1d, 240e:f7:4f00:2510:5e::1, 240e:f7:4d0f:301:66::22, 240e:f7:4f00:2510:5e::19
* IPv4: 183.134.93.98, 183.134.93.121, 183.134.39.55, 183.131.41.62, 115.231.145.183, 183.134.39.69, 183.134.39.57, 60.188.68.175, 183.134.39.62, 183.134.39.71, 183.131.59.186, 183.134.93.156, 183.134.39.72, 183.131.191.207, 183.134.39.61
*   Trying [240e:f7:4f00:2510:5e::16]:80...
* Connected to test.ipw.cn (240e:f7:4f00:2510:5e::16) port 80
> GET / HTTP/1.1
> Host: test.ipw.cn
> User-Agent: curl/8.5.0
> Accept: */*
> 
< HTTP/1.1 200 OK
< Access-Control-Allow-Origin: https://ipw.cn
< Content-Type: application/json;charset=UTF-8
< Date: Mon, 27 Jan 2025 06:25:31 GMT
< Server: SLT-MID
< X-Cache-Lookup: Cache Miss
< Content-Length: 37
< X-NWS-LOG-UUID: 18412974302972723441
< Connection: keep-alive
< X-Cache-Lookup: Cache Miss
< 
* Connection #0 to host test.ipw.cn left intact
240e::xxxx
```

* https

``` bash
tianyf@docker:~$ curl https://test.ipw.cn -v
* Host test.ipw.cn:443 was resolved.
* IPv6: 240e:f7:4d0f:301:64::c, 240e:f7:4f00:2510:5e::1c, 240e:f7:4f00:2510:5e::1d, 240e:f7:4f00:2510:5e::1b, 240e:f7:4d0f:301:64::1d, 240e:f7:4f00:2510:5e::1, 240e:f7:4d0f:301:66::22, 240e:f7:4f00:2510:5e::19, 240e:f7:4f00:2510:5e::16
* IPv4: 183.134.39.69, 183.134.39.57, 60.188.68.175, 183.134.39.62, 183.134.39.71, 183.131.59.186, 183.134.93.156, 183.134.39.72, 183.131.191.207, 183.134.39.61, 183.134.93.98, 183.134.93.121, 183.134.39.55, 183.131.41.62, 115.231.145.183
*   Trying [240e:f7:4d0f:301:64::c]:443...
* Connected to test.ipw.cn (240e:f7:4d0f:301:64::c) port 443
* ALPN: curl offers h2,http/1.1
* TLSv1.3 (OUT), TLS handshake, Client hello (1):
*  CAfile: /etc/ssl/certs/ca-certificates.crt
*  CApath: /etc/ssl/certs
^C
```

这就很离谱

## 排查思路
### dns

```
Host test.ipw.cn:443 was resolved.
* IPv6: 240e:f7:4d0f:301:64::c, 240e:f7:4f00:2510:5e::1c, 240e:f7:4f00:2510:5e::1d, 240e:f7:4f00:2510:5e::1b, 240e:f7:4d0f:301:64::1d, 240e:f7:4f00:2510:5e::1, 240e:f7:4d0f:301:66::22, 240e:f7:4f00:2510:5e::19, 240e:f7:4f00:2510:5e::16
* IPv4: 183.134.39.69, 183.134.39.57, 60.188.68.175, 183.134.39.62, 183.134.39.71, 183.131.59.186, 183.134.93.156, 183.134.39.72, 183.131.191.207, 183.134.39.61, 183.134.93.98, 183.134.93.121, 183.134.39.55, 183.131.41.62, 115.231.145.183
*   Trying [240e:f7:4d0f:301:64::c]:443...
```
既然curl都解析出来了IPv6的地址了，那DNS应该是没问题（暂时不管DNS污染什么的）

### TCP连接建立

```
*   Trying [240e:f7:4d0f:301:64::c]:443...
* Connected to test.ipw.cn (240e:f7:4d0f:301:64::c) port 443
```
已经提示`Connected to test.ipw.cn (240e:f7:4d0f:301:64::c) port 443`，证明TCP三次握手已经通了

### TLS握手
```
* TLSv1.3 (OUT), TLS handshake, Client hello (1):
*  CAfile: /etc/ssl/certs/ca-certificates.crt
*  CApath: /etc/ssl/certs
```
这里客户端虽然发了hello包，但是始终没有收到服务端的hello包，


用IPV6测试网站发现出现了大包丢失的情况（MTU），于是根据关键字mikrotik、mtu查到可能需要修改mss来解决问题

## 解决方法
在IPV6防火墙里添加一条mangle，就是每次建立tcp连接都去钳制（clamp）mtu，防止被都到pmtu黑洞
```shell
/ipv6 firewall mangle
add action=change-mss chain=forward new-mss=clamp-to-pmtu passthrough=yes protocol=tcp tcp-flags=syn
```

加上这条规则以后，https访问直接就通了
```bash
tianyf@docker:~$ curl https://test.ipw.cn -v
* Host test.ipw.cn:443 was resolved.
* IPv6: 240e:f7:4f00:2510:5e::1d, 240e:f7:4d0f:301:64::c, 240e:f7:4f00:2510:5e::19, 240e:f7:4d0f:301:64::1d, 240e:f7:4f00:2510:5e::1b, 240e:f7:4f00:2510:5e::16, 240e:f7:4d0f:301:66::22, 240e:f7:4f00:2510:5e::1, 240e:f7:4f00:2510:5e::1c
* IPv4: 183.131.41.62, 183.134.39.55, 183.131.59.186, 115.231.145.183, 183.134.39.71, 183.134.39.62, 183.134.93.121, 183.134.93.98, 183.131.191.207, 183.134.39.72, 183.134.39.57, 183.134.39.61, 60.188.68.175, 183.134.39.69, 183.134.93.156
*   Trying [240e:f7:4f00:2510:5e::1d]:443...
* Connected to test.ipw.cn (240e:f7:4f00:2510:5e::1d) port 443
* ALPN: curl offers h2,http/1.1
* TLSv1.3 (OUT), TLS handshake, Client hello (1):
*  CAfile: /etc/ssl/certs/ca-certificates.crt
*  CApath: /etc/ssl/certs
* TLSv1.3 (IN), TLS handshake, Server hello (2):
* TLSv1.2 (IN), TLS handshake, Certificate (11):
* TLSv1.2 (IN), TLS handshake, Server key exchange (12):
* TLSv1.2 (IN), TLS handshake, Server finished (14):
* TLSv1.2 (OUT), TLS handshake, Client key exchange (16):
* TLSv1.2 (OUT), TLS change cipher, Change cipher spec (1):
* TLSv1.2 (OUT), TLS handshake, Finished (20):
* TLSv1.2 (IN), TLS handshake, Finished (20):
* SSL connection using TLSv1.2 / ECDHE-RSA-AES128-GCM-SHA256 / X25519 / RSASSA-PSS
* ALPN: server accepted h2
* Server certificate:
*  subject: CN=test.ipw.cn
*  start date: Apr 21 00:00:00 2024 GMT
*  expire date: Apr 21 23:59:59 2025 GMT
*  subjectAltName: host "test.ipw.cn" matched cert's "test.ipw.cn"
*  issuer: C=CN; O=TrustAsia Technologies, Inc.; CN=TrustAsia RSA DV TLS CA G2
*  SSL certificate verify ok.
*   Certificate level 0: Public key type RSA (2048/112 Bits/secBits), signed using sha384WithRSAEncryption
*   Certificate level 1: Public key type RSA (3072/128 Bits/secBits), signed using sha256WithRSAEncryption
*   Certificate level 2: Public key type RSA (2048/112 Bits/secBits), signed using sha1WithRSAEncryption
* using HTTP/2
* [HTTP/2] [1] OPENED stream for https://test.ipw.cn/
* [HTTP/2] [1] [:method: GET]
* [HTTP/2] [1] [:scheme: https]
* [HTTP/2] [1] [:authority: test.ipw.cn]
* [HTTP/2] [1] [:path: /]
* [HTTP/2] [1] [user-agent: curl/8.5.0]
* [HTTP/2] [1] [accept: */*]
> GET / HTTP/2
> Host: test.ipw.cn
> User-Agent: curl/8.5.0
> Accept: */*
> 
< HTTP/2 200 
< access-control-allow-origin: https://ipw.cn
< content-type: application/json;charset=UTF-8
< date: Mon, 27 Jan 2025 07:19:38 GMT
< server: SLT-MID
< x-cache-lookup: Cache Miss
< content-length: 37
< x-nws-log-uuid: 13837217515456469921
< x-cache-lookup: Cache Miss
< 
* Connection #0 to host test.ipw.cn left intact
240e:xxxx
```

## 参考文献
* [开启 IPv6 后网速变得很慢？可能是 PMTU 黑洞的问题](https://www.v2ex.com/t/800024)
* [ROS修改MTU和MSS解决上网慢和页面显示不全问题](https://zhuanlan.zhihu.com/p/117365627)s

### 部分节选
> 发现最近经常有人提到开启 IPv6 连接速度慢的问题。目前国内确实存在支持 IPv6 的服务器、CDN 节点不够多，IPv6 国际带宽比 IPv4 带宽小的问题，但也不至于会打开国内网站都卡。通常情况下遇到这个问题说明你到目标服务器的链路上存在 PMTU 黑洞。
>
> # 关于 PMTU 黑洞
>
> MTU (Maximum transmission unit) 是一条链路上可以通过的三层数据包的最大尺寸（包含 IP 包头）。以太网上默认的 MTU 是 1500 字节，但是你和目标服务器之间的路径上可能存在小于 MTU 1500 的链路。这条路径上最小的 MTU 值就是整条路径的 PMTU 值。路由器在转发包时，超过 MTU 大小的包会被分片（ Fragmentation ），也就是一个大包会被分切为多个不超过 MTU 的小包进行传输，传输效率会下降。
>
> 终端设备在发包时，也可以设置 DF （ Don't Fragment ）标记来告诉路由器不要分片。这时中间路由器会丢掉超过 MTU 的包，回复一条 ICMP Fragmentation Needed 消息。发送者收到这个包后，下次就会发小一点的包，这个过程叫做 PMTU Discovery 。现实中可以看到 HTTPS （ TLS ）的流量大都是带 DF 标记的。
>
> 然而，互联网上有大量的中间设备为了所谓的“安全”或者没有正确配置，不回应 ICMP Fragmentation Needed 包，这使得访问某些网站时如果某个包的大小超过了 PMTU，会被无声地丢弃，直到 TCP 协议发现超时丢包进行重传，这非常缓慢。遇到这种情况，我们可以说你和目标服务器的路径上存在 PMTU 黑洞。
> 
> 此外，IPv6 不支持分片，换句话说可以理解为 IPv6 下所有的包都是带 DF 标记的。中间路由器在遇到包尺寸大于 MTU 的情况时，应该回应 ICMPv6 Packet Too Big 消息。同样的，由于种种原因，某些中间设备可能会直接丢包而不回应 ICMPv6 Packet Too Big 消息，直到 TCP 协议发现超时丢包进行重传。。。
> 
> # 为什么 IPv4 没有这个问题
> 其实 IPv4 也有这个问题，我不只一次见网友说自己搭的软路由访问某些网站非常慢，而换回硬路由就正常。这是因为多数家用路由器默认对 IPv4 下的 TCP 开启了 MSS (maximum segment size) Clamping （使用 OpenWRT 软路由的朋友们可以在防火墙设置中找到 MSS Clamping 开关）。MSS Clamping 是针对 PMTU 黑洞的 Workaround，简单来说就是 TCP 握手时有个 MSS 字段决定单个 TCP 包的最大尺寸。路由器可以通过嗅探 TCP 握手包，把 MSS 值改小，使最终的三层 IP 包的尺寸（ MSS+TCP 头大小+IP 头大小）不超过某个特定的值。
> 
> # 总结
> 现在国内 ISP 一般都是通过 PPPoE 虚拟拨号建立 WAN 口连接的。Ethernet 的默认 MTU 是 1500，但是 PPPoE 隧道有 8 个 bytes 的开销，所以 PPPoE 虚连接的 MTU 就是 1500-8=1492，减掉 IPv4 包头（ 20 字节）和 TCP 包头（ 20 字节），可以得知 IPv4 下需要把 MSS 设为 1452 以下。
> 
> IPv6 的包头是 40 字节，所以 IPv6 下需要把 MSS 设为 1432 以下。
> 
> 这时问题来了，目前很多光猫、家用路由器对 IPv6 的优化很差，不支持对 IPv6 下的 TCP 包进行 MSS Clamping，这就导致访问 IPv6 网站时，若路径中存在 PMTU 黑洞，则打开很慢。
> 
> 我前段时间帮朋友配置 IPv6 时发现了很多光猫、家用路由器的固件问题，使得国内使用 IPv6 的体验不太理想。我打算抽空专门开一个帖子去讨论这些问题，声讨那些垃圾厂家。目前来看，要想在国内比较理想地体验 IPv6，你需要把光猫改为桥接模式，并使用 OpenWRT 或者 VyOS 这类对 IPv6 支持较好的软路由。

