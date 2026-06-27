# Phase 7 — 生命周期管理

## 7.1 释放 ECS 资源

**触发条件：** 用户说不要云服务器了 / 不需要网络名片了 / 想释放资源

### 7.1.1 二次确认（铁律 #4 — 必须执行！）

**第一次确认话术：**
> 确认一下，你要释放云服务器吗？释放后：
> - ❌ 你的网站将无法通过互联网被任何人访问
> - ❌ 你的「网络名片」将不再印刷 & 发放
> - ❌ 服务器上的所有数据将永久删除，无法恢复
>
> 确定要释放吗？

**等待用户确认。**

**第二次确认话术（用户第一次说"确定"后）：**
> ⚠️ 再确认一次：释放后数据无法恢复，网站将从互联网消失。你确认知晓这些风险，并确定要释放吗？

**等待用户再次确认后才执行。**

---

### 7.1.2 执行释放

**释放顺序（反向依赖顺序）：**

```bash
# 1. 释放 ECS 实例
aliyun ecs DeleteInstance \
  --InstanceId <InstanceId> \
  --Force true \
  --profile opc-deploy

# 2. 删除密钥对
aliyun ecs DeleteKeyPairs \
  --RegionId cn-shanghai \
  --KeyPairNames '["opc-web-card-key"]' \
  --profile opc-deploy

# 3. 删除安全组
aliyun ecs DeleteSecurityGroup \
  --RegionId cn-shanghai \
  --SecurityGroupId <SecurityGroupId> \
  --profile opc-deploy

# 4. 删除 VSwitch
aliyun vpc DeleteVSwitch \
  --VSwitchId <VSwitchId> \
  --profile opc-deploy

# 5. 删除 VPC
aliyun vpc DeleteVpc \
  --VpcId <VpcId> \
  --profile opc-deploy
```

> 💡 VSwitch/VPC 删除可能需要等几秒（依赖资源释放），如果报错"有依赖资源"，等 10 秒重试。

**释放完成话术：**
> ✅ 所有资源已释放完毕！
> - 从现在起不再产生任何费用
> - 24 小时后你可以在阿里云控制台提现余额
> - 提现入口：https://billing-cost.console.aliyun.com/fortune/fund-management/withdraw
>
> 如果以后又想创建网络名片了，随时跟我说！

---

### 7.1.3 清理本地凭证（可选建议）

**建议用户执行：**
> 你也可以清理一下本地的配置文件（可选）：
> ```bash
> # 删除密钥文件
> rm -f ~/.ssh/opc-web-card-key.pem
> # 删除 CLI profile（从 config.json 中移除 opc-deploy 配置）
> aliyun configure delete --profile opc-deploy
> ```

---

## 7.2 按量付费转包年包月

**触发条件：** 用户对网络名片满意，想让网站持续在线

### 7.2.1 询价（铁律 #6 — 必须实际询价！）

**⚠️ 不可硬编码价格！必须通过 DescribePrice 获取真实价格。**

```bash
aliyun ecs DescribePrice \
  --RegionId cn-shanghai \
  --ResourceType instance \
  --InstanceType ecs.e-c1m1.large \
  --SystemDisk.Category cloud_essd_entry \
  --SystemDisk.Size 40 \
  --InternetChargeType PayByBandwidth \
  --InternetMaxBandwidthOut 3 \
  --PriceUnit Year \
  --Period 1 \
  --profile opc-deploy \
  --user-agent "AlibabaCloud-Agent-Skills/opc-web-card/{SESSION_ID}"
```

**从返回中提取价格信息：**
- `PriceInfo.Price.TradePrice` — 实际成交价
- `PriceInfo.Price.OriginalPrice` — 原价
- `PriceInfo.Price.DiscountPrice` — 折扣金额

---

### 7.2.2 向用户确认（铁律 #3 — 扣款前必须确认！）

**确认话术模板：**
> 我查了一下包年的价格：
>
> 📋 **转包年方案：**
> - 配置：2核 2G / 40G ESSD Entry / 3Mbps 固定带宽
> - 变化：网络从按流量计费（100Mbps 峰值）→ 固定带宽（3Mbps），日常浏览完全够用
> - 周期：1 年
> - 价格：¥<TradePrice>（原价 ¥<OriginalPrice>，优惠 ¥<DiscountPrice>）
>
> ⚠️ **确认扣款：** 这将从你的阿里云账户扣除 ¥<TradePrice>。确认要转为包年吗？

> 💡 该优惠价每个用户（新老用户均可）可享受一次。如果你之前已经用过这个优惠，实际价格可能更高，以 DescribePrice 返回为准。

**必须等用户明确回复"确认/好的/可以"后才执行！**

---

### 7.2.3 执行转换（两步操作）

**Step 1 — 更改网络计费方式：**

先将网络从「按使用流量（PayByTraffic）」改为「按固定带宽（PayByBandwidth）3Mbps」：

```bash
aliyun ecs ModifyInstanceNetworkSpec \
  --InstanceId "<InstanceId>" \
  --NetworkChargeType PayByBandwidth \
  --InternetMaxBandwidthOut 3 \
  --profile opc-deploy \
  --user-agent "AlibabaCloud-Agent-Skills/opc-web-card/{SESSION_ID}"
```

> 💡 这一步将网络计费从按流量改为固定带宽，使后续转包年时能命中 99 元优惠价格。

**Step 2 — 转包年：**

```bash
aliyun ecs ModifyInstanceChargeType \
  --RegionId cn-shanghai \
  --InstanceIds '["<InstanceId>"]' \
  --InstanceChargeType PrePaid \
  --Period 1 \
  --PeriodUnit Year \
  --AutoPay true \
  --profile opc-deploy \
  --user-agent "AlibabaCloud-Agent-Skills/opc-web-card/{SESSION_ID}"
```

> ⚠️ `ModifyInstanceChargeType` 不支持 `InternetChargeType` 参数，必须先通过 Step 1 更改网络计费方式。

**转换成功话术：**
> ✅ 已成功转为包年！
>
> 📋 **你的网络名片状态：**
> - 访问地址：http://<公网IP>
> - 有效期：一年（到 2027 年 6 月）
> - 到期前会收到续费提醒
>
> 你的网络名片现在是持续在线的了！🎉
>
> **后续建议：**
> - 如果想要专属域名（比如 yourname.com），可以去注册一个域名并绑定
> - 域名注册：https://wanwang.aliyun.com/
> - 如果要备案（.cn/.com 等国内访问需要），可以在阿里云 ICP 备案系统操作

---

### 7.2.4 转换失败处理

| 错误 | 原因 | 处理 |
|------|------|------|
| `InvalidAccountStatus.NotEnoughBalance` | 余额不足以支付包年费用 | 提示用户充值后重试 |
| `InvalidInstanceId.NotFound` | 实例不存在 | 确认实例 ID 正确 |
| `OperationDenied` | 实例状态不允许转换 | 确保实例处于 Running 状态 |
| `LastOrderProcessing` | 上一单还在处理 | 等待几分钟后重试 |
