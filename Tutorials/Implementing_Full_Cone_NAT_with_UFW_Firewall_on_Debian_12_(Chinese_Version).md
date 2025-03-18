# 在 Debian 12 上通过 UFW 防火墙实现 Full Cone NAT (NAT1)

需要结合 iptables 的 NAT 规则和内核参数调整。以下是详细配置步骤：

## 一、基础环境准备

### 1. 启用 IP 转发

```bash
# 临时生效
echo 1 > /proc/sys/net/ipv4/ip_forward

# 永久生效
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
sysctl -p
```
### 2. 确认 UFW 允许流量转发
```bash
sudo nano /etc/default/ufw
```
修改以下参数：
```conf
DEFAULT_FORWARD_POLICY="ACCEPT"
```
## 二、Full Cone NAT 核心配置
### 1. 修改 UFW 的 NAT 规则文件
```bash
sudo nano /etc/ufw/before.rules
```
在 `*nat` 部分添加以下规则（替换 `eth0` 为你的公网接口名）：
```conf
*nat
:PREROUTING ACCEPT [0:0]
:POSTROUTING ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]

# Full Cone NAT 规则
-A POSTROUTING -o eth0 -j MASQUERADE -m comment --comment "Full Cone NAT"

# 允许已建立的连接保持状态
-A POSTROUTING -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT

COMMIT
```
### 2. 调整内核参数（优化 UDP 连接跟踪）
```bash
sudo nano /etc/sysctl.d/99-fullcone.conf
```
添加以下内容：
```conf
# 允许 Full Cone NAT
net.netfilter.nf_conntrack_udp_timeout=30
net.netfilter.nf_conntrack_udp_timeout_stream=180

# 增大连接跟踪表
net.netfilter.nf_conntrack_max=524288
```
生效配置：
```bash
sysctl -p /etc/sysctl.d/99-fullcone.conf
```
## 三、开放游戏相关端口（以 Xbox/PS 为例）
```bash
# 开放基础游戏端口
sudo ufw allow 3074/udp   # Xbox
sudo ufw allow 3478:3480/udp  # PSN
sudo ufw allow 27015:27030/udp  # Steam
sudo ufw allow 3658/udp   # Switch

# 允许特定协议
sudo ufw allow proto udp from any to any port 1024:65535
```
## 四、应用配置
```bash
sudo ufw disable && sudo ufw enable  # 重启 UFW
sudo systemctl restart ufw
```
## 五、验证 Full Cone NAT
### 1. 使用 `netstat` 检查端口映射
```bash
watch -n 1 "conntrack -L | grep -E 'udp.*ASSURED'"
```
应看到类似输出：
```bash
udp      17 29 src=192.168.1.100 dst=1.2.3.4 sport=12345 dport=3074 [ASSURED] ...
```
### 2. 在线工具检测
访问 https://www.whatismynat.com/ 或使用游戏机的网络测试功能，确认 NAT 类型为 Type 1 (Open)。

## 六、常见问题排查
### 1. 如果 NAT 类型仍为 Strict/Moderate
检查接口名称：确认 eth0 是否为实际公网接口（可通过 ip a 查看）。
禁用其他防火墙：确保没有 nftables 或 firewalld 干扰。
UPnP 冲突：关闭路由器上的 UPnP 功能。
### 2. 性能优化
```bash
# 增大 UDP 缓冲区
echo "net.core.rmem_max=26214400" >> /etc/sysctl.conf
echo "net.core.wmem_max=26214400" >> /etc/sysctl.conf
sysctl -p
```
配置原理说明
MASQUERADE 与 Full Cone：

传统 MASQUERADE 实现的是 Symmetric NAT，通过 -m conntrack 参数调整使其支持 Full Cone 行为。
nf_conntrack_udp_timeout：控制 UDP 映射表项的保留时间，延长可维持 NAT 映射。

端口范围开放：游戏主机会动态使用高端口号（1024-65535），需允许 UDP 全范围通信。

通过以上配置，可实现与商业路由器相同的 NAT1 效果，适合 P2P 联机游戏、BT 下载等场景。
