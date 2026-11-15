# 本仓库与上游的关系（FORK）

本仓库是 [`BuvkB/dsh-vaultwarden`](https://github.com/BuvkB/dsh-vaultwarden) 0.17.6（MIT）的
**本地维护分支**，包名改成 `dsh-bitwarden`，保留上游全部功能，只做必要修复。

## 改动清单

1. **客户端标识头（唯一的逻辑改动）**

`lib/vault.js` 的 `prelogin()` 原本只发 `content-type` / `accept`。官方 Bitwarden 云对没有
`bitwarden-client-name` + `bitwarden-client-version` 的 prelogin 请求返回**遗留通用 KDF**
（`PBKDF2, kdfIterations=600000`），而不是账号真实 KDF（如 `Argon2id, 3, 64, 4`）。
错误的 KDF 派生出服务端不认的 `masterPasswordHash`，登录被拒并报
`invalid_username_or_password`（密码其实是正确的）。

修复：`prelogin()` 复用 token 请求的 `tokenHeaders()`：

```js
headers: { ...tokenHeaders(), 'content-type': 'application/json' },
```

代码里带注释标记 `Client identification is REQUIRED on prelogin`，可用它定位。
对自建 Vaultwarden 无副作用（它不按客户端头区分）。

2. **改名**：`dsh-vaultwarden` → `dsh-bitwarden`（包名、bundle 行 id、设置命名空间、
客户端模块 id、本地数据目录 `~/.dsh/data/dsh-bitwarden`）。工具名仍是 `bitwarden_*`。

3. **仓库信息**：`repository` / `homepage` / `bugs` 指向本仓库；删掉上游的徽章图、
release 流程文档与英文 README（内容合并进 README.md）。

4. 上游的 `LICENSE`、`THIRD-PARTY-NOTICES.md`、`CHANGELOG.md` 原样保留（MIT 署名）。

## 本地构建 / 发布

```bash
cd dsh-bitwarden
node --version            # 需要 >= 22
pnpm pack --pack-destination ..     # 产出 dsh-bitwarden-<version>.tgz
```

把 tgz 挂到 GitHub Release（文件名保持 `dsh-bitwarden.tgz` 也可以，README 用的是
`releases/latest/download/dsh-bitwarden.tgz`）。

## 同步上游新版本

1. 下载上游新版包内容，覆盖到本仓库目录（保留本文件与 README 的 fork 说明）。
2. 重新打上 `prelogin()` 的修复（若上游已修复则删除该改动并在 README 里注明）。
3. 改 `package.json` 的 `version`，提交并发布新 Release。
4. 如果上游已修复该 bug，推荐直接切回 registry 版本：`dsh plugin --profile desktop add dsh-vaultwarden`。

## 许可

上游 MIT（原文保留在 `LICENSE`）；本分支的修改同样以 MIT 发布。
