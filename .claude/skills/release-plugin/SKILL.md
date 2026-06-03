---
name: release-plugin
description: 发布 DooTask 审批（Approval）插件新版本：更新中英双语 CHANGELOG、确定版本号、打 tag 触发 CI 构建并发布到 Docker Hub 和 DooTask AppStore，监控 Action 直至成功。用户要发版/打 tag/出新版本时使用。
---

# 发布 DooTask 审批插件

发布靠**推送 tag** 触发 `.github/workflows/release.yml`：构建多架构 Docker 镜像（`kuaifan/dooapprove`）推到 Docker Hub，再打包 `dootask-plugin/` 以系统应用方式发布到 DooTask AppStore（appid `approve`）。常规 push / PR 不会触发，光 `git tag` 不 push 也不会。

发布公开且不可逆，**每一步操作前先与用户确认**（尤其版本号）；常规 git 操作自行判断。

## 三个非踩不可的坑

**1. Tag 不带 `v` 前缀。** 版本号就是 `0.1.9` 这种纯数字。workflow 监听 `tags: '*'`，带 v 也能发，但 Docker tag 会变成 `v0.1.9`，破坏排序和 AppStore 显示。镜像 tag 与插件版本必须一致，所以 tag 名 = 插件版本号。

**2. 源目录永远叫 `dootask-plugin/0.1.0/`，是固定占位名，不随版本变。** 打包步骤在 runner 里 `mv 0.1.0 <tag>` 重命名成 tag 名再打包——所以 repo 里它必须一直叫 `0.1.0`，哪怕已发到 `0.1.9` 也别改名、别按版本号新建目录。repo 里一旦出现 `dootask-plugin/0.2.0/` 之类，`mv 0.1.0` 会因目标已存在而失败。改插件运行配置（`docker-compose.yml`、`nginx.conf`、`CHANGELOG`）就改 `dootask-plugin/0.1.0/` 下的文件；改插件元信息（名称、描述、tags、fields）改 `dootask-plugin/config.yml`。

**3. 版本号只增不重复。** AppStore 拒绝重复版本号，重发只能递增 patch（`0.1.9` → `0.1.10`）。误推的 tag 删了也撤不回已发布的 AppStore 版本。

## CHANGELOG（发版前必做，中英双语）

发版前同步更新这两个文件（位于固定占位目录，会随打包重命名进版本包，AppStore 用作更新说明）：

- `dootask-plugin/0.1.0/CHANGELOG.md`（英文，默认）
- `dootask-plugin/0.1.0/CHANGELOG_zh.md`（中文）

**覆盖式，不是追加式** —— 每次发版直接把整个文件替换为本次更新内容，**不保留上一版条目**（AppStore 自己维护历史）。文件里不写版本号和日期，版本号由 tag 隐式承载。

按分类列点，**只用本次涉及的分类**。常用分类（中英必须严格对应）：

| 英文 | 中文 |
|------|------|
| Added | 新增 |
| Fixed | 修复 |
| Updated | 更新 |
| Changed | 变更 |
| Improved | 优化 |
| Removed | 移除 |

写法要求：
- 一句话一条，**写给最终用户看**，不是开发者——说"支持删除已结束的审批"，不要说"新增 process/delById 接口"。
- 涉及具体功能、配置项时写出名字方便用户判断。
- 中英两个文件**分类、条数、含义一一对应**（第 N 条说的是同一件事）。
- 每条一行，保持简洁。

格式示例（按本次实际改动写）：

```markdown
### 新增
- 支持删除已结束的审批（仅发起人或管理员）。

### 修复
- 修复 <问题>。
```

## 发布流程

1. 在 `main`、工作区干净、与 `origin/main` 同步。
2. 看最新 tag（`git tag --sort=-creatordate | head`），按 SemVer 与用户确认下一个号（patch=bugfix，minor=新功能）。
3. **更新中英双语 CHANGELOG**（见上节），连同待发布的代码改动一起 commit 并 `git push origin main`。
4. `git tag <版本号> && git push origin <版本号>`——推上去立即触发，没回头路。
5. 盯 Action：`gh run watch` 或 https://github.com/dootask/approval/actions 。
6. 验证：Docker Hub `kuaifan/dooapprove` 出现新 tag；AppStore 里 `approve` 插件版本已更新，且更新说明显示本次 CHANGELOG 内容。

> 误推 tag 可 `git push --delete origin <tag> && git tag -d <tag>` 删除重打；但若 Action 已发布到 AppStore，删 tag 不会撤回已发布版本。

## 发布失败时

- **Docker 登录失败**：secrets 名是 `DOCKER_USERNAME` / `DOCKER_PASSWORD`（不是 `DOCKERHUB_*`），多半过期或缺失；账号需有推送到 `kuaifan/dooapprove` 的权限。
- **镜像构建失败**：构建走根目录多阶段 `Dockerfile`（前端 vite + 后端 Go + nginx），本地可先用 `DOCKER_BUILDKIT=1 docker build -t kuaifan/dooapprove:test .` 复现。
- **AppStore 发布失败**：查 `dootask-plugin/config.yml`、`dootask-plugin/0.1.0/docker-compose.yml` 和 `DOOTASK_USERNAME` / `DOOTASK_PASSWORD`。
- **版本号已存在**：AppStore 拒绝重复版本号，递增 patch 重发。

发布依赖上面这 4 个 Repository Secrets（仓库 Settings → Secrets and variables → Actions），由管理员一次性配置，本技能不负责设置；登录类报错时提醒用户检查。
