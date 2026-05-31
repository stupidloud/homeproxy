# HomeProxy 自定义模式指南

## 一、大陆白名单 → 自定义模式等效迁移

白名单模式切换为自定义模式后，DNS 分流、规则集、出站节点需手动创建等效配置。

### 前提：已存在的配置
- 主节点 `<your-main-node-section>`（Taiwan 03 ANYTLS 为例）
- WAN DNS 自动获取（本例为 ISP 分配的 DNS，如 `xx.xx.xx.xx`）

### 步骤

```bash
# 1. 切换路由模式 + 基础参数
uci set homeproxy.config.routing_mode='custom'
uci set homeproxy.routing.sniff_override='0'        # 关闭 sniff 目标覆盖
uci set homeproxy.routing.default_outbound='rnode_main'
uci set homeproxy.routing.default_outbound_dns='default-dns'
uci set homeproxy.dns.default_server='dns_main'

# 2. DNS 服务器（main-dns 代理解析 + china-dns 国内解析）
uci set homeproxy.dns_main=dns_server
uci set homeproxy.dns_main.enabled='1'
uci set homeproxy.dns_main.label='Main DNS'
uci set homeproxy.dns_main.type='tcp'
uci set homeproxy.dns_main.server='1.1.1.1'
uci set homeproxy.dns_main.address_resolver='default-dns'
uci set homeproxy.dns_main.outbound='rnode_main'

uci set homeproxy.dns_china=dns_server
uci set homeproxy.dns_china.enabled='1'
uci set homeproxy.dns_china.label='China DNS'
uci set homeproxy.dns_china.type='udp'
uci set homeproxy.dns_china.server='119.29.29.29'
uci set homeproxy.dns_china.address_resolver='default-dns'
uci set homeproxy.dns_china.address_strategy='prefer_ipv6'
uci set homeproxy.dns_china.outbound='direct-out'

# 3. DNS 规则（geosite-cn → china-dns）
uci set homeproxy.dnsrule_cn=dns_rule
uci set homeproxy.dnsrule_cn.enabled='1'
uci set homeproxy.dnsrule_cn.label='CN DNS Route'
uci add_list homeproxy.dnsrule_cn.rule_set='rs_geosite_cn'
uci set homeproxy.dnsrule_cn.action='route'
uci set homeproxy.dnsrule_cn.server='dns_china'
uci set homeproxy.dnsrule_cn.domain_strategy='prefer_ipv6'

# 4. 逻辑 AND DNS 规则（需要 custom-mode-logical-rules 特性）
# 子规则1: NOT geosite-noncn
uci set homeproxy.dnsrule_noncn=dns_rule
uci set homeproxy.dnsrule_noncn.enabled='1'
uci set homeproxy.dnsrule_noncn.label='Not CN'
uci add_list homeproxy.dnsrule_noncn.rule_set='rs_geosite_noncn'
uci set homeproxy.dnsrule_noncn.invert='1'

# 子规则2: geoip-cn
uci set homeproxy.dnsrule_cnip=dns_rule
uci set homeproxy.dnsrule_cnip.enabled='1'
uci set homeproxy.dnsrule_cnip.label='CN IP'
uci add_list homeproxy.dnsrule_cnip.rule_set='rs_geoip_cn'

# 逻辑规则: AND 组合
uci set homeproxy.dnsrule_logical=dns_rule
uci set homeproxy.dnsrule_logical.enabled='1'
uci set homeproxy.dnsrule_logical.label='Logical AND CN'
uci set homeproxy.dnsrule_logical.rule_type='logical'
uci set homeproxy.dnsrule_logical.logical_mode='and'
uci add_list homeproxy.dnsrule_logical.logical_rules='dnsrule_noncn'
uci add_list homeproxy.dnsrule_logical.logical_rules='dnsrule_cnip'
uci set homeproxy.dnsrule_logical.action='route'
uci set homeproxy.dnsrule_logical.server='dns_china'
uci set homeproxy.dnsrule_logical.domain_strategy='prefer_ipv6'

# 5. 路由节点（包装主节点）
uci set homeproxy.rnode_main=routing_node
uci set homeproxy.rnode_main.enabled='1'
uci set homeproxy.rnode_main.label='Main Outbound'
uci set homeproxy.rnode_main.node='<your-main-node-section>'
uci set homeproxy.rnode_main.outbound='direct-out'

# 6. 规则集（geoip-cn, geosite-cn, geosite-noncn）
uci set homeproxy.rs_geoip_cn=ruleset
uci set homeproxy.rs_geoip_cn.enabled='1'
uci set homeproxy.rs_geoip_cn.label='GeoIP CN'
uci set homeproxy.rs_geoip_cn.type='remote'
uci set homeproxy.rs_geoip_cn.format='binary'
uci set homeproxy.rs_geoip_cn.url='https://fastly.jsdelivr.net/gh/1715173329/IPCIDR-CHINA@rule-set/cn.srs'
uci set homeproxy.rs_geoip_cn.outbound='rnode_main'

uci set homeproxy.rs_geosite_cn=ruleset
uci set homeproxy.rs_geosite_cn.enabled='1'
uci set homeproxy.rs_geosite_cn.label='GeoSite CN'
uci set homeproxy.rs_geosite_cn.type='remote'
uci set homeproxy.rs_geosite_cn.format='binary'
uci set homeproxy.rs_geosite_cn.url='https://fastly.jsdelivr.net/gh/1715173329/sing-geosite@rule-set-unstable/geosite-geolocation-cn.srs'
uci set homeproxy.rs_geosite_cn.outbound='rnode_main'

uci set homeproxy.rs_geosite_noncn=ruleset
uci set homeproxy.rs_geosite_noncn.enabled='1'
uci set homeproxy.rs_geosite_noncn.label='GeoSite !CN'
uci set homeproxy.rs_geosite_noncn.type='remote'
uci set homeproxy.rs_geosite_noncn.format='binary'
uci set homeproxy.rs_geosite_noncn.url='https://fastly.jsdelivr.net/gh/1715173329/sing-geosite@rule-set-unstable/geosite-geolocation-!cn.srs'
uci set homeproxy.rs_geosite_noncn.outbound='rnode_main'

# 7. 提交并重启
uci commit homeproxy
/etc/init.d/homeproxy restart
```

