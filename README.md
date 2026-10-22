# GitHub 2FA 配置说明（面向中国大陆用户）

> 记录一次 GitHub 双因素认证的配置过程，以及一个较少被提及的限制：
> **Bitwarden 免费版可以保存 TOTP 密钥，但不生成验证码。**

`$0` · 无需 VPN · Windows / macOS / Linux 通用

---

## 摘要

- GitHub 强制要求开启 2FA。短信通道不覆盖中国大陆号段，实际可用的方式是 TOTP 验证器。
- Bitwarden 免费版的「验证器密钥」字段仅支持保存密钥，生成动态码属于 Premium 功能。界面表现为：字段保存成功、复制按钮无效、旁边显示「升级」标签。
- 可用的免费方案：手机应用商店提供的 **Microsoft Authenticator**，或 **2FAS**。
- 配置完成后需保存恢复码，并将 Git 推送切换为 SSH，否则 `git push` 会因密码认证已废弃而失败。

---

## 短信通道不可用

GitHub 支持的第二因素包括短信验证，但该通道不覆盖中国大陆号段，与运营商无关。GitHub 社区讨论 [community#185029](https://github.com/orgs/community/discussions/185029) 记录了该问题，官方未提供替代通道。

安全密钥（YubiKey 等）可用，但涉及购买与物流，难以在 45 天期限内完成。GitHub Mobile 只能作为补充，前提是该应用内已登录同一账号。

因此实际方案是 TOTP 验证器。其机制为：GitHub 生成一串 Base32 密钥，验证器用该密钥结合当前时间计算一个 6 位数字，服务器以相同方式计算并比对，每 30 秒更新一次。整个过程在本机完成，不依赖手机号和外网。

与之相关的一条排查规则：**设备时间不准确时验证码必然校验失败**。误差超过 30 秒即报 invalid，提示中不包含原因。配置前应确认手机与电脑均开启自动时间同步。

---

## Bitwarden 免费版的限制

### 复现步骤

1. 在 Bitwarden 新建一条 GitHub 登录条目；
2. 将 GitHub 2FA 页面的 setup key 粘贴至 **验证器密钥 / Authenticator Key (TOTP)** 字段；
3. 保存。字段中存在该字符串，无报错；
4. 打开条目详情：**「验证码（TOTP）」右侧显示「升级」标签，复制按钮无效，无倒计时标识。**

### 限制的表现特征

密钥可以保存，条目正常显示，界面无异常提示。因此无法通过报错信息判断操作是否正确，容易判断为按钮失效。

问题暴露的时机也较晚：前置步骤均正确，直到 GitHub 页面等待输入 6 位验证码时才确认无法获取，且此时该页面不能关闭。

### 原因

TOTP 生成属于 Bitwarden Premium 功能，约 $10/年。免费版开放的是密钥存储，不含验证码生成。这属于产品分层设计，不是缺陷。

1Password 存在类似情况：TOTP 功能可用，但仅限 14 天试用期，到期后账户转为只读，不再生成动态码。以其作为长期验证器时，试用结束即失去 2FA 能力。

---

## 方案对比

| 工具 | 生成 TOTP | 长期免费 | 说明 |
|---|---|---|---|
| Bitwarden Free | 否 | — | 仅存储密钥，界面显示「升级」标记 |
| Bitwarden Premium | 是 | 否 | 约 $10/年 |
| Microsoft Authenticator | 是 | 是 | 国内应用商店与 App Store 可直接获取，不依赖 GMS |
| 2FAS | 是 | 是 | 开源，备份与恢复流程完整 |
| Google Authenticator | 是 | 是 | 国内获取不便，迁移需手动导出 |
| Authy | 是 | 有限 | 存在强制付费与限制桌面端的记录 |
| 1Password | 是 | 否 | 仅 14 天试用，到期失效 |
| 微信小程序类 | 是 | 是 | 数据归属与迁移机制不透明，不建议作为主验证器 |

---

## 配置步骤

### 1. 打开 GitHub 的 2FA 设置

```
头像 → Settings → Password and authentication → Enable two-factor authentication
```

页面会显示二维码，下方提供 setup key。该页面在完成验证前不应关闭。

### 2. 在移动端绑定

Microsoft Authenticator 中点击右上角 `+`，选择其他账户，扫描屏幕上的二维码。绑定后应用内显示 GitHub 条目、6 位验证码与 30 秒倒计时。

二维码无法识别时，可点击 GitHub 页面的 `enter this text code` 展开明文密钥，在应用中选择手动输入。密钥需完整复制，不包含空格。

### 3. 回 GitHub 完成开启

填入当前验证码并点击 Continue，随后页面显示 Recovery Codes。

**需要下载并离线保存。** 手机丢失、应用卸载或换机未迁移时，恢复码是唯一的登录入口。GitHub 的账号恢复申请通常需要数周，且不保证成功。恢复码不建议仅保存在验证器应用或密码管理器中。

### 4. 清空 Bitwarden 的密钥字段

编辑条目，删除已验证器密钥后保存，避免后续在密码管理器中查找验证码。

---

## 验证器应用的真实性核对

TOTP 密钥即第二因素的完整凭据，获知密钥者可在任意时间生成有效验证码。仿冒应用获取密钥后可绕过 2FA，且使用者无法察觉。

应用商店页面的描述文字不构成保证，应核对以下信息：

- 发布者名称是否为 `Microsoft Corporation`，或页面主体是否包含微软（中国）有限公司；
- 版本号形式是否规整，例如 `6.2609.6214`；
- 安装量与更新记录，百万级安装且持续更新属正常形态；
- 申请的权限。验证器仅需相机（扫码）与通知，索取通讯录、短信、无障碍权限的应用不应安装。

国内商店将标题简写为 `Authenticator` 属常见做法，不能据此判断真伪。

---

## Git 推送会失败

开启 2FA 后 `git push` 会报认证失败。该现象与 2FA 无关，GitHub 已停止支持账号密码认证，只是时间上容易与 2FA 配置混淆。

### SSH

```bash
ssh-keygen -t ed25519 -C "your-email@example.com"
```

连续回车后获取公钥：

```bash
cat ~/.ssh/id_ed25519.pub                            # macOS / Linux / Git Bash
Get-Content $env:USERPROFILE\.ssh\id_ed25519.pub     # Windows PowerShell
```

添加至 GitHub 的 Settings → SSH and GPG keys → New SSH key，仓库地址改为 `git@github.com:<user>/<repo>.git`。

### PAT

Settings → Developer settings → Personal access tokens → Tokens (classic)，新建时勾选 `repo` 权限。生成的 token 仅显示一次，推送时作为密码填入。

---

## 常见问题

| 现象 | 原因 | 处理 |
|---|---|---|
| 验证码字段显示「升级」、复制无效 | Bitwarden 免费版不提供 TOTP 生成 | 改用 Microsoft Authenticator 或 2FAS，或升级 Premium |
| 验证码持续报 invalid | 设备时间不同步 | 手机与电脑均开启自动同步 |
| 二维码无法识别 | 屏幕反光或缩放 | 使用 `enter this text code` 手动输入密钥 |
| 短信收不到 | 大陆号段不在支持范围 | 不使用短信方式 |
| 1Password 到期后无法出码 | 试用结束账户转只读 | 到期前迁移，或使用恢复码重新绑定 |
| `git push` 认证失败 | 密码认证已废弃 | 改用 SSH 或 PAT |
| 换机后验证码丢失 | 验证器未备份 | 使用支持备份的应用；恢复码离线留存 |
| 密码管理器主密码遗忘 | 零知识架构，无法恢复 | 使用 GitHub 恢复码登录后重新绑定 |

---

## 恢复码与验证码均不可用时的处理

若手机丢失、验证器未备份且恢复码不可用，仅剩 GitHub 的账号恢复申请流程，周期以周计，成功率不保证。

可提前完成的预防措施：离线保存恢复码；在 GitHub 绑定第二个 TOTP 来源；条件允许时增加 Passkey 作为第三因素。

---

## 参考

- [GitHub Docs: Configuring two-factor authentication](https://docs.github.com/en/authentication/securing-your-account-with-two-factor-authentication-2fa/configuring-two-factor-authentication)
- [Bitwarden: Password Manager Plans](https://bitwarden.com/help/password-manager-plans/)
- [GitHub Community: 2FA SMS verification without China (#185029)](https://github.com/orgs/community/discussions/185029)
- [RFC 6238: TOTP](https://datatracker.ietf.org/doc/html/rfc6238)

---

[MIT](LICENSE)
