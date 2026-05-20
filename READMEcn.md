# macOS 上 Mihomo Core 的 DNS/TUN 配置教程

本指南介绍如何在 macOS 上配置 Mihomo 的 TUN 模式和 DNS 解析器，以防止 DNS 泄漏，并让 Apple 服务走直连（DIRECT）规则，从而避免相关功能异常。

该配置将**在系统范围内拦截所有 DNS 请求（无任何例外）**，并使用你提供的 DNS 服务器来解析 DIRECT 规则匹配到的域名。其余所有流量都会交由代理服务器处理。此配置同样适用于命令行工具（如 Homebrew、SSH）以及 Wine，无需额外设置。

推荐通过 [Homebrew](https://brew.sh) 来安装和管理 Mihomo，并使用其默认配置目录。如果你尚未安装 Mihomo（前提是 Homebrew 已可正常工作）：

```bash
brew install mihomo
mkdir ~/.config/mihomo
touch ~/.config/mihomo/config.yaml
```

## 注意事项

1. **不建议在此配置下使用图形界面客户端（GUI）**，因为这些客户端往往会覆盖 Mihomo 的 DNS/TUN 配置块，从而导致冲突。此方案旨在直接使用 Mihomo Core，或配合 [Zashboard](https://github.com/Zephyruso/zashboard?utm_source=chatgpt.com) Web UI 使用。你可以通过以下方式部署 Zashboard：

```bash
mkdir ~/.config/mihomo/ui && cd ~/.config/mihomo/ui
curl -L https://github.com/Zephyruso/zashboard/releases/latest/download/dist-no-fonts.zip -o zash.zip
unzip zash.zip && mv dist/* . && rm -rf dist zash.zip
```

然后在你的 `config.yaml` 中添加：

```yaml
external-controller: 127.0.0.1:9090
external-ui: /Users/YOUR-USERNAME/.config/mihomo/ui
```

完成配置并启动 Mihomo 后，即可通过浏览器访问：

`http://127.0.0.1:9090/ui`

2. **你的代理服务器必须自行配置 DNS**，因为本方案依赖远程解析（remote resolution）。

3. **这只是一个路由模板，不包含任何代理服务器配置。** 代理节点需要由用户自行提供。更多信息请参考 [Mihomo Wiki](https://wiki.metacubex.one)。

以下所有内容都应写入：

`~/.config/mihomo/config.yaml`

---

# 配置步骤

## 1. 设置 Rule Providers（规则提供器）

以下规则集来自 [blackmatrix7 的 ios_rule_script](https://github.com/blackmatrix7/ios_rule_script)，可直接使用，无需修改：

```yaml
rule-providers:
  Apple:
    type: http
    path: "./rules/apple.yaml"
    url: "https://raw.githubusercontent.com/blackmatrix7/ios_rule_script/refs/heads/master/rule/Clash/Apple/Apple_Classical_No_Resolve.yaml"
    interval: 86400
    behavior: classical
    format: yaml

  LAN:
    type: http
    path: "./rules/lan.yaml"
    url: "https://raw.githubusercontent.com/blackmatrix7/ios_rule_script/refs/heads/master/rule/Clash/Lan/Lan_No_Resolve.yaml"
    interval: 86400
    behavior: classical
    format: yaml
```

然后在规则部分添加：

```yaml
- DOMAIN,raw.githubusercontent.com,DIRECT
- RULE-SET,LAN,DIRECT
- RULE-SET,Apple,DIRECT
```

---

## 2. 添加 DNS 配置块

```yaml
dns:
  enable: true
  respect-rules: false
  listen: '127.0.0.1:53'
  enhanced-mode: fake-ip
  fake-ip-range: 198.18.0.0/16
  fake-ip-filter-mode: blacklist
  fake-ip-filter:
    - rule-set: LAN
    - rule-set: Apple
  ipv6: false
  default-nameserver:
    - 223.5.5.5
  nameserver:
    - https://doh.pub/dns-query
  fallback: []
```

此模板中最关键的部分包括：

* **监听地址 `127.0.0.1:53`**
* 使用 **`fake-ip` 模式**
* 基于规则集的黑名单（`fake-ip-filter`）
* **`respect-rules: false`**

工作原理如下：

Mihomo 会对**不匹配黑名单规则集**的域名返回一个虚拟 IP（Fake IP），从而避免在本地进行真实 DNS 查询，因此不会发生 DNS 泄漏。

由于我们已经使用了 `fake-ip` 模式，因此**无需启用 `respect-rules`**。若将其设置为 `true`，只会产生额外的 DNS 查询，并可能引发 bootstrap 死锁问题。

如果你希望某些自定义规则集或特定域名返回真实 IP，可以将它们加入 `fake-ip-filter` 中。但请注意：

> **任何被加入黑名单的域名都会在本地进行真实 DNS 解析。**

你可以将阿里云 DNS（223.5.5.5）替换为其他 DNS，也可以配置 fallback。但通常没有必要，因为本方案的设计目标是让代理流量严格使用远程 DNS 解析，你无需担心中国大陆网络环境中对境外网站的 DNS 污染问题。

此外，`direct-nameserver` 和 `proxy-server-nameserver` 也没有必要配置，因为 `nameserver` 已经完全取代它们的作用。

---

## 3. 添加 TUN 配置块

```yaml
tun:
  enable: true
  stack: system
  dns-hijack:
    - any:53
    - tcp://any:53
  auto-route: true
  auto-detect-interface: true
  strict-route: true
  disable-icmp-forwarding: true
  route-exclude-address:
    - 192.168.0.0/16
    - 10.0.0.0/8
    - 172.16.0.0/12
    - fc00::/7
```

---

## 4. 启动 Mihomo

你需要完成两件事：

1. **将系统 DNS 设置为 `127.0.0.1`**
2. **以 root 权限运行 Mihomo**

### 设置系统 DNS

路径如下：

**系统设置 → 网络 → 当前使用的网络接口 → 详细信息 → DNS**

将 DNS 服务器改为：

`127.0.0.1`

### 启动 Mihomo

```bash
sudo mihomo
```

---

## 为什么必须这样做？

### 如果不将系统 DNS 设置为 `127.0.0.1`

可能出现以下情况：

* 如果 DNS 是通过 DHCP 自动获取的，则所有查询都会无条件绕过 Mihomo，直接使用本地 DNS；
* 如果你手动设置了其他 DNS，则部分查询可能绕过 Mihomo。

### 如果不以 root 身份运行

TUN 模式将无法正常工作。

---

## 自动化建议

你可以编写一个 Bash 脚本，通过 macOS 自带的 `networksetup` 命令自动完成：

* 设置系统 DNS
* 启动 Mihomo
* 恢复原始 DNS

当然，也可以让 AI 帮你生成这个脚本。
