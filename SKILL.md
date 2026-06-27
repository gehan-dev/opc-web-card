---
name: opc-web-card
description: "2026年6月28日上海OPC专场活动——帮助一人公司创建'网络名片'（个人官网/作品集/落地页）并部署到阿里云ECS。当用户提及网络名片、活动部署、上海专场、OPC建站、ECS部署网站时使用。涵盖：账号准备、CLI安装、RAM临时凭证配置、ECS创建、连通性测试、资源释放、按量转包年。仅适用于2026年6月28日下午上海专场活动。"
description_zh: "2026年6月28日上海OPC专场活动——帮助一人公司创建'网络名片'（个人官网/作品集/落地页）并部署到阿里云ECS。涵盖账号准备、CLI安装、RAM临时凭证、ECS创建、连通性测试、资源释放、按量转包年。"
version: "1.0.0"
metadata:
  author: gehan
  event_date: "2026-06-28"
  event_location: "上海"
---

# OPC 网络名片 — 上海专场活动 Skill

帮助到场的 OPC（one person company）快速完成"网络名片"搭建 & 部署到阿里云 ECS。

**适用范围：仅限 2026年6月28日下午上海专场活动。**

All user-facing output MUST be in **Chinese (zh-CN)**.

---

## When to Use This Skill

- 用户说要创建网络名片 / 个人官网 / 作品集 / 落地页
- 用户在上海专场活动中需要部署网站到 ECS
- 用户问"怎么让别人看到我的网站"
- 用户提及本次活动、OPC 建站

**与其他 skill 的区别：**
- `alibabacloud-opc-advisor`：SKU 选型推荐（本 skill 配置已固定，不需要走 advisor）
- `opc-cloud-deploy`：通用部署 skill（本 skill 是活动专用精简版）

---

## Concept Introduction (Phase 0)

当用户首次进入对话时，用以下类比帮助理解：

> 当你把产品做出来后，下一步就是让别人看见。一个官网、一个落地页、一个作品集，都是你的「网络名片」。云服务器负责稳定承载这些内容，域名负责让用户找到你。先把展示窗口搭起来，你的产品才更容易被看见、被记住，也更容易获得第一批用户。

**三档名片类比：**

| 档位 | 配置 | 类比 |
|------|------|------|
| 简洁名片 | 云服务器 + 公网IP访问 | 今天我们要做的 ✅ |
| 专业名片 | 云服务器 + 域名访问 | 可选，活动中简单引导 |
| 精致名片 | 云服务器 + 域名 + ESA加速 | 可选，活动中简单引导 |

**今天的目标：** 先搭好「简洁名片」，用公网IP让别人能访问你的网站。域名 & 备案不是必要步骤，有兴趣可以之后自行完成。

---

## User Scenario Detection

```
用户当前状态？
├── 场景 A：已有 QoderWork CN → 已有阿里云账号，确认实名即可
├── 场景 B：有其他 AI 助手（WorkBuddy/Codex 等），不打算装 QoderWork CN
│   ├── 有阿里云账号 → 确认实名
│   └── 无阿里云账号 → 引导注册 & 实名
└── 场景 C：没有任何工具（现场引导下载，skill 不关注）
```

**Entry Probe（中文）：**
> "你现在用的是哪个 AI 助手？已经有阿里云账号了吗？"

---

## Master Workflow

```
Phase 0 — 概念介绍（网络名片三档类比）
    │
Phase 1 — 账号准备（注册 / 实名 / 余额 ≥ 100元）
    │         详见 → references/account-prep.md
    │
Phase 2 — 安全凭证配置（RAM 子账号 + 角色 + STS 临时凭证）
    │         详见 → references/ram-credential-setup.md
    │
Phase 3 — CLI 安装 & 配置（skill 必须帮用户安装！）
    │         详见 → references/ram-credential-setup.md §CLI安装
    │
Phase 4 — ECS 创建 & 连通性测试
    │         详见 → references/ecs-deploy.md
    │
Phase 5 — 交付（公网IP + 部署就绪）
    │
Phase 6 — [可选] 网站部署（用户说"帮我部署"时执行）
    │
Phase 7 — [可选] 生命周期管理（释放 / 转包年）
              详见 → references/ecs-lifecycle.md
```

