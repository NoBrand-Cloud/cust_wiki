## 方案概述
Wireguard: 在主机间建立L3隧道，打通各个主机间的通信。

OSPF: 建立内网路由，打通内网间任意两点的连接。且确保在某条线路出现故障时，流量能够自动切换到其他可用的线路。

操作系统：这里假定你使用的都是debian系的linux系统，其他系统的配置方法可能会有所不同，但整体思路是类似的。
## 实现步骤
### 1. 规划网络
这里我们使用2台服务器作为示例，首先我们需要选择一个私有IP地址段来为我们的内网分配IP地址，这里我们选择`172.20.177.0/26`作为我们的内网地址段，这个地址段可以提供62个主机地址，足够我们使用，这里我们的地址分配如下
| 服务器 | 内网IP地址 |
| --- | --- |
| A | 172.20.177.2 |
| B | 172.20.177.3 |

上面表格中的地址就是我们今后在内网中访问这两台服务器时使用的地址，接下来我们需要将它们配置到对应服务器上的loopback接口上，让本机能够识别这个地址。
### 2. 配置Loopback接口
如果你的服务器使用netplan进行网络配置，你可以按照以下步骤来配置Loopback接口：
1. 打开终端，使用文本编辑器（如nano或vim）编辑或创建netplan配置文件，通常位于`/etc/netplan/`目录下，文件名使用`01-netcfg.yaml`或类似的名称(一般需要你自行创建)。
2. 在配置文件中添加Loopback接口的配置，示例如下，注意下面的172.20.177.2需要替换为对应的内网IP地址：
```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    lo:
      addresses:
        - 127.0.0.1/8
        - 172.20.177.2/32
```
3. 保存文件并退出编辑器，然后执行`sudo netplan apply`命令来应用新的网络配置。
4. 此时，你可以使用`ip addr show lo`命令来验证Loopback接口是否正确配置了内网IP地址。
### 3. 安装和配置WireGuard
接下来需要在每台服务器上安装WireGuard，并配置它们之间的连接，在两台服务器上执行以下命令来安装WireGuard：
```bash
sudo apt update
sudo apt install wireguard -y
```
这里我们使用wg quick工具来简化配置，首先在A上切换到`/etc/wireguard/`目录下，创建一个名为`wg-inetB.conf`的配置文件，内容如下:
```ini
[Interface]
Address = 10.0.0.1/30
Address = fe80::1/64
MTU = 1420
Table = off
PrivateKey = <A端私钥>
ListenPort = <A端监听端口>

[Peer]
PublicKey = <B端公钥>
AllowedIPs = 0.0.0.0/0, ::/0
Endpoint = <B端Endpoint>
```
同样的，在B上也创建一个名为`wg-inetA.conf`的配置文件，内容如下:
```ini
[Interface]
Address = 10.0.0.2/30
Address = fe80::2/64
MTU = 1420
Table = off
PrivateKey = <B端私钥>
ListenPort = <B端监听端口>

[Peer]
PublicKey = <A端公钥>
AllowedIPs = 0.0.0.0/0, ::/0
Endpoint = <A端Endpoint>
```
解释一下上面的Address字段，这里配置的10.0.0.1/30和10.0.0.2/30是WireGuard接口的IP地址，这些地址用于WireGuard隧道内的通信，而不是我们之前规划的内网地址。你可以根据需要选择其他的私有IP地址段来作为WireGuard接口的地址，确保它们在你的网络中是唯一的即可。fe80::1/64和fe80::2/64是IPv6的链路本地地址，这些地址在同一链路上是唯一的，可以用于IPv6通信。MTU设置为1420是在默认的1500上减去80的wireguard开销，如果你的网络环境有特殊要求，可以相应调整。Table设置为off表示不使用默认的路由表，而是由OSPF来管理路由。PrivateKey和PublicKey需要替换为你生成的密钥对，ListenPort需要选择一个未被占用的端口，Endpoint需要填写对方服务器的公网IP地址和监听端口，例如`wg.example.com:51820`。

关于密钥生成，这里提供一个简单的脚本来生成密钥对，你可以在服务器上执行以下命令来生成两端的私钥和公钥：
```bash
#!/bin/bash
# 快速生成双端密钥对
key1_priv=$(wg genkey)
key1_pub=$(echo "$key1_priv" | wg pubkey)
key2_priv=$(wg genkey)
key2_pub=$(echo "$key2_priv" | wg pubkey)

echo "=== Peer A (Server 1) ==="
echo "PrivateKey = $key1_priv"
echo "PublicKey  = $key1_pub"
echo ""
echo "=== Peer B (Server 2) ==="
echo "PrivateKey = $key2_priv"
echo "PublicKey  = $key2_pub"
echo ""
echo "--- 提示: 配置时，A 的 Peer 栏填 B 的 PublicKey，反之亦然 ---"
```
执行这个脚本后，你会得到两端的私钥和公钥，记得将A端的私钥填入A的配置文件中，B端的公钥填入A的Peer配置中，反之亦然。

保存配置文件后，在两台服务器上分别执行以下命令来开机启动WireGuard接口：

