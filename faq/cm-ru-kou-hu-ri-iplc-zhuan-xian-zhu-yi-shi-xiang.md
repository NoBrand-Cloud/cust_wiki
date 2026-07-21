# CM入口沪日IPLC专线注意事项

**Q1: 移动v4可用地区和运营商？**

**移动：全网**

**联通：全网**（山东、河南、北京、湖南不可用）

**电信可用地区：**&#x6D59;江、海南、江西、黑龙江、河南、山东、青海、重庆、山西；上海工作&#x65E5;**（全部不保障可用，仅供参考）**

**Q2: 移动入口支持什么协议？**

**不支持一切常见不合规协议如：**

SS，HTTP，HTTPS，TLS，VLESS，VMESS，TROJAN，TUIC，HYSTERIA等

**支持一切常见企业组网协议如：**

WireGuard，OpenVPN，ZeroTier，Tinc，AnyConnect，FortiClient VPN，IPsec，L2TP，VXLAN，Tailscale等。

**Q3：Mieru一键脚本**

{% embed url="https://github.com/ike-sh/mieru-OneClick" %}

**Q4：当然你也可以用SSH 隧道使用我们CM款机器，具体怎么用自行谷歌**

**Q5： 部分地区联通客户SSH连不上，请开工单处理，我们会给您开VNC然后运行脚本即可使用了。后续管理也是VNC即可。**

**Q6：Mieru脚本生成的配置导入时候可能端口号会错误，请注意检查端口号。**

