---
name: chained-vps-proxy-deployment
description: Use when deploying, migrating, or troubleshooting an authorized multi-node chained proxy in which a relay node forwards client traffic to a separate egress node, especially with Xray, x-ui, sing-box, VLESS, Reality, VMess/WebSocket, IPv4/IPv6, firewall rules, NAT, cloud port mappings, or V2rayN delivery.
metadata:
  short-description: Deploy and verify an authorized chained proxy by discovered roles
---
# 二级代理（中转→落地）部署技能

## Overview

把具有不同职责的节点组织成一条可验证、可回滚的链路：

```text
客户端 → Relay（中转/入口）→ Egress（落地/出口）→ Internet
```

本技能使用**角色、能力和运行时事实**描述流程，而不是假设地区、供应商、固定端口、固定服务名或固定协议。具体实现应根据审计结果在以下方案中选择：

- **协议终止**：Relay 接收客户端协议，再以兼容协议连接 Egress；
- **透明转发**：Relay 只转发 TCP/UDP，不终止客户端协议；
- **混合模式**：入口和上游分别采用不同的协议或地址族。

目标是形成闭环：**发现 → 选择方案 → 备份 → 最小变更 → 分层验证 → 交付或回滚**。所有真实地址、凭据、私钥、UUID、Token 和完整分享链接只在运行时安全使用，不写入本技能或普通变更记录。

仅用于已获授权的服务器。只连接用户明确提供的目标，不做未授权扫描、凭据猜测、撞库或第三方探测。

## Terminology and Scope


| 术语               | 含义                                                       |
| -------------------- | ------------------------------------------------------------ |
| `Relay`            | 面向客户端的中转节点；可以是协议终止点，也可以是透明转发点 |
| `Egress`           | 面向 Internet 的落地节点；提供上游入站并承担最终出口       |
| `Management plane` | SSH、面板、API、云控制台等管理通道                         |
| `Data plane`       | 客户端代理流量经过的入口、上游、路由和出口                 |
| `Edge mapping`     | 云安全组、NAT、容器映射、负载均衡或宿主机端口转发          |
| `Client profile`   | 客户端需要导入的节点、订阅或配置片段                       |

默认处理一个 Relay 和一个 Egress。若存在额外跳点，把中间节点按相同角色模型重复审计；不要因为节点数量增加就跳过单跳验证。

## When to Use

适用于以下请求：

- 在两台或多台自有节点之间建立中转到落地的链式代理；
- 需要 Xray/x-ui、sing-box 或其他兼容代理组件协同工作；
- 需要 VLESS、Reality、VMess、WebSocket、透明 TCP/UDP 转发或跨 IPv4/IPv6 链路；
- 需要迁移入口、切换地址族、调整云端端口映射或限制上游来源；
- 既有服务必须保留，同时需要追加入站、出站、路由或安全策略；
- 需要生成 V2rayN 或其他客户端可以使用的配置，并验证最终出口。

不适用于：

- 未获授权的主机探测、代理滥用、匿名化攻击或规避第三方访问控制；
- 没有明确目标、角色、管理方式和凭据边界的“先扫一遍再说”；
- 只需要单节点常规安装、且不存在中转到落地链路的请求。

## Core Invariants

1. **先发现，后假设。** 服务名、配置路径、监听地址、协议、端口、防火墙实现和云端映射都必须从目标环境读取；技能中的占位符不是默认值。
2. **先只读，后修改。** 第一次连接只执行身份、系统、进程、监听、配置摘要、路由和网络策略审计。
3. **先备份，后变更。** 每个会影响服务、路由、NAT 或防火墙的变更都要有可识别、权限正确、可验证的回滚快照。
4. **角色边界清晰。** Relay 的客户端入口、Relay→Egress 的上游连接和 Egress 的最终出口必须分别命名、分别验证。
5. **优先最小变更。** 能追加专用入站就不要覆盖现有入站；能复用兼容上游就不要无理由替换稳定服务。
6. **不清空安全策略。** 安装脚本或面板提供的“关闭/清空防火墙、全放行流量”选项默认拒绝；除非有明确隔离环境和恢复方案。
7. **地址族独立验证。** IPv4 和 IPv6 的 DNS、云端规则、监听、防火墙、路由和外部可达性不能互相推断。
8. **透明转发不等于协议终止。** 透明 TCP/UDP 转发不会自动产生新的代理入站；从透明转发迁移到协议终止时，客户端参数、鉴权和验收方式都要重新确认。
9. **端口可达不等于链路可用。** 监听和 TCP 探测只能证明局部网络层，不能证明协议握手、路由和最终出口。
10. **敏感信息最小暴露。** 普通记录只保留脱敏摘要、参数类型和验证结论；密码、私钥、数据库、完整分享链接和管理 Token 不回显、不落盘到项目。

