# dsh-bitwarden

把 **Bitwarden / Vaultwarden** 接入 [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness)（`dsh`）：
密码库可以在「设置 → 密码库面板」里浏览和编辑，模型可以按需检索、读取和写入；密码、TOTP、
卡号等默认以占位符脱敏，会话日志里的明文也能擦除。

支持 **官方 Bitwarden 云（US / EU）** 与 **自建 Vaultwarden**。

> 本仓库是 [`dsh-vaultwarden` 0.17.6](https://github.com/BuvkB/dsh-vaultwarden)（MIT）的本地维护分支。
> 上游 0.17.6 在**官方 Bitwarden 云 + Argon2id 账号**上会把登录误报成「邮箱或主密码不正确」，
> 这里修掉了它。其余功能、面板与工具与上游一致。

## 修了什么（Why this fork）

官方云会在 `POST /identity/accounts/prelogin` 上按客户端标识头区分响应：
不带 `bitwarden-client-name` + `bitwarden-client-version` 的请求，拿到的是**遗留的通用 KDF**
（`PBKDF2, kdfIterations=600000`），而不是账号真实的 KDF（例如 `Argon2id, iterations=3, memory=64, parallelism=4`）。
用错误的 KDF 派生出的 `masterPasswordHash` 服务端永远不认，于是返回
`400 invalid_username_or_password`——看起来是密码错，其实密码是对的。

上游 0.17.6 的 `prelogin()` 恰好没带这两个头，所以凡是**改过 KDF 的官方云账号**（Argon2id）
100% 登录失败；默认 PBKDF2-600k 的账号因数值碰巧相同而不受影响。

修复只有一处：`prelogin()` 复用 token 请求已有的客户端标识头（`lib/vault.js`，
搜索 `Client identification is REQUIRED on prelogin`）。对自建 Vaultwarden 无副作用。

## 功能

- **实时同步与缓存**：SignalR WebSocket 推送变更，不可用时降级轮询；本地加密缓存 + revision 探测。
- **密码库面板**：分区导航、搜索、分页、条目详情、TOTP 倒计时、复制、新建/编辑、归档/回收站/恢复、文件夹移动。
- **脱敏与日志擦除**：工具读到密码/TOTP/卡号返回稳定占位符，按条目+字段单次取回；定期把会话日志中的明文替换为 `[REDACTED:BW]`。
- **通行密钥（passkey）**：读取 passkey 元数据，按插件权限导出为 0600 的 WebAuthn 凭据文件（含 `cdp` 与 `playwright` 两种块），支持导入/移除/清理；用法见 [docs/passkey-login.md](docs/passkey-login.md)。
- **权限档位**：只读 / 每次确认 / 完全权限；DSH 会话处于 `workspace-write` / `read-only` 时，写操作与通行密钥导出仍走宿主审批。
- **两步验证**：官方云账号启用 2FA（TOTP / 邮箱 / 恢复码）时，登录页会要求输入验证码，可勾「记住此设备」。

## 安装

```sh
# 从本仓库的 Release 安装（推荐）
dsh plugin --profile desktop add https://github.com/netxeye/dsh-bitwarden/releases/latest/download/dsh-bitwarden.tgz
```

或者在插件市场（dsh-market）里搜索 **dsh-bitwarden** 一键安装。

要求 Node.js >= 22。

## 配置

在 **设置 → 插件 → dsh-bitwarden** 填写：

| 字段 | 说明 |
| --- | --- |
| `serverUrl` | `https://vault.bitwarden.com/`（US）、`https://vault.bitwarden.eu/`（EU），或自建 Vaultwarden 地址（如 `http://127.0.0.1:8080`） |
| `email` | 登录邮箱 |
| `masterPassword` | 主密码；只写字段，不回显 |
| `apiKeyClientId` / `apiKeyClientSecret` | 可选：API 密钥登录（绕过两步验证；解密库存仍需主密码） |
| `accessMode` | `readonly` / `ask` / `auto`，控制模型写回权限 |
| `maskEnabled` | 插件脱敏总开关（默认开） |
| `sweepMode` | 会话日志明文擦除开关 |

也支持 `DSH_BITWARDEN_*` 环境变量。

## 工具

`bitwarden_find` · `bitwarden_get` · `bitwarden_reveal` · `bitwarden_status` · `bitwarden_sync` ·
`bitwarden_folders` · `bitwarden_create` · `bitwarden_update` · `bitwarden_delete` · `bitwarden_restore`

通行密钥：`bitwarden_passkey_list` · `bitwarden_passkey_export` · `bitwarden_passkey_cleanup` ·
`bitwarden_passkey_import` · `bitwarden_passkey_remove`；另有 `bitwarden_sweep` 立即擦除日志明文。

## 安全说明

- 面板只走宿主已认证的 `/api` connection RPC，插件不注册自己的 HTTP 路由。
- 主密码与 API 密钥是只写字段，不落浏览器；存储在本机 profile 配置中。
- 脱敏开启时，凭据以 `{{BW_*}}` 占位符进入模型上下文与日志；确实需要明文时按条目 + 字段单次取用。
- 插件代码会在你的权限下运行：请只安装你信任的来源，并先检查源码。

## 维护与上游同步

本仓库是本地维护的分支，见 [FORK.md](FORK.md)：

- 上游修好该问题后，可以直接切回 registry 版本；
- 需要同步上游新版本时，把新包内容覆盖过来、保留 `prelogin()` 的修复、升版本后重新打包发布。

## 许可与致谢

MIT，见 [LICENSE](LICENSE)、[THIRD-PARTY-NOTICES.md](THIRD-PARTY-NOTICES.md)。
原始实现：[BuvkB/dsh-vaultwarden](https://github.com/BuvkB/dsh-vaultwarden)（MIT）。
本分支的修复与维护：[netxeye](https://github.com/netxeye)。

---

### English

A DeepSeek Harness plugin that connects **Bitwarden / Vaultwarden** to `dsh`: an entry browser
panel, masked credential reads for models, TOTP, passkey export, read/write tools, live sync and
session-log redaction. Works with Bitwarden cloud (US/EU) and self-hosted Vaultwarden.

This repository is a maintained fork of `dsh-vaultwarden` 0.17.6 (MIT) that fixes the official-cloud
password login for **Argon2id** accounts: the `prelogin` request now carries the client identification
headers, so the server returns the account's real KDF instead of the legacy generic PBKDF2-600k
parameters, which used to derive an authentication hash the server always rejected
(`invalid_username_or_password`). See [FORK.md](FORK.md). Licensed MIT; original implementation by
[BuvkB](https://github.com/BuvkB/dsh-vaultwarden).
