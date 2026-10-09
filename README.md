# mihomo-config

[3x-ui](https://github.com/zenvor/3x-ui) subconverter 模块使用的 Mihomo 配置（订阅模板）。

## 工作原理

面板（subconverter）通过 raw 地址定时拉取 `main` 分支的 `mihomo.yaml`：

```
https://raw.githubusercontent.com/zenvor/mihomo-config/main/mihomo.yaml
```

- 拉取使用 `If-None-Match`（ETag）条件请求，内容没变化时不重复下载。
- 拉取间隔 10 分钟；也可以在面板的 subconverter 设置里手动刷新，立即生效。
- 下载内容校验通过才会写入缓存（见下方占位符约定），所以 push 了坏文件不会污染线上订阅。
- 修改本仓库的 `mihomo.yaml` 不需要重新部署面板。

## 占位符约定

配置里有两个占位符，面板在响应订阅请求时替换：

| 占位符 | 替换为 |
|---|---|
| `__API_DOMAIN__` | 订阅请求的 `scheme://host`（反代时取 `X-Forwarded-*`） |
| `__TOKEN__` | 该订阅的访问 token |

两个占位符都必须出现在文件里，否则面板拒绝更新缓存（保留上一份可用配置）。

## OpenAI / Anthropic 前置代理规则

`rules` 中加入了以下来源列出的域名，统一指向 `PROXY`，优先于广告、国内域名和中国 IP 直连规则；已有 QUIC（UDP 443）拦截仍优先执行：

- [OpenAI 网络建议](https://help.openai.com/zh-hans-cn/articles/9247338-network-recommendations-for-chatgpt-errors-on-web-and-apps)：允许列表，以及文中用于 iOS 排障的 Apple App Attest 主机。
- [Claude Desktop 网络要求](https://code.claude.com/docs/en/desktop#network-access-requirements)：应用、CDN、用户内容和 MCP 交互组件，以及 Artifact 字体和库依赖。
- [Claude Code 网络要求](https://code.claude.com/docs/en/network-config#network-access-requirements)：API、认证、更新、插件、遥测及可选 Gerrit 主机。
- [用户提供的社区帖子](https://x.com/wlzh/status/2108017417670860900)：补充短链接、CDN、认证和遥测域名。这部分不是官方完整清单；仅采用域名规则，策略改为 `PROXY`，不采用帖子的 `direct` 出口、固定 IP/ASN 或 `FINAL,DIRECT`。

去重时，`DOMAIN-SUFFIX` 已覆盖的具体子域名不再重复添加。核心域名覆盖新增子域名，但无法预先覆盖新增的独立域名、用户自定义 MCP/连接器、第三方模型提供商或任意外部链接。

这些规则按目标域名匹配，不能只代理某个应用的请求：其他应用访问同一第三方主机也会走代理；帖子中的 `datadog`、`sift` 关键词规则会覆盖所有包含这些词的域名。已列出的遥测域名优先于广告拦截。匹配时仅有 IP、没有域名的请求不受这些域名规则保护，仍按后续 IP 规则分流。

本次前置规则不改变 DNS 上游选择策略：命中前置域名规则的连接直接选择代理出口，即使该域名的真实 IP 属于中国；其余流量继续按下方 DNS/IP 分流策略处理。清单记录的是当前已知依赖，不保证未来所有相关请求都使用代理，也不构成账号安全保证。

## DNS 策略

- `cn`、`private`、`tld-cn` 集合中的域名使用腾讯/阿里 DoH；`.cn` 由 `tld-cn` 覆盖。
- 其他域名（包括未收录的新域名）默认使用 Google/Cloudflare DoH，并通过当前 `PROXY` 节点查询。
- 不配置 `fallback`、`fallback-filter` 或 `direct-nameserver`，避免未知域名回退国内 DNS，以及直连出口绕过域名 DNS 策略重新解析。这些国外上游查询失败时，本配置不会改用系统或国内 DNS；这不涵盖浏览器、系统或代理节点自行解析的路径。
- `proxy-server-nameserver` 保留腾讯/阿里 DoH，仅解析代理节点域名，避免建立代理连接时循环依赖；`default-nameserver` 仅用于解析 DNS 上游服务器的域名。

连接出口按 `rules` 从上到下匹配，已命中域名规则的请求直接按该规则处理。域名未收录且不是中国后缀时，默认使用经过代理的 Google/Cloudflare DoH；若未命中前面的规则，`GEOIP,CN,DIRECT` 会主动解析真实 IP，再进行分流：命中中国 IP 段数据集则直连，未命中（包括无法识别的 IP 段）由 `MATCH,PROXY` 兜底代理。DNS 查询经过代理，不代表后续业务连接也一定经过代理。

前置代理清单之外，已有域名直连、广告和 QUIC 拦截策略保留，因此本配置不等同于所有请求都强制代理，也不构成账号安全保证；未被前置规则覆盖的新 OpenAI/Anthropic 独立域名若解析到中国 IP 段，也会直连。域名未收录且不是中国后缀时使用国外 DNS，可能影响国内 CDN 调度；已有 Google Play 大陆分发直连规则不变，但其解析只有命中国内域名集合时才保证使用国内 DNS。

参考：[Mihomo DNS 文档](https://wiki.metacubex.one/config/dns/)、[路由规则与 no-resolve](https://wiki.metacubex.one/config/rules/#no-resolve)。推送后还需面板重新拉取模板、客户端更新订阅并加载新配置；提交成功不等于运行中的客户端已经应用。
