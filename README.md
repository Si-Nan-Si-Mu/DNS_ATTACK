# DNS 劫持演示靶场

网络安全课程演示材料。在**完全隔离**的环境里，复现完整的 DNS 劫持攻击链：

> 受害者访问 **www.baidu.com** → ARP 欺骗 + DNS 抢答 → 浏览器落到
> **仿百度钓鱼页**（页脚为 TANEI·计算机社团 / www.tanei.top）

提供 **Linux 版**（network namespace 完全隔离，**首选**）和 **Windows 版**
（`local` 本地安全 / `lan` 需授权）两套材料。

本仓库用于存放与分发实验资料。

---

## 仓库内容

```
DNS_ATTACK/
├── README.md                 # 本文件（总览与快速上手）
├── dns-lab/
│   └── README.md             # Linux 版完整文档（讲解词 / 原理 / 防御）
├── dns-lab.tar.gz            # 完整实验包（解压即用）
├── install-dns-lab.sh        # 单文件自解压安装脚本（可选）
└── （解压后）dns-lab/        # lab.sh、Python 脚本、WINDOWS-使用说明.md …
```

**包内主要文件**

| 文件 | 说明 |
|------|------|
| `lab.sh` | Linux 靶场主控（搭建 / 攻击 / 恢复 / 清理） |
| `dns-spoof.py` | DNS 抢答器（scapy） |
| `fake-baidu.py` | 仿百度钓鱼页（HTTP + HTTPS 自签） |
| `webserver.py` | 通用极简 Web 服务（备用） |
| `dns-hijack-win.py` | Windows 版（`local` / `lan`） |
| `WINDOWS-使用说明.md` | Windows 用法、Npcap、端口冲突 |
| `README.md` | Linux 逐步讲解与防御视角 |

更细的课堂讲稿与原理见包内 / 本仓库 `dns-lab/README.md`。

---

## 环境要求

**Linux（首选）**：root + `iproute2`、`dnsmasq`、`dsniff`、`openssl`、`dnsutils`、`curl`、`python3-scapy`。

```bash
sudo apt update && sudo apt install -y dnsmasq dsniff openssl curl dnsutils python3-scapy
```

**Windows**：Python 3 即可跑 `local` 模式（标准库，零额外依赖）。  
`lan` 模式才需要 Npcap + `pip install scapy`（校园网一般**不要**用 `lan`）。

---

## 获取材料

**方式 A：克隆本仓库**

```bash
git clone git@github.com:Si-Nan-Si-Mu/DNS_ATTACK.git
cd DNS_ATTACK
tar xzf dns-lab.tar.gz
cd dns-lab
```

**方式 B：只要压缩包**

下载本仓库中的 `dns-lab.tar.gz`，解压后进入 `dns-lab/`。

**方式 C：单文件安装脚本**

```bash
bash install-dns-lab.sh          # 安装到当前目录下的 dns-lab/
cd dns-lab
```

> **提示**：解压后目录应是**一层** `dns-lab/`（里面直接是 `lab.sh`）。  
> 不要解压到已经叫 `dns-lab` 的文件夹里再套一层。

---

## 快速演示（Linux 命令行）

```bash
sudo ./lab.sh deps           # 检查依赖
sudo ./lab.sh up             # ① 搭建靶场
sudo ./lab.sh check          # ② 劫持前：解析到真实百度 IP
sudo ./lab.sh attack         # ③ ARP 欺骗 + DNS 抢答
sudo ./lab.sh dig            # ④ 再看解析 → 攻击者 10.66.0.10
sudo ./lab.sh curl           # ⑤ 访问 → 仿百度钓鱼页
sudo ./lab.sh attack-stop    # ⑥ 停止攻击
sudo ./lab.sh down           # ⑦ 全部清理（务必执行）
```

课堂高光：

- `sudo ./lab.sh status` —— 同一 `10.66.0.1`，攻击前后 MAC 不同  
- `sudo ./lab.sh logs` —— 攻击者视角看到受害者请求 `Host: www.baidu.com`

### 桌面浏览器版（更直观）

