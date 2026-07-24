# 沪日IPLC专线注意事项

**Q1: 移动v4可用地区和运营商？**

**移动：全网**

**联通：全网**

**电信：全网（广西、河北、江苏南京、江苏淮安、江苏扬州不可用）**

**由于国内网络环境特殊，不保障跨网可用性！具体怎么样自己测试**

### **具体是否可用，您可以下单最小款SHAIX-JP-1-CM试用一段时间，19.8一个月，如果可用，开工单退款到余额升级其他款式。**

**Q2: 移动入口支持什么协议？**

**不支持一切常见不合规协议如：**

SS，HTTP，HTTPS，TLS，VLESS，VMESS，TROJAN，TUIC，HYSTERIA等

**支持一切常见企业组网协议如：**

WireGuard，OpenVPN，ZeroTier，Tinc，AnyConnect，FortiClient VPN，IPsec，L2TP，VXLAN，Tailscale等。

**Q3：Mieru一键脚本**

{% embed url="https://github.com/ike-sh/mieru-OneClick" %}

**Q4：当然你也可以用SSH 隧道使用我们CM款机器，具体怎么用自行谷歌**

**Q5：如果SSH连接不上，可能是你没有省白，您在省白添加北京地区即可使用。**

如果添加后仍不行且您是联通，请开工单处理，我们会给您开VNC然后运行脚本即可使用了，后续管理也是VNC即可，您的运营商ban了ssh。

**Q6：Mieru脚本生成的配置导入时候可能端口号会错误，请注意检查端口号。**

**Q7：如何关闭IPv6？**

```
cat >/etc/sysctl.d/99-disable-ipv6.conf <<'EOF'
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1
EOF
sysctl -p
```

输入上述指令后重启您的机器即可。

**Q8：SSH隧道？**

1. 新创建一个SSH密钥对, 和你登录用的不一样
2. 在机器上上创建专用用户 tunnel

```
useradd -m -s /usr/sbin/nologin tunnel
mkdir /home/tunnel/.ssh && echo "你的公钥" > /home/tunnel/.ssh/authorized_keys
```

3. 管理 tunnel 用户 ssh 权限

```
cat << 'EOF' | sudo tee /etc/ssh/sshd_config.d/00-tunnel.conf > /dev/null
 Match User tunnel
    AllowTcpForwarding yes
    X11Forwarding no
    PermitTunnel no
    AllowAgentForwarding no
    ForceCommand /usr/sbin/nologin
EOF
```



4. 重启 ssh

`sudo systemctl restart ssh`