## Runtime Inventory

执行前建立一份仅在内存或安全凭据存储中使用的运行时清单。变量名是示意，实际可按目标环境重命名；不要把真实值写回本技能。


| 变量                               | 含义                                                                  |
| ------------------------------------ | ----------------------------------------------------------------------- |
| `RELAY_ENDPOINTS`                  | Relay 对客户端暴露的地址、地址族和入口端口集合                        |
| `RELAY_ADMIN_ENDPOINT`             | Relay 的 SSH/面板/API 管理入口                                        |
| `RELAY_LISTEN_SPEC`                | Relay 服务实际监听的地址族、地址和端口                                |
| `EGRESS_ENDPOINT`                  | Egress 为 Relay 提供的上游入站地址、地址族和端口                      |
| `EGRESS_ALLOWED_SOURCE`            | Egress 允许访问上游入站的 Relay 地址或安全组来源                      |
| `RELAY_UPSTREAM_SPEC`              | Relay 到 Egress 的协议、鉴权、传输和地址族参数                        |
| `RELAY_SERVICE` / `EGRESS_SERVICE` | 审计后确定的实际服务单元，不要预先假定名称                            |
| `RELAY_CONFIG` / `EGRESS_CONFIG`   | 审计后确定的实际生效配置或生成配置路径                                |
| `CONTROL_DB`                       | 如果有面板或数据库，记录实际路径和权限                                |
| `EDGE_MAPPING`                     | 云安全组、NAT、容器、负载均衡或宿主机映射关系                         |
| `FIREWALL_BACKEND`                 | 实际生效的 nftables、iptables、ip6tables、firewalld、ufw 或其他策略层 |
| `BACKUP_DIR`                       | 远端受保护的本次变更快照目录                                          |
| `CLIENT_PROFILE`                   | 目标客户端所需的配置格式和导入信息                                    |

### 事实与决策分离

把清单分成两部分：

- **Facts**：从系统或云平台读到的事实，例如“某服务监听某地址”“某规则允许某来源”；
- **Decisions**：本次选择的架构，例如“采用协议终止”“只允许 Relay 来源”“新入口验收后删除旧入口”。

没有事实支撑的决策必须标记为候选方案，不能直接写入生产配置。

## Architecture Decision Gate

在修改前回答以下问题：

1. **目标拓扑是什么？** 客户端是否只连接 Relay，还是客户端还要启用前置代理？
2. **Relay 是否终止协议？**
   - 需要独立的客户端节点、鉴权、路由或地址族转换：选择协议终止；
   - 只要求原样转发且上游协议已经固定：评估透明转发；
   - 两者并存时，为每条入口分别建立配置和验收记录。
3. **Egress 当前提供什么？** 从运行配置和监听确认协议、传输、鉴权、来源限制和地址族；不要只看面板名称或历史文档。
4. **云端边界在哪里？** 区分公网端口、宿主机端口、容器端口、服务监听端口和安全组规则。
5. **客户端和每一跳的地址族是什么？** 明确 `CLIENT_AF → RELAY_AF → UPSTREAM_AF → EGRESS_AF`，不能用“IPv6 已开启”替代逐段验证。
6. **是否有已有业务需要保持？** 先定义允许改变的文件、服务、规则和端口；未纳入范围的资源默认不修改。

### 方案选择矩阵


