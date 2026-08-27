### 1. 创建专用独立用户并配置密钥
为了安全，我们不使用 root 直接建隧道，而是从头创建一个专属的 proxyuser 并为其配置密钥。
```Bash
useradd -m -s /bin/bash proxyuser
su - proxyuser
ssh-keygen -t ed25519 -f ~/.ssh/mihomo_proxy
cat ~/.ssh/mihomo_proxy.pub >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
chmod 700 ~/.ssh
### 务必保存下来，客户端 private-key 字段要用
cat ~/.ssh/mihomo_proxy
exit
### 保存下来，客户端 host-key 字段要用
cat /etc/ssh/ssh_host_ed25519_key.pub | awk '{print $2}'
```
### 2. 开启 SSH TCP 转发功能
编辑 /etc/ssh/sshd_config，确保以下两项处于开启状态：
```Plaintext
AllowTcpForwarding yes
PubkeyAuthentication yes
```
重启 SSH 服务应用配置：
`systemctl restart sshd`

### 3. 安装与配置 Mihomo 服务端
```Bash
wget https://github.com/MetaCubeX/mihomo/releases/download/v1.18.2/mihomo-linux-amd64-v1.18.2.gz
gunzip mihomo-linux-amd64-v1.18.2.gz
mv mihomo-linux-amd64-v1.18.2 /usr/local/bin/mihomo
chmod +x /usr/local/bin/mihomo
mkdir -p /etc/mihomo
```
写入配置文件
```Bash
cat << 'EOF' > /etc/mihomo/config.yaml
mode: rule
log-level: info

listeners:
  - name: "ss"
    type: shadowsocks   
    listen: 127.0.0.1
    port: 5201
    cipher: none
    password: "123456"  
    udp: true

rules:
  - MATCH,DIRECT
EOF
```
配置 Systemd 守护进程并启动：

```Bash
cat << 'EOF' > /etc/systemd/system/mihomo.service
[Unit]
Description=Mihomo Server Proxy
After=network.target

[Service]
Type=simple
LimitNOFILE=65536
ExecStart=/usr/local/bin/mihomo -d /etc/mihomo
Restart=on-failure
RestartSec=3s

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable --now mihomo
```
### 4.客户端配置 (本地 Mihomo)
在客户端的 proxies 列表中添加以下两个节点，需要修改的是移动入口，SSH端口，上面输出的两个字段：private-key和host-key。
```YAML
proxies:
  - name: SSH-Tunnel
    type: ssh
    server: <移动入口IP>     # CM入口IP
    port: 8888               # CM入口分配的监听端口
    username: proxyuser      # 刚刚创建的专用用户
    private-key: |-
      -----BEGIN OPENSSH PRIVATE KEY-----
      # 粘贴服务端生成的 ~/.ssh/mihomo_proxy 内容
      -----END OPENSSH PRIVATE KEY-----
    host-key:
      - "ssh-ed25519 <粘贴服务端获取的 Host Key 元数据>" 
    host-key-algorithms:
      - ssh-ed25519

  - name: "IPLC"
    type: ss
    server: 127.0.0.1
    port: 5201
    cipher: none
    password: "123456"       
    udp: true
    udp-over-tcp: true
    udp-over-tcp-version: 2
    dialer-proxy: SSH-Tunnel
```
后续分流规则里面，只需要添加第二个IPLC这个名称进去就好，也可以自己命名添加对应国旗例如："🇯🇵 NoBrand IPLC"。如果有相关问题配合现在AI应该能轻松扫荡~