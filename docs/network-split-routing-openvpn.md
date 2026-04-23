# 阿里云 ECS 网络分流：中国直连 + 境外走 OpenVPN

> **场景**：阿里云杭州 ECS，国内流量直连，境外流量经 OpenVPN 隧道转发  
> **系统**：Ubuntu 20.04 / 22.04 LTS  
> **验证日期**：2026-04-22  
> **最终状态**：✓ 分流正常（github/anthropic 走 VPN，feishu/aliyun 直连）

---

## 架构

```
ECS (杭州, eth0: 172.16.216.154)
  ├── eth0 → 172.16.223.253 → 国内流量（直连）
  └── tun0 → 10.8.0.1 → VPN Server (8.219.240.184) → 境外流量
```

**分流逻辑**：以中国 IP 列表（~8800 条 CIDR）为分界，命中则直连，其余默认走 VPN 隧道。

---

## 文件清单

| 文件 | 用途 |
|------|------|
| `/etc/openvpn/client.ovpn` | OpenVPN 客户端配置 |
| `/etc/openvpn/vpn-up.sh` | VPN 建立后执行：配置分流路由 |
| `/etc/openvpn/vpn-down.sh` | VPN 断开后执行：恢复默认路由 |
| `/etc/openvpn/china_ips.txt` | 中国 IP 段列表（CIDR 格式） |
| `/etc/openvpn/scripts/update-chinaips.sh` | 更新中国 IP 列表 |
| `/etc/systemd/system/openvpn-split.service` | Systemd 服务 |
| `/etc/cron.d/chinaips-update` | 每周定时更新 IP 列表 |

---

## 一、中国 IP 列表

### 更新脚本

```bash
sudo mkdir -p /etc/openvpn/scripts

sudo tee /etc/openvpn/scripts/update-chinaips.sh << 'EOF'
#!/bin/bash
curl -sf 'https://ftp.apnic.net/apnic/stats/apnic/delegated-apnic-latest' \
  | grep 'apnic|CN|ipv4' \
  | awk -F'|' '{
      n = log($5)/log(2);
      printf "%s/%d\n", $4, 32-n
    }' \
  | sort -u > /etc/openvpn/china_ips.txt
echo "[$(date)] Updated: $(wc -l < /etc/openvpn/china_ips.txt) China IP ranges"
EOF

sudo chmod +x /etc/openvpn/scripts/update-chinaips.sh
sudo /etc/openvpn/scripts/update-chinaips.sh   # 首次执行
```

> **注意**：APNIC 下载在国内偶尔超时，不要放在 systemd `ExecStartPre` 中（会阻塞启动）。改用 cron 定期更新。

### 定时更新（每周一凌晨 3 点）

```bash
echo "0 3 * * 1 root /etc/openvpn/scripts/update-chinaips.sh" \
  | sudo tee /etc/cron.d/chinaips-update
```

---

## 二、OpenVPN 客户端配置

在现有 `client.ovpn` 末尾追加（确认已移除或注释掉 `redirect-gateway`）：

```bash
# 注释掉服务端推送的 redirect-gateway
sudo sed -i 's/^redirect-gateway/#redirect-gateway/' /etc/openvpn/client.ovpn

# 追加分流控制段
sudo tee -a /etc/openvpn/client.ovpn << 'EOF'

# === Split Routing ===
route-nopull
script-security 2
up /etc/openvpn/vpn-up.sh
down /etc/openvpn/vpn-down.sh
EOF
```

**关键说明**：
- `route-nopull`：阻止服务端下发路由，由 up 脚本全权接管
- `script-security 2`：允许执行外部脚本
- `up`/`down` 路径**不能有前导空格**，否则 shebang 解析失败（详见踩坑记录）

---

## 三、路由脚本

> **重要**：脚本必须放在 `/etc/openvpn/` 根目录，不能放子目录。
> Ubuntu AppArmor 限制 OpenVPN 只能执行 `/etc/openvpn/*.sh`，子目录不在白名单内。

### vpn-up.sh

