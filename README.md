# capyvv/rules

个人 Clash / Mihomo 规则配置。

主要配置文件：

- `clash/Custom.ini`：Subconverter / ACL4SSR 自定义配置入口。
- `clash/rules/OfficeDirect.list`：个人需要强制直连的公司/内网域名。

项目尽量直接引用 ACL4SSR 上游规则，仅维护真正需要的个人规则，减少重复维护。

## 设计原则

### 1. 精确规则优先于宽泛规则

Clash / Mihomo 规则按照从上到下顺序匹配，命中后停止继续匹配。

因此配置顺序遵循：

```text
精确业务规则
    ↓
普通海外代理规则
    ↓
国内域名/IP兜底
    ↓
FINAL
```

AI、Google、媒体、开发工具等专用规则放在前面；`ProxyGFWlist`、`ChinaDomain`、`ChinaCompanyIp`、`GEOIP,CN` 等宽泛规则放在后面，避免提前覆盖更具体的业务规则。

### 2. 节点选择与业务分流分离

`🚀 节点选择` 只是基础出口选择器，不直接承载网站规则。

各业务分组可以独立选择：

```text
🚀 节点选择
♻️ 自动选择
具体节点
DIRECT
```

这样既可以统一控制默认出口，也可以针对 AI、Google、开发工具、媒体等服务单独指定节点。

### 3. 隐藏强制直连规则

以下规则直接使用内置 `DIRECT`，不额外创建可见策略组：

- 局域网地址：ACL4SSR `LocalAreaNetwork.list`
- 明确白名单：ACL4SSR `UnBan.list`
- 个人公司/内网域名：`clash/rules/OfficeDirect.list`

这样可以避免这些明确需要直连的流量被误切换到代理节点。

### 4. 专用服务独立分组

当前主要业务分组包括：

- `🤖 AI服务`：OpenAI、Claude、Gemini
- `📢 谷歌服务`：Google、Google FCM、GoogleCN
- `💻 开发工具`：GitHub 等开发相关服务
- `Ⓜ️ 微软服务`
- `🍎 Apple服务`
- `📺 国内媒体`
- `▶️ 海外媒体`
- `📱 Telegram`
- `🌐 海外服务`

保留 `▶️ 海外媒体` 与 `🌐 海外服务` 两个独立分组。当前它们可以使用相同节点，但以后如果出现 Netflix、Disney+ 等流媒体解锁节点，可以直接在海外媒体分组中单独管理，而不影响普通海外网站。

## 国内媒体规则

国内媒体不再维护自定义 `CNMedia.list`，全部使用 ACL4SSR 上游专用规则：

```text
ByteDance.list
TencentVideo.list
Youku.list
Iqiyi.list
```

具体原则：

- 抖音及相关字节域名由 `ByteDance.list` 负责；
- 腾讯只使用 `TencentVideo.list`，避免将 `qq.com`、`tencent.com` 等腾讯通用业务整体归入国内媒体；
- 优酷使用 `Youku.list`；
- 爱奇艺使用 `Iqiyi.list`。

这样可以避免自行维护媒体域名，同时减少过宽规则导致的误匹配。

## 海外媒体与 Google 的顺序

`YouTube.list` 必须位于宽泛的 `Google.list` 前面。

原因是 Google 规则中也可能包含 YouTube 相关域名。如果 Google 先匹配，YouTube 流量会进入 `📢 谷歌服务`，而不是预期的 `▶️ 海外媒体`。

当前逻辑：

```text
YouTube / Netflix / ProxyMedia
    ↓
Google FCM / Google / GoogleCN
```

## 普通海外服务

普通海外网站统一使用 ACL4SSR：

```text
ProxyGFWlist.list
```

并进入：

```text
🌐 海外服务
```

`🚀 节点选择` 本身不直接承接 `ProxyGFWlist`，从而保持“基础选择器”和“业务规则”分离。

## 自动选择说明

`♻️ 自动选择` 使用 Mihomo `url-test`：

```text
检测地址：https://www.gstatic.com/generate_204
测速周期：300 秒
延迟容忍：100 ms
```

其作用是测试通过各代理节点访问检测地址的响应时间，并自动选择延迟较低且可用的节点。

注意：

- 测试的是代理连接和 HTTP/TCP 响应时间，不是下载带宽测速；
- 不等同于 ICMP Ping；
- Ping 被禁但代理连接正常时，节点仍可正常使用；
- 代理连接失败或检测超时时，节点会被视为不可用；
- `100 ms` 容忍值用于减少几十毫秒波动造成的频繁节点切换。

## 当前规则优先级

```text
LocalAreaNetwork / UnBan / OfficeDirect → DIRECT
    ↓
广告拦截 / 应用净化
    ↓
AI 服务
    ↓
Microsoft / Apple
    ↓
海外媒体（YouTube / Netflix / ProxyMedia）
    ↓
Google 服务
    ↓
开发工具
    ↓
国内媒体（ByteDance / TencentVideo / Youku / Iqiyi）
    ↓
Telegram
    ↓
普通海外服务 ProxyGFWlist
    ↓
ChinaDomain
    ↓
ChinaCompanyIp
    ↓
GEOIP,CN
    ↓
FINAL → 漏网之鱼
```

其中规则匹配顺序与 Clash Party 中策略组的显示顺序是两个独立概念，不需要保持完全一致。

## 策略组设计

基础选择器：

```text
🚀 节点选择
♻️ 自动选择
```

主要业务分组均可直接看到基础选择器、自动选择和具体节点，方便临时切换。

`🎯 全球直连` 仅保留 `DIRECT`，避免国内兜底规则被误切换到代理。

`🐟 漏网之鱼` 只承接真正没有被前面规则匹配的 `FINAL` 流量。

## 规则维护原则

1. 优先引用 ACL4SSR 等成熟公共规则。
2. 上游已有专用规则时，不在本仓库重复维护相同域名。
3. 只有实际出现误分流或特殊直连需求时，才增加个人规则。
4. 新增规则前先检查与现有规则的覆盖关系和匹配顺序。
5. 个人自定义规则统一放在 `clash/rules/`。
6. 当前仅保留确有必要的 `OfficeDirect.list`；国内媒体已完全切换为上游规则。

## 当前维护目标

这套配置主要追求：

```text
规则覆盖完整
+
业务分类清晰
+
节点切换方便
+
尽量少维护自定义域名
+
为未来流媒体专用节点等场景保留扩展空间
```
