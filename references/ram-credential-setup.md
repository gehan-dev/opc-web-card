# Phase 2 & 3 — RAM 安全凭证 & CLI 安装配置

## 概念引入（跑腿员比喻）

**必须使用以下比喻向用户解释（铁律 #7）：**

> 为了你的账户安全，我们用「跑腿员穿制服」的方式来部署：
>
> 1. **跑腿员**（子账号 `ecs-deploy`）：只会请求穿制服，没有任何业务权限
> 2. **制服**（角色 `ecs-deploy-role`）：临时穿一小时，挂着这次部署需要的最小权限
> 3. **工牌**（永久 AccessKey）：存在你本地，即便泄露，攻击者也只能拿到限时一小时的临时通行证
>
> 这样做的好处：即使工牌丢了，小偷也只能用一小时，而且只能做部署相关的事。

---

## Phase 2 — RAM 配置（在阿里云控制台操作）

### 2.1 创建 RAM 子账号

**引导用户在控制台操作：**

> 现在我需要你在阿里云控制台帮我创建一个「跑腿员」。打开这个链接：

**RAM 控制台：** https://ram.console.aliyun.com/users

**操作步骤（引导用户）：**

1. 点击「创建用户」
2. 登录名称填：`ecs-deploy`
3. 显示名称填：`ECS部署专用`
4. **勾选** "OpenAPI 调用访问" （这会生成 AccessKey）
5. **不勾选** "控制台访问"
6. 点击确定

**⚠️ 重要提示（告知用户）：**
> 创建成功后，页面会显示 AccessKey ID 和 AccessKey Secret。**请把这两个值保存到本地安全的地方**（比如备忘录），关闭页面后就看不到了。
>
> 🔒 **注意：不要把 AccessKey 发给我或任何人！** 后面配置的时候，你在自己电脑的终端里输入就好。

---

### 2.1b 给子账号授予 STS 扮演权限

**⚠️ 必须步骤！否则后续凭证配置会失败。**

> 跑腿员需要有"请求穿制服"的资格。我们给他加一个小权限。

**操作步骤：**
1. 在 RAM 用户列表中，点击刚创建的 `ecs-deploy`
2. 点击「添加权限」
3. 搜索 `AliyunSTSAssumeRoleAccess` 并勾选
4. 点击确定

> 💡 这个权限只允许"请求穿制服"，不包含任何业务能力。真正的能力在制服（角色）上。

---

### 2.2 创建 RAM 角色

**引导用户在控制台操作：**

> 跑腿员创建好了，现在我们来准备「制服」。打开这个链接：

**RAM 角色控制台：** https://ram.console.aliyun.com/roles

**操作步骤：**

1. 点击「创建角色」
2. 信任主体类型选：**云账号**
3. 信任主体名称选：**当前云账号**，点击「确定」
4. 角色名称填：`ecs-deploy-role`
5. 点击完成

---

### 2.3 给角色授权

**创建角色后，继续在角色详情页操作：**

1. 点击刚创建的 `ecs-deploy-role` 进入详情
2. 点击「新增授权」
3. 搜索并添加以下系统策略：
   - `AliyunECSFullAccess`（ECS 完全访问）
   - `AliyunVPCFullAccess`（VPC 完全访问）

> 💡 这些权限是给「制服」的，跑腿员穿上制服后才有这些能力，脱了制服什么都做不了。

---

### 2.4 配置信任策略（允许子账号扮演角色）

**在角色详情页 → 信任策略管理 → 编辑信任策略：**

> 现在我需要你修改信任策略，让刚才的跑腿员有权限"穿上制服"。在角色详情页找到「信任策略」标签，点击编辑：

```json
{
  "Statement": [
    {
      "Action": "sts:AssumeRole",
      "Effect": "Allow",
      "Principal": {
        "RAM": [
          "acs:ram::<你的阿里云账号ID>:user/ecs-deploy"
        ]
      }
    }
  ],
  "Version": "1"
}
```

**获取账号 ID 的方法：**
> 你的阿里云账号 ID 可以在右上角头像处看到，是一串数字。或者打开：https://account.console.aliyun.com/v2/#/basic-info

---

### 2.5 收集必要信息

**需要用户提供的信息（用于后续 CLI 配置）：**

| 信息 | 说明 | 如何获取 |
|------|------|----------|
| AccessKey ID | 步骤 2.1 创建子账号时保存的 | 用户本地保存的 |
| AccessKey Secret | 步骤 2.1 创建子账号时保存的 | 用户本地保存的 |
| Role ARN | 角色的资源标识 | 角色详情页顶部 `acs:ram::xxx:role/ecs-deploy-role` |

**话术：**
> 好的，RAM 配置完成！现在需要你帮我找到以下信息（都不用发给我，等下在终端里自己输入就好）：
> 1. 之前保存的 AccessKey ID 和 Secret
> 2. 角色的 ARN（在角色详情页顶部能看到，格式是 `acs:ram::你的账号ID:role/ecs-deploy-role`）

---

## Phase 3 — CLI 安装 & 凭证配置

### 3.1 安装阿里云 CLI

**⚠️ 铁律 #2：必须帮用户安装，不可让用户自行安装！**

**检测用户操作系统后，执行对应安装命令：**

#### macOS (Homebrew)

```bash
brew install aliyun-cli
```

如果没有 Homebrew：
```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
brew install aliyun-cli
```

#### macOS (curl 直接安装)

