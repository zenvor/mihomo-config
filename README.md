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