A端：
```bash
sudo systemctl enable wg-quick@wg-inetA
sudo systemctl start wg-quick@wg-inetA
```
B端：
```bash
sudo systemctl enable wg-quick@wg-inetB
sudo systemctl start wg-quick@wg-inetB
```
接下来尝试在A端执行`ping 10.0.0.2`来测试WireGuard隧道的连通性。如果一切配置正确，你应该能够成功ping通对方的WireGuard接口地址。
### 4. 配置OSPF
进行完上面的步骤，A和B之间的点对点连接就已经建立好了，但此时它们之间还没有路由信息，无法实现内网地址的互通。为了让它们能够互相访问对方的内网地址，我们需要配置OSPF协议来动态路由流量。在Linux系统中，我们可以使用Quagga或FRRouting来实现OSPF协议，这里我们选择bird2来配置OSPF，首先在两台服务器上安装bird2：
```bash
sudo apt update && sudo apt -y install apt-transport-https ca-certificates wget lsb-release
sudo wget -O /usr/share/keyrings/cznic-labs-pkg.gpg https://pkg.labs.nic.cz/gpg

echo "deb [signed-by=/usr/share/keyrings/cznic-labs-pkg.gpg] https://pkg.labs.nic.cz/bird2 $(lsb_release -sc) main" | sudo tee /etc/apt/sources.list.d/cznic-labs-bird2.list

sudo apt update && sudo apt install bird2
```
执行`bird --version`你应当能够看到
```
BIRD 2.18.1
```
接下来配置bird以启用OSPF协议，在A上进入`/etc/bird/`目录，创建一个名为`bird.conf`的配置文件(原有的你可以覆盖)，内容如下:
```javascript
################################################
#               Variable header                #
################################################

define OWNIP =  172.20.177.2;
define OWNNET = 172.20.177.0/26;
define OWNNETSET = [172.20.177.0/26+];

################################################
#                 Header end                   #
################################################

router id OWNIP;

protocol device {
    scan time 10;
}

function is_self_net() -> bool {
  return net ~ OWNNETSET;
}

filter ospf_export {
    if is_self_net() then accept;
    reject;
}

protocol direct {
    ipv4;
    interface "lo";
}

protocol kernel {
    scan time 20;

    ipv4 {
        import all;
        export filter {
            if source = RTS_STATIC then reject;
            krt_prefsrc = OWNIP;
            accept;
        };
    };
}

protocol ospf v2 internal_ospf {
    ipv4 {
        import all;
        export filter ospf_export;
    };
    area 0 {
        interface "wg-inet*" {
            type ptp;
            cost 10;
            hello 5;
        };
    };
}
```
保存文件后，在A端执行以下命令来启动bird服务：
```bash
bird configure
```
同样的，在B端也创建一个名为`bird.conf`的配置文件，内容如下:
```javascript
################################################
#               Variable header                #
################################################

define OWNIP =  172.20.177.3;
define OWNNET = 172.20.177.0/26;
define OWNNETSET = [172.20.177.0/26+];

################################################
#                 Header end                   #
################################################

router id OWNIP;

protocol device {
    scan time 10;
}

function is_self_net() -> bool {
  return net ~ OWNNETSET;
}

filter ospf_export {
    if is_self_net() then accept;
    reject;
}

protocol direct {
    ipv4;
    interface "lo";
}

protocol kernel {
    scan time 20;

    ipv4 {
        import all;
        export filter {
            if source = RTS_STATIC then reject;
            krt_prefsrc = OWNIP;
            accept;
        };
    };
}

protocol ospf v2 internal_ospf {
    ipv4 {
        import all;
        export filter ospf_export;
    };
    area 0 {
        interface "wg-inet*" {
            type ptp;
            cost 10;
            hello 5;
        };
    };
}
```
保存文件后，在B端执行以下命令来启动bird服务：
```bash
bird configure
```
过一会在A上执行`birdc show ospf neighbor`，预期应该能看到：
```
BIRD 2.18.1 ready.
internal_ospf:
Router ID       Pri          State      DTime   Interface  Router IP
172.20.177.3    1            Full/PtP   39.727  wg-inetB   10.0.0.2
```
此时A和B之间的OSPF邻居关系已经建立好了，你可以在A上执行`ping 172.20.177.3`来测试内网地址的连通性，如果一切配置正确，你应该能够成功ping通对方的内网地址。

今后如果你有更多的服务器需要加入这个内网，只需要按照上面的步骤在新服务器上配置WireGuard和OSPF，并将它们连接到已有的服务器上即可，**注意上方bird配置中的`interface "wg-inet*"`表示的是匹配wg-inet开头的接口，所以后续建立连接的wireguard接口也需要以wg-inet开头**，OSPF会自动学习新的路由信息。
## 总结
通过以上步骤，我们已经成功地建立了一个自己的内网，后续我们可以随意添加新的服务器加入自己的内网，在内网中为自己提供安全的服务访问，同时也可以通过OSPF协议实现多条线路间的容灾，确保在某条线路出现故障时，流量能够自动切换到其他可用的线路。希望这个方案能够帮助你更好地管理和使用你的服务器资源。

以上教程仅为笔者作为初学者的实践总结，可能存在一些不完善的地方，如果你有更好的建议或者发现了什么问题，欢迎留言讨论。
