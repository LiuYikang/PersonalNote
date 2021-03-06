# tcpdump

## 常用参数
* -i interface: 设置抓取的网卡名（可以使用 -i any 抓取所有网卡的数据包）
```
tcpdump -i eth0
```

* -D: 列出可用的网卡列表
```
$ sudo tcpdump -D
1.eth0
2.nflog (Linux netfilter log (NFLOG) interface)
3.nfqueue (Linux netfilter queue (NFQUEUE) interface)
4.usbmon1 (USB bus number 1)
5.any (Pseudo-device that captures on all interfaces)
6.lo
```

* -w file: 把捕获的包数据写入到文件中（可以使用 -w - 输出到标准输出）
```
tcpdump -i eth0 -w debug.cap
```

* -C size: 使用 -w 写入文件时，限制文件的最大大小，超出时新开一个文件（单位是 1,000,000 bytes）
```
$ sudo tcpdump -i eth0 -w debug.cap -C 1
$ ls debug* -l
-rw-r--r-- 1 tcpdump tcpdump 1000956 Jan 14 10:16 debug.cap
-rw-r--r-- 1 tcpdump tcpdump 1000323 Jan 14 10:32 debug.cap1
-rw-r--r-- 1 tcpdump tcpdump 1000017 Jan 14 10:51 debug.cap2
-rw-r--r-- 1 tcpdump tcpdump  970705 Jan 14 11:08 debug.cap3
-r file: 从文件中读取包数据

tcpdump -r debug.cap
```

* -v: 启用 verbose output，抓包时输出包的附加信息（可以使用多个 -v: -v, -vv, -vvv 多个 v 会显示更多更详细的信息）
```
tcpdump -v
tcpdump -vv
```
* -A: 以 ASCII 码方式显示每一个数据包(不会显示数据包中链路层头部信息). 在抓取包含网页数据的数据包时, 可方便查看数据

* -x: 打印每个包的头部数据, 同时会以16进制打印出每个包的数据(但不包括连接层的头部)

* -xx: 打印每个包的头部数据, 同时会以16进制打印出每个包的数据, 其中包括数据链路层的头部

* -X: 打印每个包的头部数据, 同时会以16进制和 ASCII 码形式打印出每个包的数据(但不包括连接层的头部)

* -XX: 打印每个包的头部数据, 同时会以16进制和 ASCII 码形式打印出每个包的数据, 其中包括数据链路层的头部

* -c count: 设置抓取到多少个包后就退出
```
tcpdump -i eth0 -c 100
```

* -n: 不要把地址转换为主机名（直接显示 ip 不要解析为域名）
```
tcpdump -n
```

* -nn: 不要把转换协议和端口号（直接显示协议和端口号，不要转换为协议名称，比如 http）
```
tcpdump -nn
```

* -s snaplen: 设置 tcpdump 的数据包抓取长度为 snaplen , 为 0 时表示让 tcpdump 自动选择合适的长度来抓取数据包.
```
tcpump -s 0
```

* -S: 打印TCP 数据包的顺序号时, 使用绝对的顺序号, 而不是相对的顺序号.
```
tcpdump -S
```

* -Z user: 使tcpdump 放弃自己的超级权限(如果以root用户启动tcpdump, tcpdump将会有超级用户权限), 并把当前tcpdump的用户ID设置为user, 组ID设置为user首要所属组的ID
```
sudo tcpdump -Z user2
```


## 常用的参数组合

* sudo tcpdump -i eth0 -nnS -s 0 -c 100 -Avvv [<expression>]
* sudo tcpdump -i eth0 -nnS -s 1024 -c 100 -Avvv [<expression>]
* sudo tcpdump -i eth0 -nnS -s 1024 -C 10 -c 10000 -v -w debug.cap [<expression>]
* tcpdump抓取tcp三次握手：tcpdump 'port $PORT' -i eth0 -S
> 默认 tcpdump 会抓取所有的数据，可以通过指定过滤规则来过滤数据包。

## 常用过滤规则
**过滤规则一般包含三种修饰符的组合：**
1. type: 指定id 所代表的对象类型, id可以是名字也可以是数字. 可选的对象类型有: host, net, port 以及portrange，默认是 host
2. dir: 描述id 所对应的传输方向, 即发往id 还是从id 接收（而id 到底指什么需要看其前面的type 修饰符）.可取的方向为: src, dst, src or dst, src and dst
3. proto: 描述id 所属的协议. 可选的协议有: ether, fddi, tr, wlan, ip, ip6, arp, rarp, decnet, tcp以及 upd

**通过括号(\( xxx \)) 和 bool 操作符可以组合多种过滤规则，一对括号是一组:**
1. 否定操作: ! 或 not
2. 与操作: && 或 and
3. 或操作: || 或 or

#### 下面列出一下常用的过滤规则:

