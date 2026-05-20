# macOS Mihomo core DNS/TUN tutorial

This guide shows how to configure Mihomo's TUN mode and DNS resolver in macOS to prevent leaks and route Apple services directly, avoiding breakage.

It will intercept all DNS system-wide (no exceptions) and resolve DIRECT rules through your provided nameservers. Everything else will be handled by the proxy server. It will also apply to CLI tools (Homebrew, SSH) and Wine - no extra setup needed.

The recommended way to manage Mihomo is to install it via Homebrew (`brew install mihomo`) and keep the default config folder (~/.config/mihomo)

## Notes

1. It's **not recommended to use GUI clients with this setup**, because they tend to override Mihomo's DNS/TUN blocks and conflicts may occur. The setup is intended for using Mihomo core as-is, or with Zashboard webUI that you can host this way:
```
mkdir ~/.config/mihomo/ui && cd ~/.config/mihomo/ui
curl -L https://github.com/Zephyruso/zashboard/releases/latest/download/dist-no-fonts.zip -o zash.zip
unzip zash.zip && mv dist/* . && rm -rf dist zash.zip
```
And in your Mihomo config.yaml:
```
external-controller: 127.0.0.1:9090
external-ui: /Users/YOUR-USERNAME/.config/mihomo/ui
```

2. **Your server must have its own DNS configured** - this setup relies on remote resolution.

3. **This is only a routing template. Proxy servers must be provided by the user.**

Everything onwards is meant to go in config.yaml in your Mihomo config folder.

## Guide

### 1. Set up rule providers (these are fully functional rulesets from blackmatrix7 - no alterations needed)

```
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
Then specify rules:
```
- DOMAIN,raw.githubusercontent.com,DIRECT
- RULE-SET,LAN,DIRECT
- RULE-SET,Apple,DIRECT
```

### 2. Add a DNS block

```
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

The important things about this template are the **listener on 127.0.0.1:53**, **fake-ip mode** with ruleset blacklists and **`respect-rules: false`**. Mihomo will return a fake IP for any query that doesn't match blacklisted rulesets, avoiding resolving it locally (thus, no leak occurs). We don't need `respect-rules` since we're using fake-ip - enabling it will only result in unnecessary queries and potential bootstrap deadlocks.

In case you need to receive real IPs for your own rulesets or domains, you can add them under `fake-ip-filter` but **be aware that any blacklisted domain will be resolved locally**.

You can replace Aliyun with any other DNS or set up a fallback if you want to - however, you're not likely to need it, since this setup is intended for strict remote DNS resolution of proxied queries and you will not be dealing with GFW's DNS poisoning of overseas websites. `direct-nameserver` and `proxy-server-nameserver` are also unnecessary - `nameserver` takes over them.

### 3. Add a TUN block

```
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

### 4. Start Mihomo
This is where the magic happens. 

You need to **set your system DNS to 127.0.0.1** (System Settings -> Network -> Your active network interface -> Details -> DNS) and **run Mihomo as root**. If you **don't** set system DNS to 127.0.0.1, two things may occur: all queries will go through local DNS unconditionally (if it's obtained with DHCP), or some queries will bypass Mihomo (if you already have different DNS specified there). If you don't run Mihomo as root, TUN mode will not work properly.

You can write a bash script with `networksetup` commands (or ask an AI to do it for you) to handle it automatically.