### 等效性对比

| 项目 | 白名单模式 | 自定义模式 |
|------|-----------|-----------|
| DNS 国内分流 | geosite-cn → china-dns | 等效 |
| 逻辑 AND 规则 | (NOT noncn) AND geoip-cn → china-dns | 等效（需 logical 特性） |
| DNS 最终 | main-dns | 等效 |
| 路由最终 | main-out | 等效 |
| sniff_override_destination | 可关 | 可关 |
| 规则集 | 自动生成 | 手动创建 |

### 关键注意事项
- **必须用命名 section**（`uci set homeproxy.xxx=type`），不能 `uci add`，否则自动生成的名字无法被其他 section 引用
- `rule_set` 等 list 类型字段必须用 `uci add_list`，`uci set` 不生效
- 自定义模式 `default_domain_resolver` 行为略有不同（`action: resolve` vs `action: route`），日常无感
- **带 `!reverse` 的 depends** 语法：在 `dns_rule` logical 模式中隐藏匹配字段、显示动作字段
- **自定义模式默认全局代理**：nftables 层无 bypass 逻辑，所有流量兜底重定向到 sing-box。`route.final` 决定了未匹配流量的最终出站——设为 `direct-out` 则未匹配流量直连，保持 `rnode_main` 则为全局代理。

---

## 二、劫持特定域名到自定义 IP

### 场景
绕过 GFW 对目标域名的 SNI 封锁，将流量通过 VLESS 代理隧道导向自有服务器 `<your-server-ip>`。

### 方案：DNS 预定义 + 路由 override_address + VLESS

#### 核心原理
- DNS predefined 返回自定义 IP，客户端直接连接该 IP
- 路由规则匹配 SNI 域名 → 走 VLESS 代理 → `override_address` 改写目标
- VLESS 是标准 TCP 隧道，**支持目标地址改写**（ANYTLS 不支持）

#### 配置

```bash
# 1. DNS 劫持：example.com → 1.2.3.4
uci set homeproxy.dnsrule_target=dns_rule
uci set homeproxy.dnsrule_target.enabled='1'
uci set homeproxy.dnsrule_target.label='example.com → 1.2.3.4'
uci add_list homeproxy.dnsrule_target.domain='example.com'
uci set homeproxy.dnsrule_target.action='predefined'
uci set homeproxy.dnsrule_target.predefined_rcode='NOERROR'
uci add_list homeproxy.dnsrule_target.predefined_answer='example.com. 300 IN A 1.2.3.4'

# 2. VLESS 路由节点（选择你的 VLESS 节点）
uci set homeproxy.rnode_vless=routing_node
uci set homeproxy.rnode_vless.enabled='1'
uci set homeproxy.rnode_vless.label='VLESS Node'
uci set homeproxy.rnode_vless.node='<your-vless-node-section>'
uci set homeproxy.rnode_vless.outbound='direct-out'

# 3. 路由规则：匹配 example.com → VLESS 代理 + 改写地址
uci set homeproxy.route_target=routing_rule
uci set homeproxy.route_target.enabled='1'
uci set homeproxy.route_target.label='example.com via VLESS'
uci add_list homeproxy.route_target.domain='example.com'
uci set homeproxy.route_target.action='route'
uci set homeproxy.route_target.outbound='rnode_vless'
uci set homeproxy.route_target.override_address='1.2.3.4'

uci commit homeproxy
/etc/init.d/homeproxy restart
```