---

## ECS Fixed Configuration

本次活动 ECS 配置**已固定**，无需询问用户选型：

```yaml
ecs_config:
  instance_type: ecs.e-c1m1.large    # 2 vCPU 2 GiB
  system_disk:
    category: cloud_essd_entry        # ESSD Entry
    size_gb: 40
  internet:
    charge_type: PayByTraffic
    max_bandwidth_out: 100            # 100 Mbps
  instance_charge_type: PostPaid      # 按量付费
  region_id: cn-shanghai
  image:
    family: "acs:alibaba_cloud_linux_4_lts_x64"
    resolve_cmd: "aliyun ecs DescribeImageFromFamily --ImageFamily acs:alibaba_cloud_linux_4_lts_x64 --RegionId cn-shanghai"
```

**费用说明（告知用户）：** 按量付费，每小时约 ¥0.12316。

---

## Iron Rules (Critical Constraints)

| # | Rule | Severity |
|---|------|----------|
| 1 | **AKSK 绝不进对话！** 不可跟用户要 AccessKey/SecretKey。必须用 RamRoleArn（STS 临时凭证） | 🔴 CRITICAL |
| 2 | **CLI 必须帮用户安装！** 千万不可让用户自行安装 | 🔴 CRITICAL |
| 3 | **扣款前必须用户确认！** 不可静默扣款（按量转包年场景） | 🔴 CRITICAL |
| 4 | **释放前二次确认 + 风险告知** — 释放后网站不可访问 | 🔴 CRITICAL |
| 5 | **余额检查 ≥ 100 元** — 按量付费开通前提 | HIGH |
| 6 | **转包年前实际询价** — 不可硬编码价格，必须 DescribePrice 确认 | HIGH |
| 7 | **跑腿员比喻** — RAM 凭证解释必须用"跑腿员穿制服"比喻 | MEDIUM |
| 8 | **域名&备案非必要** — 今天只做简洁名片（公网IP），域名可提及但不强制 | MEDIUM |
| 9 | **活动限定** — 明确告知用户本 skill 仅适用于 2026/6/28 下午上海专场 | LOW |

---

## Cross-Tool Compatibility (Scenario B)

对于使用其他 AI 助手（WorkBuddy / Codex / 其他）的用户：

1. 本 skill 以 Markdown 文件形式存在，可被任何支持 skill/prompt 的 AI 助手读取
2. 核心流程不依赖 QoderWork CN 特有功能
3. 所有 CLI 命令均为标准 shell 命令，跨平台兼容
4. 关键：确保用户的 AI 助手可以执行终端命令

---

## References

| Reference | File | Contains |
|---|---|---|
| **账号准备** | [account-prep.md](references/account-prep.md) | 注册、实名、余额检查、充值引导 |
| **RAM 凭证配置** | [ram-credential-setup.md](references/ram-credential-setup.md) | 子账号、角色、STS 临时凭证、CLI 安装 |
| **ECS 部署** | [ecs-deploy.md](references/ecs-deploy.md) | ECS 创建、安全组、连通性测试 |
| **生命周期管理** | [ecs-lifecycle.md](references/ecs-lifecycle.md) | 释放资源、按量转包年 |

---

## Example Interaction

**User:** "我想搭建我的网络名片"

**Agent (abbreviated):**
> 🎉 欢迎来到上海专场！让我帮你搭建「网络名片」。
>
> [网络名片概念介绍 + 三档类比]
>
> 先确认几个前提：
> 1. 你用的是哪个 AI 助手？
> 2. 有阿里云账号吗？
>
> 确认后我会一步一步带你完成，全程帮你操作，你只需要确认就好。

---

## Observability

When executing `aliyun` CLI commands:

- **User-Agent:** `AlibabaCloud-Agent-Skills/opc-web-card/{SESSION_ID}`
- **Session-ID:** UUID generated once per conversation session

```bash
aliyun ecs RunInstances \
  --RegionId cn-shanghai \
  --user-agent "AlibabaCloud-Agent-Skills/opc-web-card/{SESSION_ID}"
```
