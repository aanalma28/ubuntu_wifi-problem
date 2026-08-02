# ubuntu_wifi-problem

## 🚨 Problem Description

After conducting network experiments (such as bridging, routing configurations, and Docker containerization) for a thesis project, the Ubuntu network stack encountered severe routing and configuration corruption. The symptoms included:
- **Firewall/Packet Drop:** Residual `iptables` and Docker rules in the `FORWARD` and `NAT` chains blocked network traffic.
- **DNS Resolution Failure:** Local DNS resolver (`systemd-resolved`) malfunctioned, causing domain lookup failures and a question mark (`?`) status on the Wi-Fi icon despite an active physical connection.
- **IPv6 Routing Conflicts:** Underlying IPv6 protocol mismatches caused silent connection timeouts on external networks.
- **Corrupted Connection Profiles:** Stale Wi-Fi profiles retained conflicting routing metrics and static configurations.

## 💡 Solution

A comprehensive network stack recovery procedure was executed to restore normal connectivity:
- **Complete Firewall Flush:** Wiped all custom rules from `iptables` (Filter, NAT, Mangle) and reset default policies to `ACCEPT`.
- **DNS Re-establishment:** Re-linked and reconfigured `systemd-resolved` using stable public DNS servers (`8.8.8.8` and `1.1.1.1`).
- **IPv6 Mitigation:** Temporarily disabled IPv6 at the kernel level to prevent routing bottlenecks.
- **Profile Clean-up:** Purged corrupted Wi-Fi connection profiles to eliminate hidden metric conflicts.
- **Network Service Refresh:** Restarted `NetworkManager` to reinitialize clean routing tables and interface priorities.


## Phase 1 - Clear IPTables and Stop Docker

Docker and the rest of your thesis scripts often leave intercept rules in the `FORWARD` and `NAT` chains that cause data packets from Wi-Fi to be rejected by the kernel.
1. Stop Docker
```bash
sudo systemctl stop docker
```
2. Flush/clean all firewall rules
```bash
sudo iptables -F
sudo iptables -t nat -F
sudo iptables -t mangle -F
sudo iptables -X
sudo iptables -t nat -X
sudo iptables -t mangle -X
```
3. Restore the default firewall policy to `ACCEPT`
```bash
sudo iptables -P INPUT ACCEPT
sudo iptables -P FORWARD ACCEPT
sudo iptables -P OUTPUT ACCEPT
sudo ip6tables -P INPUT ACCEPT
sudo ip6tables -P FORWARD ACCEPT
sudo ip6tables -P OUTPUT ACCEPT
```
4. Re-ignite Docker
```bash
sudo systemctl start docker
```

## Phase 2 - Reset total DNS and systemd-resolved (possibly causing ? in ubuntu wifi)

Question mark (?) the Ubuntu Wi-Fi icon appears because the Connectivity Check feature fails to detect the internet due to local DNS being damaged or blocked.
1. Turn off the built-in DNS solver service:
```bash
sudo systemctl stop systemd-resolved
```
2. Delete old symlink resolver files:
```bash
sudo rm -f /etc/resolv.conf
```
3. Create a new `resolv.conf` file manually and navigate to stable public DNS (Google & Cloudflare):
```bash
sudo bash -c 'echo "nameserver 8.8.8.8" > /etc/resolv.conf'
sudo bash -c 'echo "nameserver 1.1.1.1" >> /etc/resolv.conf'
```
4. Restart `systemd-resolved`:
```bash
sudo systemctl start systemd-resolved
```

## Phase 3 - Temporarily Turn Off IPv6

IPv6 conflicts often cause Ubuntu laptops to experience a timeout or fail to take a route when connected to a new Wi-Fi network. Temporarily turn off via kernel:
```bash
sudo sysctl -w net.ipv6.conf.all.disable_ipv6=1
sudo sysctl -w net.ipv6.conf.default.disable_ipv6=1
sudo sysctl -w net.ipv6.conf.lo.disable_ipv6=1
```

## Phase 4 - Remove All Old Wi-Fi Profiles (Let's Make New from Empty)

Older Wi-Fi profiles store hidden static metric or IP configurations that keep them stuck.
1. See the list of connection profiles stored on your laptop:
```bash
nmcli connection show
```
2. Delete all Wi-Fi profiles you've ever used (replace them with the profile name or SSID that appears in the list):
```bash
nmcli connection delete "YOUR_WIFI_PROFILE"
```
_Delete all Wi-Fi profiles on the list to make them completely clean)._

## Phase 5 - Restart Total NetworkManager & Check the Results

1. Total reload Ubuntu network manager manager:
```bash
sudo systemctl restart NetworkManager
```
2. Reconnect to your home Wi-Fi via the Ubuntu top panel or via terminal:
```bash
nmcli device wifi connect "YOUR_WIFI_SSID" password "YOUR_WIFI_PASSWORD"
```
3. Double check the routing table to make sure the Wi-Fi path (`wlp3s0`) is correct and there are no other interface interruptions:
```bash
ip route show
```
_(Make sure there is only one default via leading to `wlp3s0`)._
4. Test internet connection directly via terminal:
```bash
ping -c 3 8.8.8.8
```
and test DNS:
```bash
nslookup google.com
```

## Phase 6 - Reactivate IPv6

```bash
sudo sysctl -w net.ipv6.conf.all.disable_ipv6=0
sudo sysctl -w net.ipv6.conf.default.disable_ipv6=0
sudo sysctl -w net.ipv6.conf.lo.disable_ipv6=0
```
