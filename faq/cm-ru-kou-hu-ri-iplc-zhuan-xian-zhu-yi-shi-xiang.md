# 沪日IPLC专线注意事项

**Q1: 移动v4可用地区和运营商？**

**移动：全网**   (四川移动可能有速度问题)

**联通：全网（河北不可用）**

**电信：全网（广西、河北不可用；江苏南京、江苏淮安、江苏扬州高峰期有小概率无速度，江苏盐城极小概率无速度，江苏南通速度有概率高峰期不达标，江苏剩下8个太保暂时没有异常报告。）**

**由于国内网络环境特殊，不保障可用性！具体怎么样自己测试**

**Tips：部分省月底省外流量超支了，所以部分省会在月底故意QOS省外流量，造成访问限速。**

### **具体是否可用，您可以下单最小款SHAIX-JP-1-CM试用一段时间，19.8一个月，如果可用，开工单退款到余额升级其他款式。**

**Q2: 移动入口支持什么协议？**

**不支持一切常见不合规协议如：**

SS，HTTP，HTTPS，TLS，VLESS，VMESS，TROJAN，TUIC，HYSTERIA等

**支持一切常见企业组网协议如：**

Mieru，Snell（不完全），SSH隧道，WireGuard，OpenVPN，ZeroTier，Tinc，AnyConnect，FortiClient VPN，IPsec，L2TP，VXLAN，Tailscale等。

**Q3：Mieru一键脚本**

{% embed url="https://github.com/ike-sh/mieru-OneClick" %}

**Q4：当然你也可以用SSH 隧道使用我们CM款机器，具体怎么用自行谷歌**

**Q5：如果SSH连接不上，可能是你没有省白，您在省白添加北京地区即可使用。**

如果添加后仍不行且您是联通，请开工单处理，我们会给您开VNC然后运行脚本即可使用了，后续管理也是VNC即可，您的运营商ban了ssh。

**Q6：Mieru脚本生成的配置导入时候可能端口号会错误，请注意检查端口号。**

### **Q7：IPv6出口有点不好用，访问网站慢，如何关闭IPv6？**

```
cat >/etc/sysctl.d/99-disable-ipv6.conf <<'EOF'
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1
EOF
sysctl --system
```

输入上述指令后重启您的机器即可。

**Q8：SSH隧道？**

1. 新创建一个SSH密钥对, 和你登录用的不一样,为了与正常登录用户隔离

> [!IMPORTANT]
>
>请一定要自己生成SSH密钥对，不要直接使用下面的密钥

```
ssh-keygen -t ed25519 -C "your_email@example.com"
```
然后一路回车
再执行`cat ~/.ssh/id_ed25519`查看私钥，执行`cat ~/.ssh/id_ed25519.pub`查看公钥
<img width="1287" height="947" alt="image" src="https://github.com/user-attachments/assets/68e6993f-bfca-4b57-9a09-ae71e488356d" />

2. 在机器上上创建SSH隧道专用用户 `tunnel`

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

5. 获取 host-key

```
cat /etc/ssh/ssh_host_ed25519_key.pub
```

只使用`ssh-ed25519 xxxxxxx`这一部分，后面的`root@nbnet-3608`不用复制
示例：<img width="1087" height="132" alt="image" src="https://github.com/user-attachments/assets/d3a4b787-4918-488b-9c2d-6aef0af2f905" />

6. 拼接配置
以 mihomo 为例，建议把你生成的SSH密钥和下面的mihomo配置格式一起丢给AI让他给你拼接
```
proxies:
  - name: "JP-NoBrand-IPLC"
    type: ssh
    server: 你的移动入口IP
    port: 输入你的SSH端口
    username: tunnel  #如果没修改过就不用改
    # 下面输入你的私钥，需要注意有缩进，建议让AI帮你替换
    private-key: |
      -----BEGIN OPENSSH PRIVATE KEY-----
      b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAAAMwAAAAtzc2gtZW
      QyNTUxOQAAACBXkiLbL/fu4tY+BbzB1Q5qstjL7qxut/mO/tVDtqzvWwAAAKDDtRJ7w7US
      ewAAAAtzc2gtZWQyNTUxOQAAACBXkiLbL/fu4tY+BbzB1Q5qstjL7qxut/mO/tVDtqzvWw
      AAAED9zXnbYdHQxx50KrtvlFNoRScPWnrC3luTcCJcksjUQVeSItsv9+7i1j4FvMHVDmqy
      2MvurG63+Y7+1UO2rO9bAAAAF292ZXJuaWdodG5la29AZ21haWwuY29tAQIDBAUG
      -----END OPENSSH PRIVATE KEY-----
    #输入前面获取的host-key
    host-key: 
      - "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIEdiXqZEg/114514hRQdPYTMe/67abcd"
    host-key-algorithms: 
      - ssh-ed25519
```
