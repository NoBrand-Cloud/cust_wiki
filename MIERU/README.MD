# Mieru 脚本使用教程

作者：FengYuchen

## Overview

沪日专线的 cm 入口有盾，不能使用常见的协议，比如 vless、ss、anytls。所有常见的都不能用。

幸运的是，我们还有一个非 tls 协议：mieru。

本文将介绍如何配置和使用 ike 大佬的 mieru 一键部署脚本，并通过简单的 IP 替代实现上网目的。

## 正文

### 进入 ssh 终端

> 如果你会自己进入，请跳过这一部分

先去 NoBrand 官网，进入相关服务的控制面板，点击详情：

![image-20260729235656303](https://picgo-img.anylocate.cc/20260729235656764.png)

找到移动入口 IP 和 SSH 端口：

![image-20260730002041308](https://picgo-img.anylocate.cc/20260730002041441.png)

之后打开 CMD 或者 PowerShell，输入：

```
ssh root@<你的移动入口 IP> -p <你的 SSH 端口号>
```

> 注意尖括号无需保留。

> 如果连不上，请：关闭你的梯子 & 检查省白名单（把北京市也放进去）。

按照提示，输入 yes 登录：

![image-20260730000455176](https://picgo-img.anylocate.cc/20260730000455316.png)

进入终端界面：

![image-20260730000601395](https://picgo-img.anylocate.cc/20260730000601516.png)

### 使用 mieru 一键脚本

> 脚本由 ike 大佬开发

ctrl + c 复制以下代码，然后在终端界面单击右键粘贴：

```
curl -fsSL https://raw.githubusercontent.com/ike-sh/mieru-OneClick/main/install-mita.sh | sudo bash
```

之后回车，出现以下界面：

![image-20260730000932247](https://picgo-img.anylocate.cc/20260730000932421.png)

敲 1，然后回车，进入以下界面：

![image-20260730001015236](https://picgo-img.anylocate.cc/20260730001015344.png)

直接回车：

![image-20260730001109133](https://picgo-img.anylocate.cc/20260730001109230.png)

继续回车：

![image-20260730001137184](https://picgo-img.anylocate.cc/20260730001137289.png)

继续回车，然后：

![image-20260730001255196](https://picgo-img.anylocate.cc/20260730001255345.png)

把下图红框中的东西复制出来，发到微信或者粘贴到记事本：

![image-20260730001430987](https://picgo-img.anylocate.cc/20260730001431124.png)

### 替换 IP

在上一步得到的节点信息中，IP 是日本方向的出口 IP，我们需要将其替换成移动入口 IP。

返回 NoBrand 控制台，找到移动入口 IP：

![image-20260730001658288](https://picgo-img.anylocate.cc/20260730001658466.png)

把上一步中，server 变量后面的 IP 换成箭头中的这个。于是你得到了可以使用的 mieru 节点信息

### 导入 Clash 

现在是 AI 的时代。把你的节点喂给 AI，然后让 AI 帮你生成可用的 Clash 文件。

### 导入 V2rayN

> 有无大佬写一下，我不用这个软件的

## 关于落地机

如果您需要使用落地机，您有两种选择：

### 1. 链式代理

1. 参照以上教程，部署 mieru
2. 在落地机上部署代理 sing-box 或者 xray，协议随便
3. 把 mieru 节点和落地机节点一起丢给 AI，让 AI 配置

### 2. 落地机部署 mieru

1. ssh 到落地机
2. 在落地机上部署 ike 大佬的脚本
3. ssh 到 ix 的终端，配备端口转发，把可用端口中的某一个，转发到落地机 mieru 配置的端口上

> 关于端口转发，用什么都行，你甚至可以用 python

4. 更改落地机拿到的节点配置，server 改成移动入口 IP，端口改成第 3 步在 ix 上选择的端口
5. 把最终的 mieru 配置发给 AI，让 AI 帮忙改成可用的订阅文件