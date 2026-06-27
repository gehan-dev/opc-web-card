# Phase 4 & 5 — ECS 创建 & 连通性测试

## 4.1 解析镜像 ID

**先获取最新的 Alibaba Cloud Linux 4 镜像 ID：**

```bash
aliyun ecs DescribeImageFromFamily \
  --ImageFamily "acs:alibaba_cloud_linux_4_lts_x64" \
  --RegionId cn-shanghai \
  --profile opc-deploy
```

**从返回结果中提取 `ImageId`：**
```bash
# 示例返回
{
  "Image": {
    "ImageId": "aliyun_4_x64_20260601.vhd",
    "ImageFamily": "acs:alibaba_cloud_linux_4_lts_x64",
    "OSName": "Alibaba Cloud Linux 4.0 64位",
    ...
  }
}
```

**将 ImageId 记录用于后续创建。**

---

## 4.2 创建 VPC & VSwitch

### 创建 VPC

```bash
aliyun vpc CreateVpc \
  --RegionId cn-shanghai \
  --VpcName "opc-web-card-vpc" \
  --CidrBlock "172.16.0.0/16" \
  --profile opc-deploy
```

**记录返回的 `VpcId`。**

### 等待 VPC 可用

```bash
# 等待 5 秒让 VPC 初始化
sleep 5

# 轮询 VPC 状态（通常 5-10 秒内变为 Available）
aliyun vpc DescribeVpcs \
  --RegionId cn-shanghai \
  --VpcId <VpcId> \
  --profile opc-deploy
```

等待 `Status` 变为 `Available`。如果仍为 `Pending`，再等 5 秒重试（最多 3 次）。

### 创建 VSwitch

```bash
aliyun vpc CreateVSwitch \
  --RegionId cn-shanghai \
  --VpcId <VpcId> \
  --ZoneId cn-shanghai-b \
  --VSwitchName "opc-web-card-vsw" \
  --CidrBlock "172.16.0.0/24" \
  --profile opc-deploy
```

**记录返回的 `VSwitchId`。**

> 💡 如果 cn-shanghai-b 可用区库存不足，可尝试 cn-shanghai-d 或 cn-shanghai-g。

---

## 4.3 创建安全组

```bash
aliyun ecs CreateSecurityGroup \
  --RegionId cn-shanghai \
  --VpcId <VpcId> \
  --SecurityGroupName "opc-web-card-sg" \
  --Description "OPC Web Card Security Group - Shanghai Event 20260628" \
  --profile opc-deploy
```

**记录返回的 `SecurityGroupId`。**

### 配置安全组规则

开放 HTTP (80)、HTTPS (443) 和 SSH (22) 端口：

```bash
# HTTP
aliyun ecs AuthorizeSecurityGroup \
  --RegionId cn-shanghai \
  --SecurityGroupId <SecurityGroupId> \
  --IpProtocol tcp \
  --PortRange 80/80 \
  --SourceCidrIp 0.0.0.0/0 \
  --Description "HTTP" \
  --profile opc-deploy

# HTTPS
aliyun ecs AuthorizeSecurityGroup \
  --RegionId cn-shanghai \
  --SecurityGroupId <SecurityGroupId> \
  --IpProtocol tcp \
  --PortRange 443/443 \
  --SourceCidrIp 0.0.0.0/0 \
  --Description "HTTPS" \
  --profile opc-deploy

# SSH (仅限当前 IP)
aliyun ecs AuthorizeSecurityGroup \
  --RegionId cn-shanghai \
  --SecurityGroupId <SecurityGroupId> \
  --IpProtocol tcp \
  --PortRange 22/22 \
  --SourceCidrIp "$(curl -4 -s ifconfig.me)/32" \
  --Description "SSH from current IP" \
  --profile opc-deploy
```

> ⚠️ **SSH 安全提示：** SSH 端口仅对用户当前公网 IP 开放。如果用户回家后需要 SSH 访问，需要更新安全组规则（替换为新 IP）。活动现场多人可能共享同一出口 IP，这是正常的。

---

## 4.4 创建密钥对

```bash
mkdir -p ~/.ssh

aliyun ecs CreateKeyPair \
  --RegionId cn-shanghai \
  --KeyPairName "opc-web-card-key" \
  --profile opc-deploy \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['PrivateKeyBody'])" \
  > ~/.ssh/opc-web-card-key.pem
```

> 💡 私钥在 API 返回的 `PrivateKeyBody` 字段中，上面的命令直接提取并写入文件。
> 如果用户没有 python3，可改用 `jq -r '.PrivateKeyBody'` 替代。