| 条件                           | 倾向方案                      | 主要验证                                     |
| -------------------------------- | ------------------------------- | ---------------------------------------------- |
| 上游已有兼容协议，追求最小改动 | Relay 使用对应出站连接 Egress | 上游鉴权、来源限制、最终出口                 |
| 需要独立客户端入口或地址族转换 | Relay 协议终止并建立专用路由  | 入口握手、路由 tag、上游协议、出口           |
| 只需原样搬运单一 TCP/UDP 流    | 透明转发或专用转发组件        | NAT/转发、回程、外部协议握手                 |
| 客户端与上游地址族不同         | 显式设置监听和出站地址族      | DNS、监听、内核策略、云端 IPv6、外部双栈测试 |
| 现有落地节点承载其他业务       | 新增隔离入站或专用服务        | 配置合并、端口冲突、来源 ACL、回滚           |

选择能满足业务目标的最简单方案。不要为了“优化”叠加额外跳点、传输层、Mux 或代理层；任何额外组件都必须有明确收益和独立验证。

## Deployment Workflow

### 0. 确认授权、边界和安全输入

收集：

- Relay 和 Egress 的明确地址、管理端口及角色；
- 入口、上游、管理面板和云端映射的预期范围；
- 目标客户端和出口验证方式；
- 当前不能修改的业务、端口和服务；
- 可用的 SSH 密钥、Agent 或安全凭据输入方式。

约束：

- 只尝试用户明确提供的地址和管理端口，不扫描邻近端口；
- 凭据失败后停止猜测，改用正确的密钥或安全输入；
- 密码不放入命令行、脚本、文件、聊天或 shell history；
- 变更期间保留当前管理会话，并在验证新路径前不要关闭最后一个可用会话。

### 1. 只读审计并建立事实基线

在每个目标节点执行与系统匹配的只读检查。以下命令是发现方向，不是固定的服务名或配置路径：

```bash
id
cat /etc/os-release
ss -lntup
ps -eo pid,user,cmd --sort=cmd
systemctl list-units --type=service --state=running
systemctl list-unit-files --type=service
ip address
ip route
ip -6 route
```

根据发现结果，再读取实际服务单元、配置和日志：

```bash
# 仅在目标使用 systemd 时执行；服务单元名称来自前一步审计。
if command -v systemctl >/dev/null 2>&1; then
  systemctl cat "$DISCOVERED_SERVICE"
  systemctl --no-pager --full status "$DISCOVERED_SERVICE"
  journalctl -u "$DISCOVERED_SERVICE" -n 100 --no-pager
fi
```

按实际环境选择网络策略命令：

```bash
iptables-save                 # 仅当 iptables 是实际生效层
ip6tables-save               # 仅当 IPv6 规则由该层管理
nft list ruleset              # 仅当 nftables 是实际生效层
```

记录但不要回显：

- 服务单元、二进制路径、运行用户和启动参数；
- 实际生效配置与生成配置的关系；
- 每个监听的地址族、绑定地址、端口和进程；
- 入站、出站、路由、鉴权和传输参数的**类型**；
- 云端公网到宿主机/容器的映射；
- 管理入口、代理入口、上游入口和回程路径；
- 防火墙默认策略、来源限制和规则计数。

把 UUID、密码、私钥、Token、证书私钥和完整分享链接替换为 `<REDACTED>` 后再写入摘要。

### 2. 识别生效的配置来源

同一服务可能存在面板数据库、模板、生成文件、systemd 参数和运行时覆盖项。变更前确认：

1. 哪个文件或 API 是**权威输入**；
2. 哪个文件是**生成结果**；
3. 服务重启或面板同步后哪些内容会被覆盖；
4. 配置检查命令和实际启动命令使用的路径是否一致；
5. 防火墙是由系统服务、云平台、容器网络还是多个层级共同管理。

如果管理 API 返回 `404`、字段结构不符或写入后配置未生效，不要继续猜接口。先读取已安装版本的前端调用、服务日志、帮助信息和实际生成结果，再决定使用 API、面板还是直接编辑权威配置。

### 3. 创建可追溯的远端备份

任何安装、API 写入、配置替换、路由调整、NAT 删除或防火墙变更前，按实际发现的资源创建快照：

