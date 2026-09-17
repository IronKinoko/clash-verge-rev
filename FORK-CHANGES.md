# Fork 改动说明

本仓库是 [clash-verge-rev/clash-verge-rev](https://github.com/clash-verge-rev/clash-verge-rev) 的 fork。
本文档列出相对上游 `dev` 分支的全部改动，便于同步时核对和冲突处理。

对比基准：`upstream/dev`（clash-verge-rev/clash-verge-rev:dev）

注意：自动同步工作流使用 rebase 对齐上游，会改写 fork commit 的 SHA，
因此本文档只列 commit 主题，不列 SHA。

## 改动总览

| 文件 | 类型 | 说明 |
| --- | --- | --- |
| `.github/workflows/build-macos-only.yml` | 新增 | macOS 专用构建 + 自动更新发布 |
| `.github/release-notes/macos-only.md` | 新增 | macOS 构建的 release 固定说明（安装提示） |
| `src-tauri/tauri.macos.ci.conf.json` | 新增 | CI 专用 Tauri 配置（含更新器公钥/端点） |
| `scripts/set_dns.sh` | 修改 | DNS 追加而非替换 |
| `.github/workflows/sync-upstream.yml` | 新增 | 每 5 分钟自动 rebase 上游 |
| `FORK-CHANGES.md` | 新增 | 本文档 |

## 1. macOS 专用构建工作流

文件：`.github/workflows/build-macos-only.yml`

为 fork 提供独立的 macOS 构建流水线，与上游的 autobuild 完全隔离：

- **触发方式**：仅 `workflow_dispatch`（手动），或由同步工作流自动触发
- **构建目标**：`macos-latest`，仅 `aarch64-apple-darwin`（Apple Silicon）
- **工具链**：Rust 1.98.1、Node 24.20.0、pnpm
- **缓存**（与上游 `autobuild.yml` 相同的 key，可直接命中上游已生成的缓存）：
  - `Swatinem/rust-cache@v2`：缓存全部 crate 下载 + 工作区 `target/` 编译产物
  - pnpm store：`setup-node` 内置缓存 + 显式 `actions/cache` 缓存 `~/.pnpm-store`
- **Release 策略**：release tag 为版本号（如 `v2.5.4`，取自 `package.json`）；
  tag 已存在时复用其 release（"Prepare release" 步骤），同版本重复构建不报错
- **签名**：使用 fork 自己的 Tauri updater 签名密钥
  （secret `TAURI_PRIVATE_KEY`，无密码）
- **更新清单**：构建成功后运行 `pnpm updater`（上游的
  `scripts/updater.mjs`），扫描本 fork 的 release，把各平台的下载 URL 和
  签名写入 `update.json` / `update-proxy.json`，发布到 fork 的 `updater`
  release 上
- **Release 正文**：固定说明来自 `.github/release-notes/macos-only.md`；
  构建结束后 `Update release notes` 步骤重新生成本次构建信息（排除 fork
  自己的最新一条 commit 后的上游提交列表、构建时间、构建 hash），
  `<!-- upstream-build-info -->` 标记以上的既有内容原样保留，标记以下的部分
  每次构建整体替换，正文不会随构建次数增长
- **Release 时间**：GitHub API 不允许写 `published_at`，因此用「先转 draft
  再发布」刷新 release 时间；该操作不影响 assets 和 prerelease 标记

## 1.1 release 正文结构

```
<固定说明，来自 .github/release-notes/macos-only.md>

<!-- upstream-build-info -->
## 本次构建

上游提交（已排除本仓库最新一条）：
- `<hash>` <subject>        # 最近 20 条上游提交

构建时间：<Asia/Shanghai 时间>
构建 Hash：`<fork HEAD short hash>`
```

`git log` 的范围是 `HEAD~1`：fork 的最新一条 commit 是保留给 fork 自己的
（如 CI 改动），不属于上游信息，因此从上游信息里排除。

## 2. CI 专用 Tauri 配置

文件：`src-tauri/tauri.macos.ci.conf.json`（新增）

fork 没有 Apple 签名身份和上游的更新密钥，因此 CI 构建单独使用此配置，
避免上游 `tauri.conf.json` 中的相关字段导致构建或更新失败：

- `createUpdaterArtifacts: true`：产出带 minisign 签名的 `aarch64.app.tar.gz`
  和 `.sig`（Tauri 更新器用）
- `signingIdentity: null`：跳过 Apple 签名
- `minimumSystemVersion: "11.0"`
- `plugins.updater`：
  - `pubkey`：fork 自生成的更新器公钥（与 `TAURI_PRIVATE_KEY` 配对）
  - `endpoints`：指向 fork 的 `releases/download/updater/update.json`
    （含 hwdns / gh-proxy 两个代理前缀，与上游一致）
- 放在独立配置文件而非主 `tauri.conf.json`，保持与上游的最小 diff，
  rebase 同步不冲突

## 3. 系统 DNS 脚本：追加而非替换

文件：`scripts/set_dns.sh`

上游版本执行 `networksetup -setdnsservers <port> <IP>`，会**替换**掉网卡上
已有的全部 DNS 服务器。fork 版本改为**追加**：

1. 读取网卡现有 DNS 列表，过滤出合法的 IPv4/IPv6 地址
2. 将过滤后的原始列表逐行写入 `.original_dns.txt`（供 `unset_dns.sh` 恢复）
3. 目标 IP 不在列表中才追加
4. 用「原有列表 + 目标 IP」整体写回

效果：设置 114.114.114.114 时不破坏用户已有的 DNS 配置，恢复时也能还原完整列表。

## 4. 上游自动同步工作流

文件：`.github/workflows/sync-upstream.yml`（新增）

- **调度**：每 5 分钟轮询一次（`*/5 * * * *`，cron 支持的最短间隔），
  也可手动触发；GitHub 在负载高时可能延迟调度
- **同步方式**：`git rebase upstream/dev`，不产生 merge commit；
  推送使用 `git push --force-with-lease`（rebase 会改写 fork commit SHA）
- **构建联动**：仅当 rebase 后 HEAD 有变化（上游有新提交）才 push 并自动
  触发 `build-macos-only.yml`
- **冲突处理**：rebase 发生冲突时 job 失败且不推送，需本地解决后提交并推送
- **上游 workflow**：保留上游原版触发配置，在 GitHub Actions 页面禁用不需要的
  workflow，避免持续修改上游高频文件导致冲突

## Commit 列表（相对 upstream/dev，主题）

```
feat(ci): record upstream build info in macOS release notes
chore(fork): consolidate fork build and sync automation
```

fork 只有以上两条 commit（rebase 会改写 SHA，因此按主题列出）：

- `chore(fork): consolidate fork build and sync automation`：macOS 构建工作流、
  CI Tauri 配置、上游同步工作流、DNS 追加脚本、本文档
- `feat(ci): record upstream build info in macOS release notes`：构建结束后向
  release 正文写入本次构建信息（上游提交列表、时间、hash）

改动文件清单见 `git diff --name-status upstream/dev dev`。

## 应用自动更新链路

App 内的更新检查（`@tauri-apps/plugin-updater`）读取编译进应用的
`plugins.updater` 配置：

1. 请求 fork 的 `update.json`（含各平台 URL + minisign 签名）
2. 下载 `aarch64.app.tar.gz`，用内嵌公钥验证 `.sig`
3. 原地替换 `.app` bundle 完成更新

该验证走 Tauri 自己的 minisign 体系，与 Apple 签名/公证无关。注意 Tauri
更新器只在清单版本**高于**当前运行版本时才提示更新：上游发新版本 →
fork 同步 + 构建 → App 自动更新到 fork 构建；版本不变的重建需手动下载。