**设置密钥权限：**
```bash
chmod 600 ~/.ssh/opc-web-card-key.pem
```

**验证密钥文件存在：**
```bash
ls -la ~/.ssh/opc-web-card-key.pem
```

> ⚠️ 私钥文件直接落盘，绝不在对话中展示完整内容。仅向用户确认文件已保存。

---

## 4.5 创建 ECS 实例

```bash
aliyun ecs RunInstances \
  --RegionId cn-shanghai \
  --ImageId <步骤4.1获取的ImageId> \
  --InstanceType ecs.e-c1m1.large \
  --SecurityGroupId <SecurityGroupId> \
  --VSwitchId <VSwitchId> \
  --SystemDisk.Category cloud_essd_entry \
  --SystemDisk.Size 40 \
  --InstanceChargeType PostPaid \
  --InternetChargeType PayByTraffic \
  --InternetMaxBandwidthOut 100 \
  --InstanceName "opc-web-card" \
  --HostName "opc-web-card" \
  --KeyPairName "opc-web-card-key" \
  --Amount 1 \
  --profile opc-deploy \
  --user-agent "AlibabaCloud-Agent-Skills/opc-web-card/{SESSION_ID}"
```

**记录返回的 `InstanceIdSets.InstanceIdSet[0]`（即实例 ID）。**

---

## 4.6 等待实例运行

```bash
# 轮询实例状态
aliyun ecs DescribeInstances \
  --RegionId cn-shanghai \
  --InstanceIds '["<InstanceId>"]' \
  --profile opc-deploy
```

等待 `Status` 变为 `Running`。

**获取公网 IP：**

从 `DescribeInstances` 返回中提取：
- `PublicIpAddress.IpAddress[0]` — 这就是用户的网络名片访问地址

---

## 4.7 连通性测试

### SSH 连通性

```bash
ssh -i ~/.ssh/opc-web-card-key.pem \
  -o StrictHostKeyChecking=no \
  -o ConnectTimeout=10 \
  root@<公网IP> "echo 'SSH OK'"
```

### HTTP 连通性（在 ECS 上启动测试页面）

```bash
ssh -i ~/.ssh/opc-web-card-key.pem root@<公网IP> << 'EOF'
# Alibaba Cloud Linux 4 基于 RHEL8，使用 dnf
dnf install -y nginx
systemctl start nginx
systemctl enable nginx
echo '<h1>🎉 网络名片准备就绪！</h1><p>OPC Shanghai Event 2026-06-28</p>' > /usr/share/nginx/html/index.html
EOF
```

**从本地验证 HTTP：**
```bash
curl -s -o /dev/null -w "%{http_code}" http://<公网IP>
```

期望返回 `200`。

---

## Phase 5 — 交付

**所有测试通过后，告知用户：**

> ✅ 你的云服务器已经创建好了！
>
> 📋 **服务器信息：**
> - 公网 IP：`<公网IP>`
> - 地域：上海
> - 配置：2 核 2G / 40G ESSD Entry / 100Mbps
> - 系统：Alibaba Cloud Linux 4
> - 费用：按量 ¥0.12316/小时
>
> 🌐 **访问地址：** http://<公网IP>
>
> 现在你可以通过浏览器打开上面的地址看看，会看到一个测试页面。
>
> 随时准备部署你的「网络名片」——你跟我说，我帮你。只需要告诉我你的网站代码在哪里（GitHub 仓库、本地文件夹等），我来帮你部署上去。

---

## 错误处理

| 错误 | 原因 | 处理 |
|------|------|------|
| `InvalidInstanceType.NotSupported` | 可用区不支持该实例规格 | 换可用区重试 |
| `OperationDenied.NoStock` | 库存不足 | 换可用区（cn-shanghai-d/g/l） |
| `InvalidSystemDiskCategory.ValueNotSupported` | ESSD Entry 不可用 | 备选 cloud_essd（PL0） |
| `QuotaExceeded.Vpc` | VPC 配额已满 | 列已有 VPC，复用用户已有的 |
| `Forbidden.RAM` | RAM 权限不足 | 检查角色策略是否正确绑定 |
| `InvalidAccountStatus.NotEnoughBalance` | 余额不足 | 回到 Phase 1.4 引导充值 |

**可用区 fallback 顺序：** cn-shanghai-b → cn-shanghai-d → cn-shanghai-g → cn-shanghai-l