```bash
stamp=$(date -u +%Y%m%dT%H%M%SZ)
backup_dir="${BACKUP_ROOT:?set a protected remote backup root}/chain-proxy-$stamp"
install -d -m 700 "$backup_dir"

# 这些变量来自审计；空变量或不存在的路径不应被伪造为备份成功。
if [ -n "${CONTROL_DB:-}" ] && [ -f "$CONTROL_DB" ]; then
  cp --preserve=mode,ownership,timestamps "$CONTROL_DB" "$backup_dir/control-db.before"
fi
if [ -n "${RELAY_CONFIG:-}" ] && [ -f "$RELAY_CONFIG" ]; then
  cp --preserve=mode,ownership,timestamps "$RELAY_CONFIG" "$backup_dir/relay-config.before"
fi
if [ -n "${EGRESS_CONFIG:-}" ] && [ -f "$EGRESS_CONFIG" ]; then
  cp --preserve=mode,ownership,timestamps "$EGRESS_CONFIG" "$backup_dir/egress-config.before"
fi

# 根据已识别的实际防火墙后端保存快照；未知后端时停止，不要假装已备份。
case "${FIREWALL_BACKEND:?set the discovered firewall backend}" in
  nftables) nft list ruleset > "$backup_dir/firewall.before" ;;
  iptables) { iptables-save; ip6tables-save 2>/dev/null; } > "$backup_dir/firewall.before" ;;
  *) echo "Unsupported firewall backend: $FIREWALL_BACKEND" >&2; exit 1 ;;
esac

sha256sum "$backup_dir"/* > "$backup_dir/SHA256SUMS"
chmod 600 "$backup_dir"/*
```

如果还有 systemd 单元、容器编排文件、云端规则导出或证书链需要回滚，也一并纳入清单。备份目录应由管理员可读，不能写入项目工作区；不要用含糊的 `.bak` 文件覆盖既有快照。

### 4. 审查并执行可选安装或升级

需要安装组件时：

1. 选择经过审查的仓库版本、发行版或提交；
2. 下载脚本后先阅读行为，记录版本和 SHA-256；
3. 确认不会清空防火墙、覆盖既有配置、开放不必要端口或上传敏感数据；
4. 对脚本提供的安全策略选项采用最小开放原则；
5. 安装后重新进行服务、监听、配置和防火墙审计。

示意流程：

```bash
install_script=/run/proxy-install.$$
install_url="${REVIEWED_INSTALL_URL:?set the reviewed installer URL}"
curl --fail --silent --show-error --location "$install_url" -o "$install_script"
sha256sum "$install_script"
less "$install_script"
bash "$install_script"
rm -f "$install_script"
```

如果组件已经存在，优先使用其受支持的权威配置入口。不要因为脚本或面板名称与历史记录相同，就跳过版本和路径确认。

### 5. 先处理 Egress，再处理 Relay

#### Egress

- 确认现有上游入站的协议、传输、鉴权、监听和来源范围；
- 能兼容时优先复用，并把来源限制为 Relay 的固定地址或受控安全组；
- 不兼容时新增隔离入站，避免覆盖其他业务；
- 变更后先执行配置检查，再重载服务；
- 记录重载前后的服务状态、监听、日志和规则计数。

#### Relay

- 创建或选择客户端入口；
- 按 Egress 实际协议建立上游出站；
- 为入口和上游分配明确的标识/tag；
- 建立显式路由，防止流量意外走默认出口；
- 生成或引用鉴权材料时，只把客户端需要的公开参数交付给客户端；
- 传输增强项、Mux、UDP/QUIC 等按兼容性和实测开启，不作无条件默认。

#### 地址族转换

若业务要求客户端与 Egress 使用不同地址族，应显式记录并验证：

```text
Client(AF_A)
    ↓
Relay listener(AF_B)
    ↓ Relay upstream(AF_C)
Egress ingress(AF_D)
    ↓
Internet(AF_E)
```

`AF_A` 到 `AF_E` 可以相同，也可以不同。每个箭头都要有实际监听、路由、DNS/字面地址和防火墙证据。IPv6 字面地址放入 URL 时必须使用方括号，例如：

```text
<scheme>://<identity>@[<RELAY_IPV6>]:<RELAY_PORT>?<parameters>#<label>
```

