# WSL2 内 GitHub 不可达：从 hosts 劫持到中间人证书链

- **日期**：2026-10
- **环境**：Windows 11 24H2（build 26100）+ WSL2 Ubuntu 24.04
- **症状**：Windows 侧访问 GitHub 正常，WSL 内所有 GitHub 域名瞬时失败
- **结论**：本机 hosts 型加速器劫持域名 + WSL 未继承其信任链

## 1. 现象

```console
$ git clone https://github.com/ros-controls/zenbedded.git
Cloning into 'zenbedded'...
fatal: unable to access 'https://github.com/ros-controls/zenbedded.git/':
Failed to connect to github.com port 443 after 2 ms: Couldn't connect to server
```

## 2. 第一步判断：`2 ms` 说明了什么

`2 ms` 是**瞬时拒绝**，不是超时。

真实网络不可达（被墙、路由黑洞）表现为数秒到数十秒的**超时**；毫秒级拒绝只可能是"目标端口上根本没有监听"。

据此先推断：域名被解析到了**本机地址**。

```powershell
Resolve-DnsName github.com -Type A | Select-Object -First 1 -ExpandProperty IPAddress
# 127.0.0.1
```

## 3. 定位 hosts 劫持

而同一时刻 Windows 侧访问该域名却返回 `200` —— 说明本机 443 端口上有服务在监听并转发。

查看 hosts（共 905 行，其中数百条是同一模式）：

```
127.0.0.1 github.com #S302
127.0.0.1 raw.githubusercontent.com #S302
127.0.0.1 registry-1.docker.io #S302
...
```

`#S302` 是 **Steamcommunity 302**（即 Steam++ / Watt Toolkit）的标记。它的"加速"原理就是：改 hosts 把域名劫持到 `127.0.0.1`，再在本机起代理转发。

**关键点**：WSL2 默认会把 Windows 的 hosts 条目复制进 `/etc/hosts`。于是 WSL 内也解析成 `127.0.0.1` —— 但那个代理**只在 Windows 侧监听**，WSL 的 `127.0.0.1` 是它自己的回环命名空间，什么都没监听，于是瞬时拒绝。

> 这也解释了"宿主能用、容器/WSL 不能用"这类典型症状的成因。

## 4. 让 WSL 复用 Windows 侧的代理

`C:\Users\<用户>\.wslconfig`：

```ini
[wsl2]
networkingMode=mirrored
dnsTunneling=true
autoProxy=true
```

```powershell
wsl --shutdown
```

镜像网络模式下 WSL 与 Windows 共享回环，WSL 的 `127.0.0.1` 可转发到 Windows 侧服务。

验证：

```bash
wslinfo --networking-mode      # 期望输出 mirrored
```

## 5. 第二层问题：TLS 信任

错误码变了：

```
curl: (7)  Failed to connect to github.com port 443 ...          ← 连不上
curl: (60) SSL certificate problem: unable to get local issuer certificate   ← 连上了，证书不信任
```

**错误码的变化本身就是进展**：问题从网络层推进到了 TLS 层。

`(60)` 说明加速器是**中间人代理**，用自己的根证书重新签发了目标站点的证书。该根证书装在 Windows 证书存储里（所以 Windows 侧正常），但不在 WSL 的 CA 库中。

用 `SslStream` 直接握手，看实际证书链：

```powershell
$tcp = New-Object Net.Sockets.TcpClient
$tcp.Connect('github.com', 443)
$cb = [Net.Security.RemoteCertificateValidationCallback]{ param($s,$c,$ch,$e) $true }
$ssl = New-Object Net.Security.SslStream($tcp.GetStream(), $false, $cb)
$ssl.AuthenticateAsClient('github.com')
$leaf = New-Object Security.Cryptography.X509Certificates.X509Certificate2($ssl.RemoteCertificate)
$chain = New-Object Security.Cryptography.X509Certificates.X509Chain
$chain.ChainPolicy.RevocationMode = 'NoCheck'
$chain.Build($leaf) | Out-Null
$chain.ChainElements | ForEach-Object { $_.Certificate.Subject }
```

输出：

```
CN=steamcommunity.com, OU=XX, O=Steamcommunity302, C=CN      ← 叶证书
CN=Steamcommunity302, OU=XX, O=Steamcommunity302, C=CN       ← 自签根
```

### 一个容易误判的点

叶证书的 CN 是 `steamcommunity.com` 而不是 `github.com`，看起来像**主机名不匹配**，似乎装了根证书也没用。

但现代 TLS 校验看的是 **SAN（Subject Alternative Name）**，不是 CN。该证书的 SAN 覆盖约 150 个域名，其中包含：

```
DNS Name=github.com, DNS Name=*.github.com, DNS Name=*.githubusercontent.com,
DNS Name=docker.io, DNS Name=*.docker.io, DNS Name=huggingface.co, ...
```

所以**装根证书就够了**，不必退化成 `sslVerify=false`。

### 顺带踩的坑

第一版取证书用的是 `HttpWebRequest.ServicePoint.Certificate`，它返回了 `null`，导致"签发者"是空字符串；而空字符串做通配匹配会命中根存储里的**第一张**证书 —— 结果导出了一张毫不相干的证书。

> **教训**：把可能为空的值用于通配匹配之前，先判空。

## 6. 修复

导出根证书为 PEM（上面的 `$chain` 末元素），装入 WSL 信任库：

```bash
sudo cp /mnt/.../accel-ca.crt /usr/local/share/ca-certificates/steamcommunity302.crt
sudo update-ca-certificates          # 期望输出 "1 added"
curl -I https://github.com | head -1
```

```console
HTTP/2 200
```

> `curl: Failed writing body` 是无害的：`-I` 输出整个响应头，而 `head -1` 读完第一行就关闭管道，curl 写余下部分时收到 EPIPE。与证书、网络无关。

## 7. 验证与遗留问题

- `git clone https://github.com/ros-controls/zenbedded.git` 成功
- `rosdep update`、pip、HuggingFace 下载随之恢复（同一个根证书覆盖）
- **Docker 是例外**：`docker pull` 不读系统 CA 库，它读 `/etc/docker/certs.d/<registry>/ca.crt`，需要单独放置

## 8. 复盘

1. **毫秒级连接失败 ⇒ 目标被解析到了本机**。这是最快的一条判别规则。
2. WSL/容器继承宿主 hosts 的行为，会把"宿主正常"和"容器异常"同时呈现出来，容易被误判成防火墙或墙。
3. **不要只看错误码，要看错误码的变化**：`(7)` → `(60)` 是从网络层推进到 TLS 层，是进展。
4. 安装第三方根证书 = 把该中间人纳入信任范围。这是使用此类加速器的固有代价，应当知情：此后这些域名的 TLS 对它是可见的。