```bash
sudo ./lab.sh up
sudo ./lab.sh nat-on         # 临时改宿主机转发（演示完必须 nat-off）
sudo ./lab.sh browser        # 受害者节点里的 Firefox
# 先确认真实百度 → sudo ./lab.sh attack → 地址栏输入 http://www.baidu.com
sudo ./lab.sh browser-kill
sudo ./lab.sh nat-off
sudo ./lab.sh down
```

---

## Windows 快速演示（推荐 local）

管理员 PowerShell：

```powershell
cd dns-lab
python dns-hijack-win.py local

# 另开管理员窗口：把「以太网」换成实际网卡名（WLAN / Wi-Fi 等）
netsh interface show interface
netsh interface ip set dns "以太网" static 127.0.0.1
ipconfig /flushdns
# 浏览器打开 http://www.baidu.com/  → 钓鱼页

# 演示完务必恢复
netsh interface ip set dns "以太网" dhcp
ipconfig /flushdns
```

**不要**在未授权的校园网/公司网跑 `lan` 模式。

---

## 容易踩的坑（请先看）

1. **地址栏用 `http://`，不要默认 `https://`**  
   假站是自签证书；`https://` 会弹出「连接不是私密连接」。这不是 bug，正好用来讲 HTTPS 对纯 DNS 劫持的缓解，以及为何钓鱼仍有效。

2. **记得清 DNS 缓存**  
   Windows：`ipconfig /flushdns`。Linux 浏览器演示用 `lab.sh browser`（已关 Firefox DNS 缓存）；仍异常就再执行一次 `browser`。

3. **`REAL_IP` 会变（百度 CDN）**  
   `lab.sh` 顶部 `REAL_IP` 需与当前网络解析一致。演示前：

   ```bash
   dig +short www.baidu.com
   # 把 A 记录更新进 lab.sh 的 REAL_IP=
   ```

4. **解压路径不要套娃**  
   正确：`…/dns-lab/lab.sh`。错误：`…/dns-lab/dns-lab/lab.sh`。

5. **`nat-on` 之后一定要 `nat-off` + `down`**  
   否则宿主机可能残留 `ip_forward` / MASQUERADE / 网桥地址。

6. **Windows 端口占用**  
   - 53：常见被 ICS（共享网络）占用 → `net stop SharedAccess` 或换管理员重试  
   - 80：IIS/Skype 等 → 先腾出 80，或 `--web-port 8080`（浏览器需带端口，演示不便）

7. **Npcap 安装被 `8021x.exe` 占用（校园网常见）**  
   仅 `lan` 模式需要 Npcap。可先停服务再装：

   ```powershell
   Stop-Service dot3svc, EapHost -Force -ErrorAction SilentlyContinue
   Stop-Process -Name 8021x -Force -ErrorAction SilentlyContinue
   # 再运行 Npcap 安装（勾选 WinPcap API-compatible Mode）
   # 装完：Start-Service EapHost, dot3svc
   ```

   课堂只做 `local` 时可**跳过 Npcap**。

8. **Linux 工作目录放本地盘**  
   源码若在 sshfs/网络盘上，部分虚拟网卡/镜像操作可能失败；请放到本机磁盘。

---

## 验证说明（资料维护）

| 项目 | 状态 |
|------|------|
| Linux netns 全流程（文档所述） | 作者环境实测通过 |
| Windows `local`：绑 53/80、劫持 `www.baidu.com`→`127.0.0.1`、上游转发、钓鱼页 | **已在 Windows 上实测** |
| Windows `netsh` 改系统 DNS + 浏览器实操 | 需本机管理员自行点一次确认 |
| Windows `lan` + Npcap | 未作为默认推荐路径；需授权网络 |

演示结束后检查：无 `dns-hijack` 进程、系统 DNS **不是** `127.0.0.1`、Linux 侧已 `lab.sh down`。

---

## 安全边界（重要）

- **Linux**：流量封在 `10.66.0.0/24` netns 内；`down` 后应零残留。  
- **Windows `local`**：只影响本机；改完 DNS 务必改回 DHCP。  
- **Windows `lan`**：对真实二层网络的攻击，仅限自有/已授权环境。

建议：学生各自跑 Linux 靶场或 Windows `local`；教师投屏演示即可。

---

## 许可证与用途

仅供教学与授权范围内的安全演示。禁止用于未授权网络或任何违法用途。