```bash
# 通过 GitHub Releases API 动态获取最新下载 URL
ARCH=$(uname -m)
if [ "$ARCH" = "arm64" ]; then
  PATTERN="macosx.*arm64"
else
  PATTERN="macosx.*amd64"
fi
CLI_URL=$(curl -fsSL https://api.github.com/repos/aliyun/aliyun-cli/releases/latest \
  | python3 -c "import sys,json; assets=json.load(sys.stdin)['assets']; print([a['browser_download_url'] for a in assets if '${PATTERN}'.replace('.*','') in a['name'] and a['name'].endswith('.tgz')][0])" 2>/dev/null)

# 如果 API 获取失败，使用固定版本 fallback
if [ -z "$CLI_URL" ]; then
  if [ "$ARCH" = "arm64" ]; then
    CLI_URL="https://github.com/aliyun/aliyun-cli/releases/download/v3.4.2/aliyun-cli-macosx-3.4.2-arm64.tgz"
  else
    CLI_URL="https://github.com/aliyun/aliyun-cli/releases/download/v3.4.2/aliyun-cli-macosx-3.4.2-amd64.tgz"
  fi
fi

mkdir -p ~/bin
curl -fsSL "$CLI_URL" -o /tmp/aliyun-cli.tgz
tar xzf /tmp/aliyun-cli.tgz -C ~/bin/
chmod +x ~/bin/aliyun
```

> 💡 安装到 `~/bin/` 避免需要 sudo 权限。如果 `~/bin` 不在 PATH 中，后续命令使用 `~/bin/aliyun` 完整路径即可。

#### Linux

```bash
ARCH=$(uname -m)
if [ "$ARCH" = "aarch64" ]; then
  PATTERN="linux.*arm64"
else
  PATTERN="linux.*amd64"
fi
CLI_URL=$(curl -fsSL https://api.github.com/repos/aliyun/aliyun-cli/releases/latest \
  | python3 -c "import sys,json; assets=json.load(sys.stdin)['assets']; print([a['browser_download_url'] for a in assets if '${PATTERN}'.replace('.*','') in a['name'] and a['name'].endswith('.tgz')][0])" 2>/dev/null)

if [ -z "$CLI_URL" ]; then
  CLI_URL="https://github.com/aliyun/aliyun-cli/releases/download/v3.4.2/aliyun-cli-linux-3.4.2-amd64.tgz"
fi

mkdir -p ~/bin
curl -fsSL "$CLI_URL" -o /tmp/aliyun-cli.tgz
tar xzf /tmp/aliyun-cli.tgz -C ~/bin/
chmod +x ~/bin/aliyun
```

#### Windows (PowerShell)

```powershell
$release = Invoke-RestMethod -Uri "https://api.github.com/repos/aliyun/aliyun-cli/releases/latest"
$asset = $release.assets | Where-Object { $_.name -match "windows.*amd64.*\.zip$" } | Select-Object -First 1
Invoke-WebRequest -Uri $asset.browser_download_url -OutFile "$env:TEMP\aliyun-cli.zip"
Expand-Archive "$env:TEMP\aliyun-cli.zip" -DestinationPath "$env:USERPROFILE\aliyun-cli" -Force
$env:PATH += ";$env:USERPROFILE\aliyun-cli"
[Environment]::SetEnvironmentVariable("PATH", $env:PATH, "User")
```

**安装后验证：**
```bash
aliyun version
```

---

### 3.2 配置 RamRoleArn 凭证

**⚠️ 铁律 #1：AKSK 绝不进对话！以下命令由用户在自己终端执行！**

**引导话术：**
> 现在请在你的终端里执行以下命令来配置凭证。这个命令会交互式地让你输入信息，你按提示填就好。**不用把任何密钥发给我。**

```bash
aliyun configure --mode RamRoleArn --profile opc-deploy
```

**交互式输入提示：**
```
Access Key Id []:          ← 输入之前保存的 AccessKey ID
Access Key Secret []:      ← 输入之前保存的 AccessKey Secret
Sts Region []:             ← 直接回车（留空）
Ram Role Arn []:           ← 输入角色 ARN，格式 acs:ram::你的账号ID:role/ecs-deploy-role
Role Session Name []:      ← 输入 opc-web-card-session
External ID []:            ← 直接回车（留空，同账号不需要）
Expired Seconds [900]:     ← 直接回车（默认900秒即可）
Default Region Id []:      ← 输入 cn-shanghai
Default Output Format [json]: ← 直接回车
Default Language [zh|en] zh:  ← 直接回车
```

---

### 3.3 验证凭证配置

**执行验证命令：**

```bash
aliyun sts GetCallerIdentity --profile opc-deploy
```

**期望输出：**
```json
{
  "AccountId": "1234567890",
  "Arn": "acs:ram::1234567890:assumed-role/ecs-deploy-role/opc-web-card-session",
  "IdentityType": "AssumedRoleUser"
}
```

**成功判断：** `IdentityType` 为 `AssumedRoleUser` 表示配置正确。

**失败处理：**
- `InvalidAccessKeyId` → AccessKey ID 输入错误，重新 configure
- `EntityNotExist.Role` → Role ARN 格式错误，检查角色名和账号 ID
- `NotAuthorized` → 信任策略未正确配置，回到 2.4 检查

**成功后的话术：**
> ✅ 凭证配置成功！「跑腿员穿好制服了」。接下来我帮你创建云服务器！