**过滤 ip/域名**
* 过滤目标域名是 baidu.com:
```
dst host baidu.com
```

* 源 ip 或者目标 ip 是 192.168.1.3:
```
host 192.168.1.3
```

* 源 ip 是 192.168.1.3:
```
src host 192.168.1.3
```

* 目标 ip 是 192.168.1.3:
```
dst host 192.168.1.3
```

* 过滤范围内的 ip /网段 ip:
```
net 192.168.0.0/24
net 192.168.0.0 mask 255.255.255.0
```

**过滤端口**
* 过滤 80 端口:
```
port 80
```

* 排除端口:
```
not port 80
host www.example.com and not \(port 80 or port 25\)
host www.example.com and not port 80 and not port 25
```

* 端口范围:
```
(tcp[0:2] > 1500 and tcp[0:2] < 1550) or (tcp[2:2] > 1500 and tcp[2:2] < 1550)

tcp portrange 1501-1549
```

* DNS 53 端口:
```
port 53
```

**过滤协议**
* ipv4: ip
* ipv6: ip6
* tcp: tcp
* udp: udp
* arp: arp
* icmp: icmp

**过滤协议头**
* 过滤 tcp SYN 消息包:
```
'tcp[tcpflags] & (tcp-syn) != 0'
```

* 过滤 tcp SYN/ACK 消息包:
```
'tcp[tcpflags] & (tcp-syn|tcp-ack) != 0'
```

* 常用的 tcp 标记:
```
tcp-fin, tcp-syn, tcp-rst, tcp-push, tcp-ack, tcp-urg, tcp-ece, tcp-cwr
```

* 源端口大于1024的TCP数据包:
```
'tcp[0:2] > 1024'
```
> 注意要用引号引起来。

**过滤 HTTP**
* 过滤 GET 请求:
```
'tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x47455420'
```
上面的 0x47455420 是 'GET ' 的 16 进制:
```python
>>> list(map(hex, [ord(x) for x in 'GET ']))
['0x47', '0x45', '0x54', '0x20']
```

* 过滤 POST 请求:
```
'tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x504f5354 && tcp[((tcp[12:1] & 0xf0) >> 2) + 4:1] = 0x20'

>>> list(map(hex, [ord(x) for x in 'POST ']))
['0x50', '0x4f', '0x53', '0x54', '0x20']
```

* 过滤 PUT 请求:
```
'tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x50555420'

>>> list(map(hex, [ord(x) for x in 'PUT ']))
['0x50', '0x55', '0x54', '0x20']
```

* 过滤 PATH 请求:
```
'tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x50415443 && tcp[((tcp[12:1] & 0xf0) >> 2) + 4:2] = 0x4820'

>>> list(map(hex, [ord(x) for x in 'PATCH ']))
['0x50', '0x41', '0x54', '0x43', '0x48', '0x20']
```
> 之所以拆分成了多个条件是因为 tcpdump: data size must be 1, 2, or 4

* 过滤 DELETE 请求:
```
'tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x44454c45 && tcp[((tcp[12:1] & 0xf0) >> 2) + 4:2] = 0x5445 && tcp[((tcp[12:1] & 0xf0) >> 2) + 6:1] = 0x20'

>>> list(map(hex, [ord(x) for x in 'DELETE ']))
['0x44', '0x45', '0x4c', '0x45', '0x54', '0x45', '0x20']
```

* 过滤 HEAD 请求:
```
'tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x48454144 && tcp[((tcp[12:1] & 0xf0) >> 2) + 4:1] = 0x20'

>>> list(map(hex, [ord(x) for x in 'HEAD ']))
['0x48', '0x45', '0x41', '0x44', '0x20']
```

* 过滤 OPTIONS 请求:
```
'tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x4f505449 && tcp[((tcp[12:1] & 0xf0) >> 2) + 4:4] = 0x4f4e5320'

>>> list(map(hex, [ord(x) for x in 'OPTIONS ']))
['0x4f', '0x50', '0x54', '0x49', '0x4f', '0x4e', '0x53', '0x20']
```

* 过滤 HTTP 响应 (HTTP/1.):
```
'tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x48545450 && tcp[((tcp[12:1] & 0xf0) >> 2) + 4:2] = 0x2f31 && tcp[((tcp[12:1] & 0xf0) >> 2) + 6:1] = 0x2e'
```

上面的规则可以通过括号和操作符进行各种组合，从而组合出复杂的过滤规则。
```
tcpdump host 192.168.8.3 -Avv
tcpdump dst host baidu.com and dst port 80 -i eth0 -vv
tcpdump dst host baidu.com and not dst port 80 -i eth0 -vv
tcpdump dst host baidu.com and not \(dst port 80 or dst port 443\) -i en0 -vv
tcpdump dst host baidu.com and 'tcp[tcpflags] & (tcp-syn) != 0'
```
