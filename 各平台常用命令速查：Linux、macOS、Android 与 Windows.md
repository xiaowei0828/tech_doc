# 各平台常用命令速查：Linux、macOS、Android 与 Windows

按平台整理开发与排障常用命令，覆盖日志、进程、网络、文件查找和文本处理。每类命令配合用途、常用参数及示例说明；Windows 章节暂留空，后续按需扩充。

## 目录

1. [使用约定](#1-使用约定)
2. [Linux 与 macOS](#2-linux-与-macos)
3. [Android](#3-android)
4. [常用排查流程](#4-常用排查流程)
5. [参考资料](#5-参考资料)
6. [Windows](#6-windows)

Linux/macOS 文本处理索引：[grep](#210-grep按内容筛选行)、[find](#211-find按文件名与属性查找)、[awk](#212-awk按字段处理与统计)、[sed](#213-sed筛选与替换文本)、[tail / head / less](#214-tailheadless查看文件与跟踪日志)、[sort / uniq / wc / cut / tr / tee / xargs](#215-配套命令排序计数分列与批量处理)。

## 1. 使用约定

- 代码块是独立示例，不要整篇连续执行。`12345`、`8080`、`com.example.app`、`example.com` 和网卡名需替换为实际值。
- 主机示例使用 Bash / Zsh。Android 命令默认从电脑执行，`adb shell` 后的命令才运行在设备上。
- `>` 覆盖输出文件，`>>` 追加；持续运行的日志、抓包或监听命令用 `Ctrl+C` 停止。
- Linux 通常使用 GNU/procps、iproute2；macOS 使用 BSD 系列工具；Android 常见 Toybox。相同命令名不保证支持相同参数。
- `sudo` 用于需要额外权限的查看操作；Android 正式版设备通常不支持 `adb root`，权限不足会导致信息缺失。
- 查本机支持的参数：Linux/macOS 用 `man 命令名`，Android 用 `adb shell 命令名 --help`；`nc` 常用 `nc -h`。精简系统可能需额外安装 `lsof`、`nc`、`dig` 等工具。

**如何读懂一条命令**

一般结构为 `命令 [选项] [操作对象]`。例如 `ps -ef` 可拆成 `ps -e -f`：`-e` 选择所有进程，`-f` 使用完整格式。很多无参数的短选项可以合并，但接受参数的选项要连同其值一起理解，例如 `tail -n 20 app.log` 中 `20` 是 `-n` 的值，`app.log` 是输入文件。

同一个字母在不同命令里含义不同：`ps -f` 是完整格式，`tail -f` 是持续跟踪，`grep -f patterns.txt` 是从文件读取匹配模式。`ps aux` 还使用不带连字符的 BSD 风格选项，不能机械改成 `ps -aux`。

`|` 将前一命令的标准输出传给下一命令；`2>` 重定向标准错误，`2>&1` 将标准错误送到标准输出当前指向的位置。单引号通常用于保护正则表达式，避免 shell 展开 `$`、`*` 等字符。支持 `--` 的命令可用它结束选项解析，例如 `grep -n -- '-error' app.log`。

## 2. Linux 与 macOS

### 2.1 ps：查看进程快照

`ps`（process status，进程状态）展示执行时刻的进程快照；持续观察负载用 `top`。

| 写法 / 参数 | 含义与拆解 |
| --- | --- |
| `-e` / `-A` | 选择所有进程 |
| `-f` | 完整输出格式（full format），常见字段包括 UID、PID、PPID 和命令行；不是“持续刷新” |
| `-ef` | `-e -f`：列出所有进程并使用完整格式 |
| `a` | BSD 风格：扩大终端相关进程的选择范围，包含其他用户的进程 |
| `x` | 包含没有控制终端的进程；与 `a` 合用通常用于查看所有进程 |
| `u` | 面向用户的格式，显示用户、CPU、内存等信息 |
| `aux` | `a`、`u`、`x` 组合：广泛列出进程，并显示资源占用 |
| `-p PID` | 按进程 ID 选择 |
| `-o 字段列表` | 自定义输出列，多个字段用逗号分隔 |
| `--sort=-rss` | Linux GNU/procps：按 RSS 降序；负号表示降序 |
| `-L` / `--forest` | Linux：显示线程 / 以树形缩进显示进程关系 |
| `-r` / `-m` | macOS：按 CPU 使用率 / 内存使用量排序 |

因此 `ps -ef` 偏向查看进程身份和父子关系，`ps aux` 偏向同时观察资源占用；两者的列布局不同。

```bash
# 两个平台均常用
ps aux
ps -ef
ps -p 12345 -o pid,ppid,user,%cpu,%mem,etime,command

# 按完整命令行查找进程；方括号写法避免匹配 grep 自身
ps aux | grep '[n]ginx'
pgrep -fl nginx
```

| 字段 | 含义 |
| --- | --- |
| PID / PPID | 进程 ID / 父进程 ID |
| USER | 所属用户 |
| %CPU / %MEM | CPU、内存占用指标；具体统计口径取决于实现 |
| VSZ / RSS | 虚拟内存大小 / 常驻物理内存，常见单位为 KiB |
| STAT | 状态，如 R 运行、S 睡眠、T 停止、Z 僵尸 |
| ELAPSED / ETIME | 自启动以来经过的时间 |
| TIME | 累计 CPU 时间，不是运行时长 |
| COMMAND / CMD | 命令及参数，长内容可能截断 |

**Linux：排序、线程与进程树**

```bash
ps aux --sort=-%cpu | head -n 11
ps aux --sort=-rss | head -n 11
ps -L -p 12345 -o pid,tid,pcpu,stat,comm
ps -ef --forest
top -H -p 12345
```

**macOS：排序与实时查看**

```bash
ps -axo pid,ppid,%cpu,%mem,rss,command -r | head -n 11
ps -axo pid,ppid,%cpu,%mem,rss,command -m | head -n 11
top -pid 12345
```

macOS 的 `ps` 不支持 GNU `--sort`、`--forest`；`top` 的参数也不能直接从 Linux 照搬。

`ps -axo ...` 拆成 `-a -x -o ...`。`pgrep -f` 按完整命令行匹配，`-l` 附带显示名称（不同系统显示细节有差异）。Linux `top -H` 显示线程，`-p PID` 限定进程；macOS 用 `top -pid PID`。`head -n 11` 只取前 11 行，即通常的一行表头加十条结果。

### 2.2 lsof：查看文件、端口与进程的关系

`lsof` 意为 list open files（列出打开的文件）。Unix 中 socket 也作为文件描述符管理，所以它既能查文件占用，也能查端口占用。

```bash
# 谁在监听 TCP 8080
sudo lsof -nP -iTCP:8080 -sTCP:LISTEN

# 所有 TCP 监听，以及 UDP socket
sudo lsof -nP -iTCP -sTCP:LISTEN
sudo lsof -nP -iUDP

# 指定进程打开的文件；指定进程的网络连接
lsof -p 12345
lsof -nP -a -p 12345 -i

# 已建立的 TCP 连接
lsof -nP -iTCP -sTCP:ESTABLISHED

# 指定文件被谁打开
lsof /path/to/file

# 递归查看目录内被打开的文件，大目录可能较慢
lsof +D /path/to/directory

# 已删除但仍被进程打开的文件，可用于排查磁盘空间未释放
sudo lsof +L1
```

| 参数 / 字段 | 含义 |
| --- | --- |
| `-n` / `-P` | 不反查主机名 / 不把端口号转成服务名 |
| `-i` | 网络文件，可限定 TCP、UDP、地址、端口 |
| `-p` | 指定进程 |
| `-a` | 对筛选条件取交集；如指定 PID **且**为网络文件 |
| `-sTCP:LISTEN` | 只看 TCP 监听状态 |
| `+D 目录` | 递归查找目录下被打开的文件 |
| `+L1` | 选择链接计数小于 1 的打开文件，常用于查找已删除但仍被占用的文件 |
| `FD` | 文件描述符，如 `5u`；`u` 表示读写，另有 `cwd`、`txt` 等 |
| `NAME` | 路径或 socket 地址；连接常显示为本地地址 → 对端地址 |

注意：多个选择条件在 `lsof` 中通常按“或”组合，按进程过滤网络连接时不要漏掉 `-a`。`-iTCP:8080` 可匹配连接两端的端口，查询本机监听者应同时限定 `LISTEN`。无结果也可能是权限不足。

### 2.3 nc：测试 TCP / UDP 与临时收发

`nc`（netcat）适合验证传输层连通性。以下采用 macOS / OpenBSD 风格；Linux 上先用 `nc -h` 确认实现。

```bash
# TCP 连接探测：不传业务数据，输出连接结果
nc -vz -w 3 example.com 443

# macOS 可显式限制连接建立超时
nc -vz -G 3 example.com 443

# 强制 IPv4 / IPv6
nc -4 -vz example.com 443
nc -6 -vz example.com 443
```

`-v` 输出细节，`-z` 探测连接，`-w` 设置超时，但不同实现的超时行为有差异。TCP 探测成功只说明连接可建立，不能证明 TLS 或应用接口正常。

`-vz` 即 `-v -z`；`-l` 表示监听，`-u` 改用 UDP（默认 TCP），`-4/-6` 强制 IPv4/IPv6；macOS 的 `-G 3` 限制连接建立等待时间为 3 秒。

**本机 TCP 收发：分别在两个终端执行**

```bash
# 终端 A：监听本机回环地址
nc -l 127.0.0.1 9000
```

```bash
# 终端 B：连接后输入文本并回车，终端 A 可看到内容
nc 127.0.0.1 9000
```

**本机 UDP 收发：分别在两个终端执行**

```bash
# 终端 A
nc -u -l 127.0.0.1 9001
```

```bash
# 终端 B
printf 'hello\n' | nc -u -w 1 127.0.0.1 9001
```

UDP 没有连接握手，不能仅凭 `nc -uz` 的成功提示断言对端端口可用；应结合接收端输出或抓包判断。传统 netcat 的监听语法可能为 `nc -l -p 9000`；`-N`、`-q` 等 EOF 行为选项也不通用。上述 TCP 示例采用交互方式，结束时手动 `Ctrl+C`。

### 2.4 地址、网卡、路由与邻居

| 目的 | Linux | macOS |
| --- | --- | --- |
| 查看网卡与 IP | `ip -br addr` | `ifconfig` |
| 查看网卡链路 | `ip link show` | `ifconfig en0` |
| IPv4 路由表 | `ip route show` | `netstat -rn -f inet` |
| IPv6 路由表 | `ip -6 route show` | `netstat -rn -f inet6` |
| 查看到目标的选路 | `ip route get 1.1.1.1` | `route -n get 1.1.1.1` |
| IPv4 邻居 / ARP | `ip neigh show` | `arp -an` |
| IPv6 邻居 | `ip -6 neigh show` | `ndp -an` |
| 网卡流量与错误计数 | `ip -s link` | `netstat -ib` |

Linux 新环境优先使用 `ip`、`ss`，旧环境常见 `ifconfig`、`route`、`netstat`。macOS 网卡名不一定是 `en0`，可用 `networksetup -listallhardwareports` 查看硬件端口与设备名的对应关系。

Linux 的 `ip route get` 可显示出口设备、下一跳与源地址。VPN、策略路由或不同网络命名空间可能使应用实际选路不同。

| 命令片段 | 含义 |
| --- | --- |
| `ip addr/link/route/neigh` | 分别操作地址、链路、路由、邻居对象；`show` 为显示，`get` 为查询到目标的路由 |
| `ip -br` / `-6` / `-s` | 简短输出 / IPv6 / 统计信息 |
| `netstat -rn` | `-r` 显示路由表，`-n` 使用数字地址 |
| `netstat -f inet/inet6` | macOS：限定 IPv4 / IPv6 地址族 |
| `netstat -ib` | macOS：`-i` 显示接口统计，`-b` 附带字节计数 |
| `arp -an`、`ndp -an` | `-a` 显示所有条目，`-n` 使用数字地址 |
| `route -n get 地址` | macOS：以数字形式显示到目标的路由查询结果 |

### 2.5 socket 与连接状态

```bash
# Linux：监听中的 TCP / UDP socket，附带进程信息
sudo ss -lntup

# Linux：所有 TCP 连接，查看 TCP 统计摘要
ss -ant
ss -s

# Linux：限定本地 TCP 8080
sudo ss -lntp 'sport = :8080'

# macOS：查看 TCP / UDP socket
netstat -anv -p tcp
netstat -anv -p udp

# macOS：查看端口对应进程
sudo lsof -nP -iTCP:8080 -sTCP:LISTEN
```

`ss` 中 `-l` 为监听、`-n` 为数字地址、`-t/-u` 为 TCP/UDP、`-p` 为进程、`-a` 为所有状态。macOS 的 `netstat -p` 用于协议选择，与 Linux 的进程选项含义不同。

`ss` 用于查看 socket 统计，`-s` 输出汇总，`sport` 表示本地端口。因此 `ss -lntup` 等价于 `ss -l -n -t -u -p`。macOS `netstat -anv` 中 `-a` 包含监听 socket，`-n` 禁用名称转换，`-v` 增加细节。

| 状态 | 排查含义 |
| --- | --- |
| LISTEN | 正在等待连接 |
| SYN-SENT | 已发送建连请求，等待回应；结合抓包检查网络和对端 |
| ESTABLISHED | TCP 连接已建立，不代表请求一定成功 |
| TIME-WAIT | 主动关闭一侧常见的协议等待状态，存在本身不代表故障 |
| CLOSE-WAIT | 对端已关闭，等待本地程序关闭；长期积累需检查应用 |

### 2.6 DNS 查询

```bash
dig example.com A
dig example.com AAAA
dig +short example.com
dig @1.1.1.1 example.com A
nslookup example.com
```

`dig @...` 指定解析服务器；返回结果反映该 DNS 服务器的视角，不一定等于系统或应用实际解析结果。

`dig` 是 DNS 查询工具（domain information groper）；`A` 查询 IPv4 地址，`AAAA` 查询 IPv6 地址，`+short` 精简输出，`@服务器` 指定 DNS 服务器。`nslookup` 也用于查询名称解析。macOS `dscacheutil -q host -a name example.com` 中，`-q host` 选择主机记录查询，`-a name ...` 指定名称属性。

```bash
# Linux：使用 systemd-resolved 的系统
resolvectl status
resolvectl query example.com

# Linux：观察系统名称解析结果与解析配置
getent hosts example.com
cat /etc/resolv.conf

# macOS：查看系统解析器配置与名称查询
scutil --dns
dscacheutil -q host -a name example.com
```

Linux 不一定安装或启用 `systemd-resolved`。macOS 的 `/etc/resolv.conf` 不能完整体现按接口、VPN 或域名划分的解析配置；应用还可能使用独立的 DoH 解析。

### 2.7 连通性与路径

```bash
ping -c 4 1.1.1.1
ping -c 4 example.com
traceroute example.com

# Linux IPv6
ping -6 -c 4 example.com

# macOS IPv6
ping6 -c 4 example.com
```

IP 可达但域名失败时优先检查 DNS。`ping` 失败可能只是 ICMP 被过滤；`traceroute` 的 `*` 表示该探测未收到响应，并不直接证明业务流量在该跳中断。

`ping` 发送 ICMP 回显探测，`-c 4` 表示发送 4 次；Linux `-6` 选择 IPv6，macOS 示例使用 `ping6`。`traceroute` 通过逐步增加 TTL / Hop Limit 探测路径上的响应节点。

### 2.8 HTTP / HTTPS 与 TLS

```bash
# 获取响应头：发送 HEAD，某些接口不支持
curl -I --connect-timeout 5 --max-time 15 https://example.com

# 实际 GET，只显示响应头，丢弃正文
curl -sS -D - -o /dev/null https://example.com

# 查看请求、连接与 TLS 细节
curl -v --connect-timeout 5 --max-time 15 https://example.com

# 跟随重定向，保存响应正文
curl -L -o response.html https://example.com

# 指定目标 IP，同时保留 HTTPS 主机名、SNI 与证书校验
curl --resolve example.com:443:203.0.113.10 https://example.com

# 查看 DNS、TCP、TLS、首字节及总耗时，单位秒
curl -sS -o /dev/null -w 'dns=%{time_namelookup}\nconnect=%{time_connect}\ntls=%{time_appconnect}\nttfb=%{time_starttransfer}\ntotal=%{time_total}\n' https://example.com

# 直接观察 TLS 握手与服务端发送的证书链
openssl s_client -connect example.com:443 -servername example.com -showcerts </dev/null
```

`203.0.113.10` 是文档示例地址，需替换为真实服务 IP。curl 的各时间指标通常是从请求开始计时的累计时间点，不是互相独立的阶段耗时；代理、重定向与连接复用也会影响解释。`openssl s_client` 是诊断工具，其校验行为与浏览器不同，不能仅凭完成握手认定主机名和信任链都有效。

| 参数 | 含义 |
| --- | --- |
| `curl -I` / `-v` | 发送 HEAD 请求 / 输出连接和协议调试细节 |
| `-sS` | `-s` 隐藏进度等常规输出，`-S` 仍显示错误 |
| `-D -` / `-o 文件` | 将响应头写到标准输出 / 将响应正文写到指定文件；`/dev/null` 表示丢弃 |
| `-L` | 跟随 HTTP 重定向 |
| `--connect-timeout` / `--max-time` | 连接阶段 / 整次传输的时间上限，单位秒 |
| `--resolve 主机:端口:IP` | 为特定主机和端口指定解析地址 |
| `-w 格式` | 传输结束后输出指定统计字段 |
| `openssl s_client` | 以 TLS 客户端身份连接服务 |
| `-connect` / `-servername` / `-showcerts` | 指定地址与端口 / 设置 SNI / 显示服务端发来的证书列表 |

### 2.9 tcpdump：抓包

```bash
# 列出可抓取接口
sudo tcpdump -D

# Linux：所有接口，限制包数
sudo tcpdump -i any -nn -c 100 'tcp port 443'

# macOS：替换为实际网卡；本机回环流量用 lo0
sudo tcpdump -i en0 -nn -c 100 'tcp port 443'

# 保存抓包供 Wireshark 等工具分析
sudo tcpdump -i en0 -nn -s 0 -c 1000 -w capture.pcap 'host 203.0.113.10 and tcp port 443'

# 离线读取
tcpdump -nn -r capture.pcap
```

Linux 的回环接口通常为 `lo`，保存示例中的 `en0` 也需按平台替换。`-nn` 不解析名称和服务，`-s 0` 使用完整抓包长度，`-w` 写入文件，`-r` 读取文件。HTTPS 通常只能观察握手、时序及加密后的数据，无法直接看到明文业务内容。

`tcpdump` 是命令行抓包分析工具；`-D` 列举接口，`-i` 选择接口，`-c N` 捕获 N 个包后退出。末尾引号内是抓包过滤表达式：`host` 限定主机，`tcp port` 限定 TCP 端口，`and` 表示同时满足。

### 2.10 grep：按内容筛选行

`grep` 从输入中找出匹配模式的行，名称源自编辑器的 `g/re/p`（全局查找正则并打印）。它查的是**内容**；按文件名查找用 `find`。

| 参数 | 含义 |
| --- | --- |
| `-n` / `-H` | 显示行号 / 总是显示文件名 |
| `-i` / `-v` | 忽略大小写 / 选择不匹配的行 |
| `-r` | 递归搜索目录；符号链接行为需注意实现差异 |
| `-E` / `-F` | 使用扩展正则 / 按固定字符串匹配，不解释正则 |
| `-w` / `-x` | 匹配完整单词 / 匹配完整行；单词通常由字母、数字、下划线组成 |
| `-e 模式` / `-f 文件` | 指定模式，可重复 / 从文件逐行读取模式；多个模式按“或”匹配 |
| `-l` / `-c` | 仅列匹配文件名 / 统计匹配行数（不是出现次数） |
| `-o` | 只输出各个非空匹配片段 |
| `-A N` / `-B N` / `-C N` | 附带后 N 行 / 前 N 行 / 前后各 N 行 |
| `-q` | 静默模式，通常用于按退出码判断是否匹配 |

```bash
grep -n 'ERROR' app.log
grep -in 'timeout' app.log
grep -nC 3 'Exception' app.log
grep -nE 'ERROR|WARN' app.log
grep -nF '192.168.1.10' app.log
grep -vE '^[[:space:]]*(#|$)' config.conf
grep -rn --include='*.java' 'TODO' ./src
grep -rl 'TODO' ./src
grep -c 'ERROR' app.log
grep -e 'timeout' -e 'refused' app.log

# 找到匹配时进入分支；非零还可能是读取出错，不能一律当成“没找到”
if grep -q 'ERROR' app.log; then
    printf '发现错误日志\n'
fi
```

`grep -nF '192.168.1.10'` 用固定字符串匹配 IP，避免正则里的 `.` 匹配任意字符。正则中 `^`、`$` 表示行首、行尾；`[[:space:]]` 表示空白字符；`*` 表示前一元素重复零次或多次。`-E` 下 `|` 表示或，`()` 用于分组，`+` 表示一次或多次。

**Linux / macOS 差异**：GNU grep 可提供 `-P`（PCRE），macOS 系统 grep 通常不支持；通用示例优先用 `-E` 或 `-F`。GNU `-r` 和 `-R` 对递归遇到的符号链接处理不同，BSD 实现也有自身规则；需要明确控制遍历时，用下面的 `find ... -exec grep ...`。`--include`、`--line-buffered` 在常见 GNU 和 macOS grep 中可用，但不是所有精简实现都支持。

### 2.11 find：按文件名与属性查找

`find` 遍历目录树，对每个文件应用条件和动作。基本结构是 `find 起始路径 条件 动作`；不指定动作时通常打印匹配路径。它默认不会像 shell 的 `*` 那样忽略隐藏文件。

| 条件 / 动作 | 含义 |
| --- | --- |
| `-name '*.log'` / `-iname '*.log'` | 按文件名通配匹配 / 忽略大小写；不是正则表达式 |
| `-path '*/build/*'` | 对整个路径进行通配匹配 |
| `-type f/d/l` | 普通文件 / 目录 / 符号链接 |
| `-size +10485760c` | 大小超过 10485760 字节；`c` 表示字节，`+` 大于、`-` 小于 |
| `-mtime N` / `-mmin N` | 按修改时间距今的天数 / 分钟数筛选；取整差异见下文 |
| `-newer 文件` | 修改时间晚于参考文件 |
| `-maxdepth N` | 最多向下遍历 N 层，起始路径为第 0 层 |
| `-prune` | 不再进入当前目录；常与 `-o` 组合排除目录 |
| `-print` / `-print0` | 输出路径，以换行 / NUL 分隔 |
| `-exec 命令 {} \;` | 对每条匹配路径执行一次命令，`{}` 替换为路径 |
| `-exec 命令 {} +` | 将多条路径成批传给命令，减少启动次数 |
| `!` / `-o` / `\( ... \)` | 非 / 或 / 分组；相邻条件默认是“且” |

```bash
find . -type f -name '*.log'
find ./src -type f \( -name '*.c' -o -name '*.h' \)
find . -type f -size +10485760c
find . -maxdepth 2 -type d
find . -type f -newer reference.txt

# 排除 .git 和 node_modules，再查 Markdown 文件
find . -type d \( -name .git -o -name node_modules \) -prune -o -type f -name '*.md' -print

# 按文件名筛选后搜索内容；路径含空格也能正确作为单个参数传入
find ./src -type f -name '*.java' -exec grep -nH 'TODO' {} +
```

`'*.log'` 的引号防止通配符先被 shell 展开。圆括号前加反斜杠，是为了把分组符号交给 `find`；`\;` 同理。`-o` 优先级低于隐含的“且”，多个文件名条件应像示例一样分组。

**Linux / macOS 差异**：

- GNU `find` 支持 `-printf`；macOS 系统 `find` 通常不支持。需要格式化元数据时可配合 `stat`，但 GNU `stat -c` 与 macOS `stat -f` 也不通用。
- GNU 扩展正则示例：`find . -regextype posix-extended -regex '.*\.(c|h)'`；macOS：`find -E . -regex '.*\.(c|h)'`。`-regex` 匹配整个路径。
- 当前 macOS `find` 支持 `-maxdepth`、`-mindepth`，它们并非 POSIX 标准选项，旧系统或其他实现仍需检查帮助。
- GNU `-mtime` 按完整 24 小时向下取整，例如 `-mtime 1` 通常匹配距今 24～48 小时，`-mtime +7` 通常从满 8 天起匹配。本机 macOS BSD 手册描述为向上取整，不能直接照搬 GNU 的边界解释。`-ctime` 是元数据变更时间，不是创建时间。

精确表达“最近 24 小时”时，可创建一个参考时间文件，再用两边都有的 `-newer`：

```bash
find_ref=$(mktemp)

# Linux：选择这一行
touch -t "$(date -d '24 hours ago' '+%Y%m%d%H%M.%S')" "$find_ref"

# macOS：选择这一行，不要与上一行连续执行
touch -t "$(date -v-24H '+%Y%m%d%H%M.%S')" "$find_ref"

find ./logs -type f -newer "$find_ref"
rm "$find_ref"
```

`mktemp` 创建临时文件，`touch -t` 设置时间；GNU `date -d` 解析相对时间，macOS `date -v-24H` 向前调整 24 小时。此例按秒设置阈值，匹配严格晚于该阈值的文件，也会包含时间戳位于未来的文件。

### 2.12 awk：按字段处理与统计

`awk` 是面向记录和字段的文本处理语言，名称来自 Aho、Weinberger、Kernighan 三位作者的姓氏。基本形式为 `awk '条件 { 动作 }' 文件`；默认每行是一条记录，按连续空白拆分字段。

| 参数 / 语法 | 含义 |
| --- | --- |
| `-F ':'` | 指定输入字段分隔符；可为正则表达式 |
| `-v name=value` | 从命令行传入变量，在执行程序前赋值 |
| `-f script.awk` | 从文件读取 awk 程序 |
| `$0` / `$1` / `$NF` | 整条记录 / 第一字段 / 最后字段 |
| `NF` | 当前记录的字段数 |
| `NR` / `FNR` | 所有输入的累计记录序号 / 当前文件的记录序号 |
| `FS` / `OFS` | 输入字段分隔符 / 输出字段分隔符 |
| `BEGIN` / `END` | 读取输入前 / 所有输入结束后执行的块 |
| `print` / `printf` | 自动添加输出记录分隔符 / 按格式输出 |

```bash
# 输出用户名及 UID，输入以冒号分隔
awk -F ':' '{print $1, $3}' /etc/passwd

# 给每行加行号；筛选包含 ERROR 的记录
awk '{print NR, $0}' app.log
awk '/ERROR/ {print $0}' app.log

# 只输出第 10～20 行
awk 'NR >= 10 && NR <= 20' app.log

# 假设 metrics.txt 每行是“名称 耗时”，筛选耗时大于阈值的行
awk -v limit=100 '$2 > limit {print $1, $2}' metrics.txt

# 对第二列求和、平均值，处理空输入
awk 'NF >= 2 {sum += $2; n++} END {if (n) printf "sum=%.2f avg=%.2f\n", sum, sum/n; else print "no data"}' metrics.txt

# 按第一列聚合第二列
awk '{total[$1] += $2} END {for (key in total) print key, total[key]}' metrics.txt

# 设置输出字段间的分隔符
awk -F ':' 'BEGIN {OFS="\t"} {print $1, $3}' /etc/passwd
```

这些统计例子假定第二列为数值且没有表头；awk 可能将非数字字符串按 0 参与数值运算，处理真实数据应先验证格式。关联数组遍历顺序不固定，需要排序时将结果交给 `sort`。

**Linux / macOS 差异**：Linux 的 `awk` 可能是 gawk、mawk 或其他实现；macOS 系统 awk 也不能当作 gawk。上例使用通用语法，不依赖 `gensub()`、`asort()`、`--csv` 等 GNU 扩展。`awk -F ','` 只按逗号拆列，无法正确处理带引号、字段内逗号或换行的完整 CSV。脚本用单引号保护 `$1`，外部变量通过 `-v` 传入。

### 2.13 sed：筛选与替换文本

`sed` 是 stream editor（流编辑器）：逐行读取、执行编辑脚本并输出，默认不修改输入文件。

| 参数 / 语法 | 含义 |
| --- | --- |
| `-n` | 关闭默认的逐行输出，通常配合 `p` 精确打印 |
| `-E` | 使用扩展正则；现代 GNU sed 和 macOS sed 均支持 |
| `-e 脚本` / `-f 文件` | 指定编辑脚本，可重复 / 从文件读取脚本 |
| `-i` | 原地编辑；备份参数语法在 GNU 与 macOS 上不同 |
| `10,20p` / `10,20d` | 打印 / 删除第 10～20 行；未用 `-i` 时删除只影响输出 |
| `/模式/p` | 打印匹配模式的行 |
| `s/旧/新/` | 替换每行第一个匹配项 |
| `s/旧/新/g` | `g` 为全局替换，即替换每行所有匹配项 |
| 替换串中的 `&` / `\1` | 整个匹配文本 / 第一个捕获组 |

```bash
sed -n '10,20p' app.log
sed -n '/ERROR/p' app.log
sed '/^[[:space:]]*$/d' input.txt
sed 's/old/new/g' input.txt > output.txt
sed -E 's/[[:space:]]+$//' input.txt
sed -E 's/^([^=]+)=(.*)$/\2=\1/' config.txt
sed -e 's/foo/bar/g' -e 's/hello/world/g' input.txt

# 路径中含 / 时可换分隔符，减少转义
sed 's#/old/path#/new/path#g' config.txt
```

正则中的旧值和替换串的解释方式不同：旧值中的 `.` 是任意字符，替换串中的 `&` 会展开匹配文本；要替换成字面量 `&`，写成 `\&`。

**原地修改语法**：以下会真实改写文件，示例按需选择。

```bash
# Linux GNU sed：无备份
sed -i 's/old/new/g' file.txt

# macOS BSD sed：必须显式给出空备份后缀
sed -i '' 's/old/new/g' file.txt

# 两边都常用：修改并保留 file.txt.bak 备份
sed -i.bak 's/old/new/g' file.txt
```

不要写 `sed 's/old/new/g' file.txt > file.txt`：shell 会先截断文件，导致原内容丢失。跨平台脚本优先用 `-E`、`[[:space:]]` 和 `[[:digit:]]`，避免依赖 GNU 的 `\s`、`\d` 或文本追加的简写。`a`/`i` 追加或插入多行文本时，BSD 对换行和反斜杠的语法要求更严格。

### 2.14 tail、head、less：查看文件与跟踪日志

`tail` 读尾部，`head` 读头部，`less` 提供可前后移动的分页查看。

| 命令 / 参数 | 含义 |
| --- | --- |
| `tail -n N` / `head -n N` | 查看末尾 / 开头 N 行；默认通常为 10 行 |
| `tail -n +N` | 从第 N 行开始一直输出到末尾，行号从 1 开始 |
| `tail -c N` / `head -c N` | 查看末尾 / 开头 N 字节，可能截断多字节字符 |
| `tail -f` | 到达末尾后继续等待追加内容，通常跟踪已打开的文件 |
| `tail -F` | 按文件名跟踪并重试，适合文件被重命名、轮转后重新创建的日志 |
| `less -N` / `-S` | 显示行号 / 长行不折行，可左右移动 |

```bash
tail -n 100 app.log
tail -n +2 data.tsv
tail -n 0 -F app.log
head -n 20 app.log
less -N app.log

# 持续跟踪并筛选，--line-buffered 减少 grep 的管道输出缓冲
tail -F app.log | grep --line-buffered -nE 'ERROR|WARN'
```

`tail -n 0 -F` 只关注启动后的新增内容。`grep -n` 在管道中统计的是它收到的输入行号，并非原文件的绝对行号。`less` 内可用 `/关键词` 搜索、`n`/`N` 跳到下一个/上一个匹配、`g`/`G` 到开头/结尾、`F` 跟踪追加、`Ctrl+C` 退出跟踪、`q` 退出。

**Linux / macOS 差异**：两边都可用 `tail -f/-F`；GNU 的 `--follow=name --retry` 可用 `-F` 表达。macOS 的 `tail -r` 可倒序输出行，GNU tail 不支持，可改用 GNU `tac`。GNU `head -n -5` 表示去掉最后 5 行，这种负数写法不要作为跨平台通用示例。避免依赖 GNU 专有的 `tail --pid`。

### 2.15 配套命令：排序、计数、分列与批量处理

| 命令 | 用途与常用参数 |
| --- | --- |
| `sort` | 排序；`-n` 数值排序，`-r` 降序，`-k 2,2` 指定第二字段，`-t ':'` 指定分隔符，`-u` 去重 |
| `uniq` | 合并相邻重复行；`-c` 输出计数，`-d` 只输出重复组，`-u` 只输出出现一次的组 |
| `wc` | word count；`-l` 统计换行符，`-w` 统计单词，`-c` 字节，`-m` 字符 |
| `cut` | 提取列；`-d ':'` 指定单字符分隔符，`-f 1,3` 选择字段，`-c 1-10` 选择字符位置（多字节处理依实现） |
| `tr` | 转换或删除字符；`-d` 删除指定字符，`-s` 压缩指定字符的连续重复；从标准输入读取 |
| `tee` | 将输入同时写到终端和文件；默认覆盖，`-a` 为追加 |
| `xargs` | 将输入构造成命令参数；`-0` 用 NUL 分隔，`-n N` 每次最多 N 个输入参数，`-P N` 最多并发 N 个进程 |

```bash
# 按第二列数字降序；统计每个完整行的出现次数并按频次降序
sort -k 2,2nr metrics.txt
sort names.txt | uniq -c | sort -nr

# 只输出行数，不附带文件名；提取冒号分隔的字段
wc -l < app.log
cut -d ':' -f 1,3 /etc/passwd

# 转小写；移除 CR 字符（不只限于行尾 CR）
tr '[:upper:]' '[:lower:]' < input.txt
tr -d '\r' < input.txt > output.txt

# 将匹配结果追加到文件，同时显示
grep 'ERROR' app.log | tee -a errors.log

# 安全传递含空格、制表符或换行的路径
find ./logs -type f -name '*.log' -print0 | xargs -0 wc -l

# 若希望无匹配时完全不执行，优先直接使用 find -exec
find ./logs -type f -name '*.log' -exec wc -l {} +
```

`uniq` 只处理相邻重复行，因此通常先 `sort`；排序会改变原始顺序。`wc -l` 数的是换行符，末行没有换行时与肉眼看到的行数可能不同。`cut -d ' '` 不会把连续空格当作一个分隔符，空白分列优先用 awk。默认排序受 locale 影响，要稳定的按字节顺序可用 `LC_ALL=C sort 文件`，但它不是中文拼音排序。

**Linux / macOS 差异**：GNU `xargs` 在空输入时默认仍可能执行命令一次，`-r` 可禁止；macOS 的选项和空输入行为需核对本机手册，通用做法是 `find -exec ... {} +`。不要把 `find` 的换行输出直接交给普通 `xargs`：空格、引号等会被重新解释。大量输入分成多次调用时，`wc` 的 total 是每批汇总，不是全部文件的全局合计。GNU `sort` 的 `--parallel` 等长选项也不要直接用于 macOS。

## 3. Android

### 3.1 ADB 与执行位置

先安装 Android SDK Platform-Tools，在设备开启 USB 调试并授权电脑。

`adb` 是 Android Debug Bridge（Android 调试桥）；`devices -l` 列出设备及详细信息，`-s SERIAL` 选择设备，`shell` 在设备执行命令，`pull` 将设备文件复制到电脑。`getprop` 读取 Android 系统属性。

```bash
adb devices -l
adb -s SERIAL shell
adb shell getprop ro.build.version.release
adb shell getprop ro.build.version.sdk
```

多设备时，把后续 `adb` 替换为 `adb -s SERIAL`。`unauthorized` 需在设备确认授权，`offline` 表示当前连接不可用。

```bash
# > 由电脑 shell 处理：文件保存在电脑
adb logcat -d > logcat.txt

# 引号内的重定向由设备 shell 处理：文件保存在设备
adb shell 'logcat -d > /sdcard/logcat.txt'
adb pull /sdcard/logcat.txt .
```

### 3.2 adb logcat：读取、筛选与保存日志

`logcat` 读取 Android 日志缓冲区，`adb logcat` 让电脑通过 ADB 调用它。

| 参数 | 含义 |
| --- | --- |
| `-v threadtime` | 使用包含线程 ID 和时间等字段的输出格式；这里 `-v` 是格式选项，不是日志 Verbose 级别 |
| `-d` | 输出已有日志后退出，不持续等待 |
| `-t N` / `-T N` | 最近 N 条后退出 / 从最近 N 条开始持续跟踪 |
| `-b 名称` | 选择日志缓冲区；`all` 为全部可访问缓冲区 |
| `-g` / `-c` | 查看缓冲区大小 / 清空所选缓冲区 |
| `--pid=PID` | 仅显示指定进程 ID 的日志 |

```bash
# 持续查看，输出时间、PID、TID、优先级及 tag
adb logcat -v threadtime

# 输出当前缓冲区后退出；从最近 200 条开始持续跟踪
adb logcat -d -v threadtime > logcat.txt
adb logcat -t 200
adb logcat -T 200 -v threadtime

# 按级别、tag 筛选；引号防止 * 被电脑 shell 展开
adb logcat '*:W'
adb logcat 'MyTag:D' 'AndroidRuntime:E' '*:S'

# 查看崩溃缓冲区或全部可访问缓冲区
adb logcat -b crash -d -v threadtime
adb logcat -b all -v threadtime

# 查看缓冲区大小；清空日志缓冲区（会删除已有日志）
adb logcat -g
adb logcat -c

# 先查 PID，再使用实际 PID 过滤
adb shell pidof com.example.app
adb logcat --pid=12345 -v threadtime

# 在电脑上保留完整日志，并实时显示
adb logcat -v threadtime | tee logcat.txt
```

| 概念 | 含义 |
| --- | --- |
| 优先级 | `V` Verbose、`D` Debug、`I` Info、`W` Warn、`E` Error、`F` Fatal；`S` Silent |
| `TAG:级别` | 显示该 tag 在指定级别及以上的日志；`*:S` 关闭其他 tag |
| `main` / `system` / `crash` | 应用相关 / 系统 / 崩溃日志缓冲区 |
| `events` / `radio` | 结构化事件 / 通信相关缓冲区 |
| `-t N` / `-T N` | 最近 N 条并退出 / 从最近 N 条开始持续跟踪 |

应用重启后 PID 会变化，需重新获取；多进程应用可能有 `com.example.app:remote` 等额外进程。只抓单一 PID 也可能漏掉系统服务输出的崩溃或 ANR 线索。支持的参数、缓冲区与可见日志取决于设备版本和权限，使用 `adb logcat --help` 核实。

### 3.3 ps：进程与线程

```bash
adb shell ps -A
adb shell 'ps -A | grep com.example.app'
adb shell pidof com.example.app

# 以下参数按设备 Toybox 版本确认
adb shell ps -A -o PID,PPID,USER,NAME
adb shell ps -T -p 12345
adb shell top -b -n 1
adb shell top -H -p 12345

# 查看应用内存与 Activity 状态
adb shell dumpsys meminfo com.example.app
adb shell dumpsys activity activities
```

不要直接套用桌面 Linux 的 `ps aux --sort=...`。`ps -A` 用于枚举进程，线程选项和可选字段以 `adb shell ps --help` 为准。

Android 示例中 `ps -o` 自定义字段，`-p` 选择 PID，`-T` 显示线程。Toybox `top -b` 为批处理输出，`-n 1` 只刷新一次，`-H` 显示线程。`pidof` 按进程名获取 PID；`dumpsys` 输出系统服务的诊断状态，后面的 `meminfo`、`activity`、`connectivity`、`wifi` 是服务名，不是通用短选项。

### 3.4 lsof 与 /proc：查看打开的文件

```bash
# 先确认设备是否提供命令
adb shell 'command -v lsof'
adb shell lsof --help

# 仅在设备帮助列出 -p 参数时使用
adb shell lsof -p 12345

# 查看进程的文件描述符，受 UID、SELinux 和 /proc 权限限制
adb shell ls -l /proc/12345/fd
```

Android 上的 `lsof` 可能缺失或仅支持少量参数，不能默认支持桌面版的 `-i`、`-nP` 等。`Permission denied` 不代表进程没有打开文件。对支持 `run-as` 的可调试应用，可尝试以应用身份查看：

```bash
adb shell run-as com.example.app ls -l /proc/12345/fd
```

此方式仍受设备策略限制，且 PID 必须属于目标应用。

### 3.5 网络状态与连通性

```bash
adb shell ip addr show
adb shell ip route show
adb shell ip rule show
adb shell ip route show table all
adb shell ip neigh show

adb shell ping -c 4 1.1.1.1
adb shell ping -c 4 example.com

# Android 系统视角的网络、Wi-Fi 状态
adb shell dumpsys connectivity
adb shell dumpsys wifi

# 设备提供 ss 时使用，参数和权限依实现而定
adb shell ss -lnt
adb shell ss -ant
```

Android 使用策略路由，单看主路由表可能不完整。ADB shell、具体应用、VPN、应用绑定的网络可能走不同路径；shell 能联网不代表应用一定能联网。`dumpsys` 输出格式随系统和厂商变化。

### 3.6 nc：以设备为测试端

```bash
adb shell 'command -v nc'
adb shell nc --help

# 设备提供 nc 时，建立交互式 TCP 连接
adb shell nc 192.168.1.10 9000
```

将 IP 替换为电脑的局域网地址，并先在电脑上启动绑定该地址的测试监听，例如 `nc -l 192.168.1.10 9000`；同时确认网络和防火墙允许访问。连接后输入文本并回车，在电脑观察接收结果。

Android 的 Toybox/BusyBox `nc` 参数可能不同，`-z`、`-v`、监听选项均需以设备帮助为准。设备通常不预装 `curl`、`dig`、`tcpdump`，不能把主机工具可用等同于设备工具可用。

### 3.7 端口转发：电脑与设备访问对方服务

```bash
# 电脑 localhost:18080 → 设备 localhost:8080
adb forward tcp:18080 tcp:8080
curl http://127.0.0.1:18080
adb forward --list
adb forward --remove tcp:18080

# 设备 localhost:8080 → 电脑 localhost:3000
adb reverse tcp:8080 tcp:3000
adb reverse --list
adb reverse --remove tcp:8080
```

`forward` 用于电脑访问设备服务；`reverse` 常用于设备上的调试应用访问电脑开发服务。目标服务需已启动。设备里的 `127.0.0.1` 原本表示设备自身；通过 `reverse` 映射后，指定端口才会转到电脑。

## 4. 常用排查流程

### 4.1 “8080 端口被占用”

1. Linux 用 `sudo ss -lntp 'sport = :8080'`；Linux/macOS 也可用 `sudo lsof -nP -iTCP:8080 -sTCP:LISTEN`。
2. 记录 PID，用 `ps -p 12345 -o pid,ppid,user,command` 确认所属程序和启动来源。
3. 查看监听地址：`127.0.0.1` 只服务本机；`0.0.0.0` 表示所有 IPv4 本地地址；IPv6 通配监听是否兼容 IPv4 取决于配置。
4. 确认程序用途后，按需修改端口或通过该服务正常的停止方式处理。

### 4.2 “域名 / 接口连不上”

1. 查看网卡、地址和到目标的路由，确认出口及 VPN 状态。
2. 用 `dig` 和系统解析工具对照结果，排除解析差异。
3. 用 `nc -vz` 检查目标 TCP 端口。
4. 用 `curl -v` 观察 TLS、HTTP 状态和响应；必要时用 `--resolve` 对比特定 IP。
5. 用 `tcpdump` 检查握手、重传、RST 与响应时序。

连接拒绝通常意味着收到了主动拒绝；超时可能涉及丢包、过滤、路由或对端无响应，应结合服务端状态与抓包判断。

### 4.3 “Android 应用启动崩溃 / 网络异常”

1. 用 `adb devices -l` 确认目标设备。
2. 复现前启动 `adb logcat -b all -v threadtime > reproduce.log`，复现后 `Ctrl+C` 保存。
3. 用 `adb logcat -b crash -d -v threadtime` 补查崩溃缓冲区，按时间、tag、PID 定位。
4. 用 `adb shell ps -A` 和 `adb shell dumpsys meminfo com.example.app` 检查进程及内存。
5. 网络问题结合 `dumpsys connectivity`、路由和应用日志；不要仅用电脑访问成功作为设备网络正常的证据。

## 5. 参考资料

- [Android Developers：ADB](https://developer.android.com/tools/adb) — 连接设备、执行 shell 与端口转发。
- [Android Developers：Logcat](https://developer.android.com/tools/logcat) — 日志格式、筛选与缓冲区。
- [Linux ps 手册](https://man7.org/linux/man-pages/man1/ps.1.html)、[ip 手册](https://man7.org/linux/man-pages/man8/ip.8.html)、[ss 手册](https://man7.org/linux/man-pages/man8/ss.8.html) — Linux 参数参考。
- [OpenBSD nc 手册](https://man.openbsd.org/nc) — nc 的一种实现；macOS 及其他实现应再核对本机帮助。
- [curl 官方手册](https://curl.se/docs/manpage.html) — HTTP 请求、解析覆盖与耗时指标。
- macOS 本机手册：`man ps`、`man lsof`、`man nc`、`man netstat`、`man tcpdump`。本机手册优先于其他平台的同名命令说明。
- [GNU grep 手册](https://www.gnu.org/software/grep/manual/grep.html)、[GNU findutils 手册](https://www.gnu.org/software/findutils/manual/)、[GNU sed 手册](https://www.gnu.org/software/sed/manual/sed.html) — 文件检索与文本处理选项。
- 文件处理的 macOS 差异另按本机 `man grep`、`man find`、`man awk`、`man sed`、`man tail` 核对，尤其是 `find` 时间边界和 `sed -i`。

## 6. Windows