```bash
sudo tee /etc/openvpn/vpn-up.sh << 'EOF'
#!/bin/bash
set -e
TUN_DEV="$1"

# 获取 VPN 网关
# topology subnet 模式下 $ifconfig_remote 为空（仅 p2p 模式才有）
# 改从 $ifconfig_local 推算：服务端始终是同网段的 .1
VPN_GW="${route_vpn_gateway}"
if [ -z "$VPN_GW" ] && [ -n "$ifconfig_local" ]; then
    VPN_GW=$(echo "$ifconfig_local" | awk -F. '{print $1"."$2"."$3".1"}')
fi
[ -z "$VPN_GW" ] && { echo "ERROR: cannot determine VPN gateway"; exit 1; }

# 获取并保存原始默认网关（eth0 网关）
ORIG_GW=$(ip route show default | awk '/default/{print $3; exit}')
[ -z "$ORIG_GW" ] && { echo "ERROR: no default gateway"; exit 1; }
echo "$ORIG_GW" > /run/openvpn-orig-gw.tmp
echo "[vpn-up] orig=$ORIG_GW vpn=$VPN_GW dev=$TUN_DEV"

# VPN 服务器公网 IP 固定走 eth0，防止隧道路由循环
VPN_SERVER_IP=$(awk '/^remote /{print $2; exit}' /etc/openvpn/client.ovpn)
[ -n "$VPN_SERVER_IP" ] && ip route replace "$VPN_SERVER_IP/32" via "$ORIG_GW"

# 当前 SSH 客户端 IP 强制直连，防止 SSH 会话中断
if [ -n "$SSH_CLIENT" ]; then
    SSH_SRC=$(echo "$SSH_CLIENT" | awk '{print $1}')
    ip route replace "$SSH_SRC/32" via "$ORIG_GW"
    echo "[vpn-up] SSH client $SSH_SRC -> Direct"
fi

# 中国 IP 段走直连（必须在删默认路由之前加，避免中间网络中断）
echo "[vpn-up] Loading $(wc -l < /etc/openvpn/china_ips.txt) China routes..."
while IFS= read -r cidr; do
    ip route add "$cidr" via "$ORIG_GW" 2>/dev/null || true
done < /etc/openvpn/china_ips.txt

# 阿里云保留段全部走直连
# 100.64.0.0/10 = CGNAT（阿里云 SSH 跳板常在此段，必须直连，否则 SSH 断线）
# 100.100.0.0/16 = 阿里云元数据/内网服务
for net in 10.0.0.0/8 172.16.0.0/12 192.168.0.0/16 100.64.0.0/10 100.100.0.0/16; do
    ip route add "$net" via "$ORIG_GW" 2>/dev/null || true
done

# 删除旧默认路由，换成 VPN（放最后：避免操作过程中断网）
ip route del default via "$ORIG_GW" 2>/dev/null || true
ip route add default via "$VPN_GW" dev "$TUN_DEV"

echo "[vpn-up] Done. Non-China->VPN($VPN_GW)  China->Direct($ORIG_GW)"
EOF

sudo chmod 755 /etc/openvpn/vpn-up.sh

# 验证 shebang 无前导空格（首字节应为 23 21，即 #!）
head -1 /etc/openvpn/vpn-up.sh | xxd | head -1
```

### vpn-down.sh

```bash
sudo tee /etc/openvpn/vpn-down.sh << 'EOF'
#!/bin/bash
if [ -f /run/openvpn-orig-gw.tmp ]; then
    ORIG_GW=$(cat /run/openvpn-orig-gw.tmp)
    ip route replace default via "$ORIG_GW"
    rm -f /run/openvpn-orig-gw.tmp
    echo "[vpn-down] Restored gateway: $ORIG_GW"
fi
EOF

sudo chmod 755 /etc/openvpn/vpn-down.sh
```

---

## 四、Systemd 服务

```bash
sudo tee /etc/systemd/system/openvpn-split.service << 'EOF'
[Unit]
Description=OpenVPN Split Routing Client
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
ExecStart=/usr/sbin/openvpn --config /etc/openvpn/client.ovpn
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now openvpn-split
```

> **注意**：不要在 `ExecStartPre` 中下载 IP 列表，APNIC 国内访问慢会导致启动超时。

---

## 五、DNS 分流（可选）

若需要 DNS 也按国内/境外分流（防止 DNS 污染影响境外域名解析）：

