# CM 专线搭配 Surge 客户端食用指南

> SSH 协议与 Snell over SSH

## 一、前言

相比于主流的客户端，Surge 自身可用的协议较少，且无法使用 Mieru 协议。因此推荐 Surge 原生的 Snell 协议进行部署。

虽然 CM 入口未明确禁止 Snell，但 Surge 在使用 Snell v5/v6 时仍存在兼容性问题，导致首包断流或无回包的情况。

针对这一弊端，本文采用 **SSH 协议作为底层跳板，为 Snell 套上 SSH 外壳（Snell over SSH）**，以此绕过协议兼容问题，获得稳定、高吞吐的体验。

---

## 二、前置准备：获取 SSH Key 并转换为 Base64

登录 NoBrand CM 专线服务器须依赖 SSH 密钥认证。

### 1. 找到你的私钥文件
* 若此前在本地生成过密钥对，默认位于用户目录的 `.ssh` 文件夹中（如 `id_ed25519` 或 `id_rsa`，注意**不要**选带 `.pub` 后缀的公钥）；
* 若不在默认路径，请确保私钥存放在你本地，并明确存放路径，用于接下来的操作。

### 2. 转换为 Base64 纯文本
根据您的操作系统，在终端中执行以下命令，终端会**直接完整打印出 Base64 文本**：

#### Windows（PowerShell 终端执行）
```powershell
[Convert]::ToBase64String([IO.File]::ReadAllBytes("$HOME\.ssh\id_ed25519"))
```
> 💡 **注意**：
> 若私钥存放在自定义路径，直接替换双引号内的路径即可（例如 `"$HOME\Downloads\your_key.pem"`）。

#### macOS / Linux 终端执行

* **macOS**：

  ```bash
  cat ~/.ssh/id_ed25519 | base64
  ```

* **Linux**：

  ```bash
  base64 -w 0 ~/.ssh/id_ed25519
  ```

执行完毕后，选中输出的整段完整 Base64 字符串并复制，留作下一步使用。

> 💡 **输出格式参考与自检方法**：
> * **头部特征**：由于 OpenSSH 私钥均以 `-----BEGIN ...` 开头，编码后前缀必然固定为 **`LS0tLS1CRUdJTi`**，可凭此快速确认是否成功提取了私钥；
> * **格式要求**：整段文本为**单行连续输出，中间绝无任何空格或换行**（字符仅由大小写字母、数字、`+`、`/` 组成，末尾可能包含 `=`）。

**Base64 格式参考示例**：

```text
LS0tLS1CRUdJTiBPUEVOU1NIIFBSSVZBVEUgS0VZLS0tLS0KYjNCbGJu......RU5EIE9QRU5TU0ggUFJJVkFURSBLRVktLS0tLQo=
```

---

## 三、手动修改 Surge 配置

推荐在服务端使用 NoBrand-OneClick 脚本安装好 Snell 服务后，打开 Surge 文本配置文件，逐个添加以下必要字段：

### 1. 添加 [Keystore] 字段（私钥凭据）

在配置文件中找到或新建 `[Keystore]` 模块，填入转换好的 Base64 文本：

```ini
[Keystore]
SSH-Key = type=openssh-private-key, base64=LS0tLS1CRUdJTiBPUEVOU1NIIFBSSVZBVEUgS0VZLS0tLS0KYjNCbGJu......=
```
> 💡 **参数注释**：
> * `SSH-Key`：为该私钥凭据自定义的名称，供后续代理节点引用。
> * `type=openssh-private-key`：声明私钥格式类型。
> * `base64=`：粘贴上一步转换得到的整段单行 Base64 字符串。若私钥设置了密码保护，可在尾部追加 `, passphrase="你的密码"`。

---

### 2. 添加 [Proxy] 基础 SSH 跳板节点

在 `[Proxy]` 区域定义基础 SSH 连接，填入供应商分配的移动入口 IP 与 SSH 外网映射端口：

```ini
[Proxy]
SSH-CMIX = ssh, 211.136.x.x, 7400, username=root, private-key=SSH-Key, tfo=true
```
> 💡 **参数注释**：
> * `SSH-CMIX`：底层 SSH 隧道名称。
> * `211.136.x.x, 7400`：NoBrand 控制面板分配给您的 **外网移动入口 IP** 与 **SSH 映射端口**（请根据实际分配信息填写）。
> * `username=root`：登录服务器的用户名。（通常为 root）
> * `private-key=SSH-Key`：绑定上一步在 `[Keystore]` 中定义的私钥名称。
> * `tfo=true`：（可选参数）开启 TCP Fast Open（TCP 快速打开）。在网络环境及服务端系统内核支持时，可进一步减少握手 RTT 往返时延。
> 
> *注：至此该节点已经可以直接使用了，但本教程内仅作为底层跳板隧道。*

---

### 3. 添加 [Proxy] Snell over SSH 链式节点

配置真正用于代理上网的 Snell 节点，使用 `underlying-proxy` 字段嵌套绑定到上一层 SSH：

