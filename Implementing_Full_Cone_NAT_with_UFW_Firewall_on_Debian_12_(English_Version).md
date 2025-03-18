# Implementing Full Cone NAT (NAT1) on Debian 12 Using UFW Firewall

This requires combining iptables NAT rules with kernel parameter adjustments. Below are the detailed configuration steps:

## 1. Basic Environment Preparation

### 1.1 Enable IP Forwarding

```bash
# Temporary activation
echo 1 > /proc/sys/net/ipv4/ip_forward

# Permanent activation
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
sysctl -p
```
### 1.2 Ensure UFW Allows Traffic Forwarding
```bash
sudo nano /etc/default/ufw
```
Modify the following parameter:
```conf
DEFAULT_FORWARD_POLICY="ACCEPT"
```
## 2. Core Configuration for Full Cone NAT
### 2.1 Modify UFW's NAT Rules File
```bash
sudo nano /etc/ufw/before.rules
```
Add the following rules under the `*nat` section (replace `eth0` with your public interface name):
```conf
*nat
:PREROUTING ACCEPT [0:0]
:POSTROUTING ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]

# Full Cone NAT Rule
-A POSTROUTING -o eth0 -j MASQUERADE -m comment --comment "Full Cone NAT"

# Allow Established Connections to Maintain State
-A POSTROUTING -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT

COMMIT
```
### 2.2 Adjust Kernel Parameters (Optimize UDP Connection Tracking)
```bash
sudo nano /etc/sysctl.d/99-fullcone.conf
```
Add the following content:
```conf
# Enable Full Cone NAT
net.netfilter.nf_conntrack_udp_timeout=30
net.netfilter.nf_conntrack_udp_timeout_stream=180

# Increase Connection Tracking Table Size
net.netfilter.nf_conntrack_max=524288
```
Apply the configuration:
```bash
sysctl -p /etc/sysctl.d/99-fullcone.conf
```
## 3. Open Game-Related Ports (Example for Xbox/PS)
```bash
# Open Basic Game Ports
sudo ufw allow 3074/udp   # Xbox
sudo ufw allow 3478:3480/udp  # PSN
sudo ufw allow 27015:27030/udp  # Steam
sudo ufw allow 3658/udp   # Switch

# Allow Specific Protocols
sudo ufw allow proto udp from any to any port 1024:65535
```
## 4. Apply Configuration
```bash
sudo ufw disable && sudo ufw enable  # Restart UFW
sudo systemctl restart ufw
```
## 5. Verify Full Cone NAT
### 5.1 Check Port Mapping Using `netstat`
```bash
watch -n 1 "conntrack -L | grep -E 'udp.*ASSURED'"
```
Expected output:
```bash
udp      17 29 src=192.168.1.100 dst=1.2.3.4 sport=12345 dport=3074 [ASSURED] ...
```
### 5.2 Online Tool Verification
Visit https://www.whatismynat.com/ or use the network test function on your gaming console to confirm the NAT type is Type 1 (Open).

## 6. Troubleshooting Common Issues
### 6.1 If NAT Type Remains Strict/Moderate
Check Interface Name: Ensure `eth0` is the actual public interface (use `ip a` to verify).

Disable Other Firewalls: Ensure no interference from `nftables` or `firewalld`.

UPnP Conflict: Disable UPnP on your router.

### 6.2 Performance Optimization
```bash
# Increase UDP Buffer Size
echo "net.core.rmem_max=26214400" >> /etc/sysctl.conf
echo "net.core.wmem_max=26214400" >> /etc/sysctl.conf
sysctl -p
```
## Configuration Principle Explanation
### MASQUERADE and Full Cone
#### Traditional `MASQUERADE` implements Symmetric NAT. By adjusting with the `-m conntrack` parameter, it supports Full Cone behavior.

#### `nf_conntrack_udp_timeout`: Controls the retention time of UDP mapping entries, extending it to maintain NAT mappings.

### Port Range Opening
#### Gaming consoles dynamically use high port numbers (1024-65535), so UDP communication across the entire range must be allowed.

With the above configuration, you can achieve the same NAT1 effect as commercial routers, suitable for P2P gaming, BT downloads, and similar scenarios.