不要仅凭通配 IPv6 监听地址推断 IPv4 一定可用；同时检查系统的双栈绑定策略、云端映射和两套防火墙。

### 6. 配置云端边界和主机网络策略

先创建端口/地址族矩阵，不要直接改规则：


| 层级             | Relay 入口                  | 管理入口                    | Egress 上游                  |
| ------------------ | ----------------------------- | ----------------------------- | ------------------------------ |
| 客户端看到的地址 | `<RELAY_PUBLIC_ENDPOINT>`   | `<RELAY_ADMIN_ENDPOINT>`    | 不直接暴露或按需限制         |
| 云端映射         | `<PUBLIC_TO_RELAY_MAPPING>` | `<PUBLIC_TO_ADMIN_MAPPING>` | `<PUBLIC_TO_EGRESS_MAPPING>` |
| 主机/容器目标    | `<RELAY_TARGET>`            | `<ADMIN_TARGET>`            | `<EGRESS_TARGET>`            |
| 实际监听         | 从`ss` 读取                 | 从`ss` 读取                 | 从`ss` 读取                  |
| 地址族           | `AF_CLIENT→AF_RELAY`       | 管理网络                    | `AF_RELAY→AF_EGRESS`        |
| 允许来源         | 客户端范围或受控来源        | 管理来源                    | Relay 地址或安全组           |

按以下顺序修改：

1. 先在云安全组/NAT/容器映射中准备新路径；
2. 确认当前管理入口仍在允许列表；
3. 在实际生效的主机防火墙后端追加最小规则；
4. 启动或重载监听服务；
5. 从本机、Relay、授权外部客户端逐层验证；
6. 完成协议和出口验收后，再删除旧映射、旧规则和旧入口。

安全策略至少应包含：

- 管理入口仅允许管理来源；
- Egress 上游仅允许 Relay 来源；
- Relay 客户端入口按业务需要开放；
- 保留回环、已建立连接和必要的 ICMP/ICMPv6；
- 其他入站默认拒绝，或保持现有明确且经过审计的策略。

不要混用多个防火墙管理层而不记录优先级和持久化方式。修改前后都要保存规则快照和计数。

### 7. 处理透明转发与协议终止迁移

迁移前先把两种模式写成独立拓扑：

```text
透明模式：
Client → Relay forwarding → Egress protocol endpoint

终止模式：
Client → Relay protocol inbound → Relay route/outbound → Egress protocol inbound
```

从透明模式切换到终止模式时：

- Relay 需要真正的协议入站、鉴权和路由；
- Egress 参数要转换为 Relay 的上游出站配置；
- 旧 DNAT/SNAT、FORWARD 规则和旧入口只能在新链路验收后删除；
- 旧客户端配置、旧订阅项和旧文档要明确标记为废弃；
- 新客户端配置必须使用新入口实际需要的公开参数；
- 如果要停用某个地址族，必须删除该地址族的云端放行、主机规则和旧映射，并从外部验证不可达；
- 如果要保留兼容入口，为每个入口定义独立的端口、协议、来源和验收记录。

端口迁移固定采用“新路径先行”：

```text
新映射/安全组 → 新防火墙规则 → 新监听 → 本地验证 → 外部验证 → 删除旧路径
```

### 8. 分层验证并保留证据

#### A. 配置和服务层

对审计发现的实际服务执行：

```bash
if command -v systemctl >/dev/null 2>&1; then
  systemctl is-active "$DISCOVERED_SERVICE"
  journalctl -u "$DISCOVERED_SERVICE" -n 100 --no-pager
fi
ss -lntup
```

如果组件支持配置检查，使用其实际二进制和实际生效路径：

```bash
# 由审计结果填充二进制、校验子命令和配置参数。
"$PROXY_BINARY" $CHECK_ARGUMENTS
```

不要因为命令名称、配置参数或服务单元在不同发行版/版本之间相似，就直接复用旧命令。

#### B. Relay → Egress 网络层

从 Relay 对 Egress 的实际上游端点执行 TCP 或 UDP 可达性测试：

```bash
nc -vz -w 5 "$EGRESS_ADDRESS" "$EGRESS_PORT"
```

