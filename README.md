# janus 知识库

工作与搞机过程中遇到的问题、方案与知识点的汇总记录。按主题分目录。

## 目录

| 目录 | 内容 | 文件 |
|---|---|---|
| `database/` | 数据库 | ORM 知识点、GORM 知识点（连接池、事务、N+1 查询、Hooks、软删除等坑） |
| `distributed/` | 分布式 | Nacos 注册/配置中心知识点、常见面试题 |
| `hardware/` | 硬件 | Intel AX200 5G AP：self-managed 监管根因、MCC 注入驱动补丁、hostapd 配置、OpenRC 服务、性能数据 |
| `network/` | 网络 | Gentoo (OpenRC) 有线口热点配置（NAT/iptables/mihomo 共存）、TLS handshake timeout 与路径 MTU 问题 |
| `os/` | 系统 | Gentoo Ebuild 编写指南（eclass、RDEPEND、USE 依赖、QA 等） |
| `security/` | 权限 | 权限模型汇总（RBAC / 数据权限 / ABAC / ACL / RLS）：原理、表设计、实现与踩坑 |

## 使用

按主题检索，比如：

- "开 5G 热点" → `hardware/ax200.md`
- "Gentoo 写 ebuild" → `os/gentoo/ebuild.md`
- "热点和 mihomo 代理冲突" → `network/hotspot-gentoo.md`
- "TLS 握手卡住" → `network/mut.md`

## 约定

- 以"根因 → 方案 → 排查命令"结构组织，记录踩过的坑与失败尝试，避免重复踩。
- 示例中的密码、SSID、内网 IP 均已脱敏（占位符），不含真实敏感信息。