```bash
sudo apt-get install -y dnsmasq

sudo tee /etc/dnsmasq.d/split-dns.conf << 'EOF'
# 国内域名 → 阿里云 DNS（直连）
server=/cn/223.5.5.5
server=/aliyun.com/223.5.5.5
server=/feishu.cn/223.5.5.5
server=/weixin.qq.com/223.5.5.5
server=/qq.com/223.5.5.5
server=/baidu.com/223.5.5.5

# 其余 → 境外无污染 DNS（走 VPN 隧道）
server=1.1.1.1
server=8.8.8.8

listen-address=127.0.0.1
bind-interfaces
EOF

sudo systemctl enable --now dnsmasq

# 锁定 resolv.conf（Ubuntu 默认是 systemd-resolved 的软链接，需先替换）
sudo rm /etc/resolv.conf
echo "nameserver 127.0.0.1" | sudo tee /etc/resolv.conf
sudo chattr +i /etc/resolv.conf

# 禁用 systemd-resolved（避免覆盖 resolv.conf）
sudo systemctl disable --now systemd-resolved
```

---

## 六、常用运维命令

```bash
# 服务管理
sudo systemctl status  openvpn-split
sudo systemctl restart openvpn-split
sudo journalctl -u openvpn-split -f

# 验证路由分流
ip route get 8.8.8.8        # 应显示 dev tun0（境外走 VPN）
ip route get 223.5.5.5      # 应显示 dev eth0（国内直连）
ip route get 100.100.100.200 # 应显示 dev eth0（阿里云元数据直连）

# 连通性测试
curl -s -o /dev/null -w "anthropic: %{remote_ip} HTTP/%{http_code}\n" https://api.anthropic.com
curl -s -o /dev/null -w "github:    %{remote_ip} HTTP/%{http_code}\n" https://github.com
curl -s -o /dev/null -w "feishu:    %{remote_ip} HTTP/%{http_code}\n" https://open.feishu.cn
curl -s -o /dev/null -w "aliyun:    %{remote_ip} HTTP/%{http_code}\n" https://www.aliyun.com

# 手动更新中国 IP 列表
sudo /etc/openvpn/scripts/update-chinaips.sh
sudo systemctl restart openvpn-split
```

---

## 七、踩坑记录（实测）

| 现象 | 根本原因 | 解决方案 |
|------|----------|----------|
| `ExecStartPre` 超时，服务无法启动 | APNIC 在国内下载慢，90s 内无法完成 | 删除 ExecStartPre，改用 cron 定期更新 |
| `could not execute external program` | `tee` heredoc 写入的脚本 shebang 有**前导空格**（`  #!/bin/bash`），kernel execve 失败 | `sed -i '1s/^[[:space:]]*//' vpn-up.sh`；用 `xxd` 确认首字节为 `23 21` |
| AppArmor 阻止脚本执行 | Ubuntu OpenVPN AppArmor profile 只允许 `/etc/openvpn/*.sh`，不允许子目录 | 脚本放 `/etc/openvpn/` 根目录，不放 `scripts/` 子目录 |
| `ifconfig_remote is empty` | 服务端使用 `topology subnet`，此模式不设置该变量（仅 p2p 模式才有） | 改从 `$ifconfig_local` 推算网关：`.x.x.x.1` |
| 路由添加成功但流量仍走 eth0 | 原 eth0 默认路由 metric=0，新加 VPN 路由 metric=10，数值大优先级低 | 先加中国路由，再**删除**旧默认路由，最后加 VPN 默认路由 |
| SSH 断线 | 阿里云 SSH 跳板 IP `100.104.x.x` 在 `100.64.0.0/10`（CGNAT），未加直连 | 在直连段中加入 `100.64.0.0/10` |

---

## 八、网络段直连速查

| 网段 | 用途 | 必须直连原因 |
|------|------|------------|
| `10.0.0.0/8` | 阿里云 VPC 内网 | RDS/Redis/OSS 内网端点 |
| `172.16.0.0/12` | ECS 宿主机网段 | eth0 网关所在段 |
| `192.168.0.0/16` | 标准私有网段 | 本地路由 |
| `100.64.0.0/10` | CGNAT（阿里云跳板/堡垒机）| **SSH 直连，必须加** |
| `100.100.0.0/16` | 阿里云元数据/内网 DNS | ECS 实例信息、RAM 角色 |
| VPN 服务器 IP/32 | VPN 公网地址 | 防隧道路由循环 |

---

*文档版本：v2.0（基于实测修订）· 适用于 Ubuntu 20.04/22.04 LTS · OpenVPN 2.6.x*