#### 验证

```bash
# DNS 劫持确认
nslookup example.com 127.0.0.1:5333
# 应返回 1.2.3.4

# 内容确认（非 Cloudflare cdn-cgi/trace，而是实际 WordPress 页面）
curl https://example.com/ -sk | grep -o '<title>[^<]*</title>'
```

---

## 三、踩坑记录

### 1. ANYTLS 不支持目标地址改写

**现象：** 路由规则 `override_address=1.2.3.4` 在日志中显示生效，代理出站到 `1.2.3.4:443`，但实际连接到域名公网 IP。

**原因：** ANYTLS 协议在代理端按域名重新解析 → 公网 IP，忽略目标地址。

**解决：** 使用 VLESS/Shadowsocks/Trojan 等标准 TCP 隧道协议。

### 2. GFW DPI 拦截 SNI

**现象：** 路由器直连 `1.2.3.4:443` 正常（SNI=IP），但 `SNI=目标域名` → TLS 握手被 RST（`unexpected eof while reading`）。

**验证：**
```bash
curl https://1.2.3.4/ -H 'Host: example.com' -sk    # SNI=IP → 能通
curl https://example.com/ --resolve example.com:443:1.2.3.4 -sk  # SNI=域名 → 被阻
```

**解决：** 必须走代理隧道，让 TLS 在隧道内传输，GFW 看不到 SNI。

### 3. `tls_fragment` 绕不过 GFW

**现象：** 路由规则加 `tls_fragment: true` 后直连仍然被 RST。

**原因：** GFW 的封锁可能是精确 SNI 匹配级别，`tls_fragment` 只是拆分 TLS 记录，SNI 仍然明文可见。

### 4. UCI `add_list` vs `set`

- `rule_set`、`domain`、`logical_rules` 等字段是 list 类型
- `uci set xxx.rule_set='value'` 不生效 → 必须用 `uci add_list`
- `uci delete` 清空旧值后再 `add_list` 是最安全的做法

### 5. UCI 命名 section vs 匿名 section

- `uci add homeproxy dns_rule` → 自动生成 `cfg021234` 这样的匿名名
- 其他 section 通过名称引用时（如 `outbound='rnode_main'`），必须用命名 section
- `uci set homeproxy.rnode_main=routing_node` → section 名即为 `rnode_main`

### 6. `override_address` 需要 `action: route`（非 `route-options`）

- `action: route-options` + `override_address` → 不生效
- `action: route` + `override_address` + `outbound` → 正确

### 7. `sniff_override_destination` vs `sniff`

- **`sniff_override_destination`**（可关闭）：控制是否用嗅探域名覆盖目标地址
- **`sniff`**（不可关闭，硬编码 `true`）：始终分析流量协议/域名，用于路由规则匹配
- 两者独立，关闭前者不影响 DNS/路由层面的域名匹配

### 8. DNS 规则顺序

- DNS 规则按数组顺序匹配，第一条命中即停止
- 域名劫持规则必须排在 geosite/geoip 分流规则**之前**

### 9. sing-box 的 cache.db

- DNS 缓存持久化在 `/var/run/homeproxy/cache.db`
- 修改 DNS 规则后应清除：`rm -f /var/run/homeproxy/cache.db && /etc/init.d/homeproxy restart`

### 10. `cdn-cgi/trace` 是 Cloudflare 专用端点

- 只有经过 Cloudflare CDN 的请求才会返回该端点的标准响应
- 直连源站的请求返回 404
- 可用它快速判断请求走了 Cloudflare（`colo=XXX`）还是直连源站（404）

### 12. 自定义模式 `route.final` = 默认兜底出站

**现象：** 切换到自定义模式后，SSH、局域网流量等本该直连的流量也走了代理。

**原因：** `firewall_post.ut:312-370` 的 nftables 链中，非自定义模式有 gfwlist/大陆白名单 bypass 逻辑，但自定义模式这些分支全部跳过，最终 line 369 兜底 `goto homeproxy_redirect_proxy_port`——所有流量进入 sing-box。

**解决：** 将 `route.final`（即 `default_outbound`）设为 `direct-out`：

```bash
uci set homeproxy.routing.default_outbound='direct-out'
uci commit homeproxy
/etc/init.d/homeproxy restart
```

未匹配路由规则的流量直连，有明确规则（如特定域名 → VLESS + override_address）的仍然走代理。