该测试只证明传输层可达。对于 UDP、SNI、Reality、WebSocket 或其他有状态协议，必须使用相同协议的临时客户端继续验证。

#### C. 协议握手层

使用与目标客户端和上游一致的临时配置，优先放在 `/run` 或其他受保护的内存目录；测试结束立即清理。临时配置不得写入项目、历史记录或报告。

若需要通过临时 SOCKS/HTTP 代理验证出口：

```bash
curl --fail --silent --show-error --proxy "$LOCAL_TEMP_PROXY" "$STABLE_EGRESS_CHECK_URL"
```

#### D. 公网入口层

从授权的外部客户端分别测试每个入口的地址族、映射和端口。Windows 客户端可使用 `Test-NetConnection`，Linux/macOS 可使用 `nc`、`curl` 或等效工具；工具只是示例，结果要与监听和云端规则相互印证。

#### E. 端到端出口层

客户端连接成功后访问稳定的出口 IP/网络信息服务，确认结果与预期 Egress 出口一致。记录：

- 客户端地址族；
- Relay 入口地址、端口和协议；
- Relay→Egress 的地址族、端点和协议；
- 实际出口地址；
- 服务状态、监听、云端可达性、协议握手和验证时间；
- 每项结论属于“已验证”“静态确认”还是“尚未验证”。

若链路已在服务器端完成，不要让客户端额外启用前置代理或重复链式代理。只有架构明确要求客户端先连接另一层代理时，才启用客户端前置代理。

### 9. 回滚与停止条件

遇到以下情况立即停止继续试错：

- 配置检查失败或服务重载后退出；
- 最后一个管理入口变得不可达；
- Egress 协议与 Relay 出站不兼容；
- 云端映射、主机监听和外部结果互相矛盾；
- 连续两次修复没有产生新的证据；
- 变更范围开始影响未授权或未纳入计划的服务。

回滚顺序：

1. 保留当前故障证据：服务状态、最近日志、监听、路由、规则计数和云端映射；
2. 如果旧链路仍可用，先恢复旧入口；
3. 恢复本次备份的权威配置、数据库、systemd/容器编排资源；
4. 只恢复本次新增、删除或修改的网络规则，不要无差别清空全表；
5. 重载相关服务并检查实际监听；
6. 验证原有管理入口、原有业务和原有出口；
7. 确认稳定后再清理失败配置或决定是否重做方案。

回滚目录应保留到新链路完成至少一次完整外部验收并确认不再需要回滚。

## Client Delivery

客户端交付应从运行时事实生成，而不是从历史链接复制。交付内容按目标客户端格式选择：

1. 入口地址和地址族；
2. 入口端口和传输方式；
3. 协议、鉴权和公开密钥参数；
4. SNI/Host/路径等确实需要的参数；
5. 节点标签和用途；
6. 是否需要客户端前置代理、Mux 或 UDP 支持；
7. 过期配置、旧端口和废弃链接的状态。

如果目标是 V2rayN，提供可导入的链接或配置片段，并同时给出参数表。IPv6 地址在 URI 中使用方括号；不要把服务器端已经完成的 Relay→Egress 链路再次配置成客户端前置代理。

普通交付文档不得包含：

- SSH 密码或私钥；
- 面板密码、API Token 或数据库原文；
- Reality/证书私钥；
- 未脱敏的完整分享链接；
- 与本次任务无关的主机地址或历史凭据。

## Sanitized Topology Pattern

使用角色占位符表达结果，避免把某次部署的地域、端口和协议误当成默认值：

```text
Client (AF_CLIENT)
    ↓ <RELAY_PUBLIC_ENDPOINT>
Relay: <INBOUND_MODE / PROTOCOL>
    ↓ <RELAY_UPSTREAM_ENDPOINT>
Egress: <INBOUND_PROTOCOL>
    ↓
Internet (expected egress: <EXPECTED_EGRESS_ID>)
```

如果同一 Relay 还有直连、管理或备用入口，应分别画出并注明：

- 是否经过 Egress；
- 使用的地址族；
- 是否为协议终止或透明转发；
- 允许来源；
- 是否属于生产入口、备用入口或已废弃入口。