```ini
CMIX-Snell = snell, 127.0.0.1, 7490, psk=Your_Snell_PSK_Secret, version=6, underlying-proxy=SSH-CMIX, reuse=true, mode=unsafe-raw
```
> 💡 **参数注释**：
> * **`127.0.0.1`**：**必须填写 `127.0.0.1`**。这里的回环地址并非客户端本地电脑，而是指 **SSH 服务端本机的本地回环**。流量先通过 SSH 隧道加密送达服务器，再由服务器内部交付给本地监听的 Snell。
> * **`7490`**：填入服务器上 Snell 实际监听的端口。
> * **`psk=`**：填入服务端部署 Snell 时生成的通信密钥。
> * **`underlying-proxy=SSH-CMIX`**：核心参数。指示 Surge 先连通基础 SSH 隧道，并将 Snell 流量封装在 SSH 隧道内部传输。
> * **`version=6`**：Snell 协议版本（建议使用 v6；若需要兼顾其他不支持 v6 的客户端，服务端亦可部署 v5 并在此填写 `5`）。
> * **`reuse=true`**：开启 TCP 连接复用，降低移动网络首包握手延迟。
> * **`mode=`**：（仅 Snell v6 支持，根据需求二选一）：
>   * **`unsafe-raw`（推荐）**：**完全关闭 Snell 内层加密**。由于外层 SSH 已经提供了高强度加密，内层关闭加密能彻底杜绝“双重加密”带来的 CPU 算力浪费，释放最大带宽吞吐。
>   * **`unshaped`**：**保留 Snell 内层加密，但关闭数据包混淆整形（Traffic Shaping）**。数据包不再填充随机长度的冗余填充物，既减少了无效的流量开销并降低延迟，又维持了内层独立加密。

---

### 4. 添加 [Proxy Group] 策略组引用

最后，将应用层代理节点 `CMIX-Snell` 引入您的选择策略组中：

```ini
[Proxy Group]
Proxy = select, CMIX-Snell, DIRECT
```
> 保存并重载 Surge 配置，即可在策略组中选中该节点正常接入。

---

### 5. 完整配置示例预览

将上述分散模块整合后，Surge 配置文件中的对应完整片段预览如下：

```ini
[Keystore]
# 1. SSH 私钥凭据（替换为转换好的 Base64 文本）
SSH-Key = type=openssh-private-key, base64=LS0tLS1CRUdJTiBPUEVOU1NIIFBSSVZBVEUgS0VZLS0tLS0KYjNCbGJu......=

[Proxy]
# 2. 基础 SSH 隧道节点（底层跳板，tfo=true 为可选加速参数）
SSH-CMIX = ssh, 211.136.x.x, 7400, username=root, private-key=SSH-Key, tfo=true

# 3. Snell over SSH 节点（实际代理上网节点，mode 可选 unsafe-raw 或 unshaped）
CMIX-Snell = snell, 127.0.0.1, 7490, psk=Your_Snell_PSK_Secret, version=6, underlying-proxy=SSH-CMIX, reuse=true, mode=unsafe-raw

[Proxy Group]
# 4. 策略组引用
Proxy = select, CMIX-Snell, DIRECT
```

> 💡 **说明**：同理，借助 SSH 外壳作为底层跳板，内部亦可运行任何 Surge 支持的代理协议，大家可按需自由组合，本文不做额外推荐，除本文推荐外的协议不保证可用。

---

## 四、落地部署方案

如果该 CM 专线节点仅作为前置中转机接入，目标流量需要进一步转发至境外落地机（如香港、新加坡等海外 VPS）：

1. **在目标 VPS 部署落地协议**：部署 Surge 支持的协议（如 Snell，推荐使用 NoBrand-OneClick 脚本），并记录其公网 IP、监听端口及 PSK 密钥。
2. **在专线 VPS 配置端口转发**：利用脚本搭建端口转发将一个新分配的端口（如 `7491`）转发至目标落地的 IP 与端口。
3. **在 Surge 中加入落地节点**：在配置文件中新增节点，端口填入本地转发端口（`7491`）并填入落地机节点密钥，继续通过 `underlying-proxy=SSH-CMIX` 复用同一个底层 SSH 隧道。

```ini
[Proxy]
# 境外落地转发节点（复用现有底层 SSH-CMIX 隧道）
HK-Forward-Snell = snell, 127.0.0.1, 7491, psk=Your_Landing_Snell_PSK, version=6, underlying-proxy=SSH-CMIX, reuse=true, mode=unsafe-raw

[Proxy Group]
Proxy = select, CMIX-Snell, HK-Forward-Snell, DIRECT
```

---

## 五、常见问题

| 常见问题 / 报错                              | 根本原因                           | 解决方法                                                                                                                                                         |
| :------------------------------------- | :----------------------------- | :----------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Keystore decode error`                | Base64 字符串中夹杂了多余换行符、空格，或误复制了公钥 | 使用本文提供的单行命令重新生成，确保输出为整段连续无空格换行的文本，且确认转换的是私钥而非 `.pub` 公钥。                                                                                                     |
| `SSH Connection Timeout`               | 入口 IP 或 SSH 端口填写有误             | 检查 `SSH-CMIX` 节点填写的 IP 是否为面板明确标注的**移动入口 IP**，并确认端口是否在**分配给您的有效端口区间**内。                                                                                      |
| `Permission denied (publickey)` / 认证失败 | 本地私钥不匹配或配置引用错误                 | 1. 先在本地终端通过常规 SSH 命令确认该私钥能够正常登录服务器；<br>2. 检查 Surge 中 `[Keystore]` 内转换的私钥 Base64 是否完整无误；<br>3. 检查 `[Proxy]` 中 `private-key=` 引用的名称与 `[Keystore]` 定义的名称是否完全一致。 |