## Common Failure Modes


| 现象                          | 优先检查                                                     | 避免的错误动作                 |
| ------------------------------- | -------------------------------------------------------------- | -------------------------------- |
| 本机监听但公网不通            | 云端映射、安全组、容器端口、主机防火墙和地址族               | 反复重装代理组件               |
| 管理面板可达但代理不监听      | 权威配置、生成配置、配置校验、端口冲突和日志                 | 猜 API endpoint 或盲目重启     |
| Relay 到 Egress 不通          | Egress 来源 ACL、上游端口、路由、地址族和回程                | 直接开放 Egress 全部端口       |
| TCP 可达但协议握手失败        | 协议、鉴权、传输、公开密钥、SNI/Host、路径和时间             | 只修改客户端地址               |
| 握手成功但出口错误            | 路由/tag、默认出站、上游实际使用情况和出口验证               | 把“连接到 Relay”当成链路完成 |
| 一种地址族正常、另一种失败    | DNS、云端规则、对应防火墙、监听绑定、系统双栈策略和 URI 格式 | 只检查另一种地址族             |
| 端口迁移后旧配置失效          | 新旧映射、实际监听、旧 NAT、客户端配置生命周期               | 同时保留冲突的旧 NAT 和新入站  |
| 管理 API 返回`404` 或字段不符 | 组件版本、前端调用、服务日志和真实配置来源                   | 不停猜接口名并写入半成品       |
| 安装器询问是否重置网络策略    | 脚本行为、现有规则快照和恢复方案                             | 选择全清空/全放行              |
| 静态参数正确但出口未改变      | 路由是否命中、临时客户端是否使用了目标出站、测试是否经过代理 | 只凭配置文本宣称端到端成功     |

## Delivery Package

每次完成后交付以下脱敏产物：

1. **角色拓扑图**：Client → Relay → Egress → Internet，标注每段地址族、协议和端点；
2. **端口/映射矩阵**：公网端点、云端目标、主机监听、允许来源和实际生效策略；
3. **变更记录**：时间、权威文件或资源、修改前后摘要、备份目录和服务重载结果；
4. **客户端参数表**：目标客户端、入口、协议、传输、公开参数和导入方式；
5. **分层验收证据**：配置校验、服务状态、监听、网络可达性、协议握手和最终出口；
6. **问题定位**：云端、主机防火墙、服务监听、协议、路由或出口中的具体边界；
7. **回滚说明**：恢复哪些权威资源、网络规则和客户端配置；
8. **安全加固建议**：SSH 密钥、管理来源限制、强面板凭据、HTTPS、组件更新和最小开放端口。

明确区分：

- **已验证**：有本次命令或外部测试证据；
- **静态确认**：从配置或规则读到，但尚未做真实流量测试；
- **尚未验证**：缺少外部条件、客户端或云平台操作，不能宣称完成。

## Final Checklist

- [ ]  仅连接已授权的目标节点，未扫描邻近目标或猜测凭据。
- [ ]  已明确 Relay、Egress、管理面和数据面的角色边界。
- [ ]  已从运行环境发现服务名、配置路径、监听、路由和防火墙后端。
- [ ]  已核对公网端点、云端映射、宿主机/容器目标和实际监听。
- [ ]  已在变更前创建权限正确、带哈希的远端回滚快照。
- [ ]  已选择协议终止、透明转发或混合模式，并记录选择依据。
- [ ]  Egress 上游协议、鉴权、来源限制和地址族已与 Relay 出站匹配。
- [ ]  未覆盖未授权或未纳入范围的既有业务。
- [ ]  IPv4、IPv6、DNS、云端策略、主机策略和外部可达性分别验证。
- [ ]  新路径已先准备、监听并完成协议和出口验收，再处理旧路径。
- [ ]  客户端配置由当前事实生成；IPv6 URI 使用方括号。
- [ ]  未误启用客户端前置代理；Mux、UDP/QUIC 等增强项有兼容性证据。
- [ ]  变更记录已脱敏，并区分已验证、静态确认和未验证。
- [ ]  已交付拓扑、矩阵、验收证据、排错结论和回滚方法。
