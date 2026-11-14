# 本地源码开发与问题排查

本文记录 RSSHub 在本地从源码构建、启动，以及调试 YouTube 和 GitHub Trending 路由时遇到的问题。

## 环境要求

- Node.js：项目要求 `^24.15.0 || ^26.13.0`。本文验证时使用 Node.js `24.22.3`。
- 包管理器：仓库指定 `pnpm@12.18.0`。
- 当前机器的 Corepack 缓存缺少 pnpm 启动文件，因此使用 `npm exec` 临时运行锁定版本的 pnpm：

```bash
export PATH="/opt/homebrew/opt/node@24/bin:$PATH"
npm exec --yes --package=pnpm@12.18.0 -- pnpm install --frozen-lockfile
npm exec --yes --package=pnpm@12.18.0 -- pnpm build
```

如果 Node.js 已满足版本要求，可按环境调整 `PATH`；也可修复 Corepack 后直接使用 `pnpm`。

## 启动开发服务器

此环境通过本地 HTTP 代理访问外网。RSSHub 的请求封装读取 `PROXY_URI`，不会自动沿用 shell 的 `HTTPS_PROXY`。下面的地址 `127.0.0.1:7890` 是本机代理示例，请按自己的代理地址调整。

GitHub Trending 路由要求 `GITHUB_ACCESS_TOKEN`。若已使用 GitHub CLI 登录，可从 `gh auth token` 读取凭据，不要把 token 提交到仓库或写进文档：

```bash
export PATH="/opt/homebrew/opt/node@24/bin:$PATH"
PROXY_URI="http://127.0.0.1:7890" \
GITHUB_ACCESS_TOKEN="$(gh auth token)" \
npm exec --yes --package=pnpm@12.18.0 -- pnpm dev
```

本地服务默认监听 `http://localhost:1200`。只调试不需要 GitHub 配置的路由时，可省略 `GITHUB_ACCESS_TOKEN`。

## 已处理的问题

### YouTube 频道返回 `TypeError: fetch failed`

**原因：** `curl` 能通过 shell 中配置的代理访问 YouTube，但 RSSHub 的请求封装使用项目自己的代理配置。未设置 `PROXY_URI` 时，Node 请求尝试直连 YouTube 并失败。

**解决：** 启动开发服务器时设置 `PROXY_URI`，例如：

```bash
PROXY_URI="http://127.0.0.1:7890" npm exec --yes --package=pnpm@12.18.0 -- pnpm dev
```

设置后，`/youtube/channel/UC08YjBHjjPFu8T1KbHGomnQ` 返回 HTTP 200。

### GitHub Trending 返回 `ConfigNotFoundError`

**原因：** 此路由在 [lib/routes/github/trending.tsx](lib/routes/github/trending.tsx) 中要求 `GITHUB_ACCESS_TOKEN`，缺少 token 时会主动禁用路由。

**解决：** 启动服务器时提供有效的 GitHub access token。若本机已用 GitHub CLI 登录，可使用 `GITHUB_ACCESS_TOKEN="$(gh auth token)"`；不要将 token 提交到 Git 或写入公开文档。

设置后，`/github/trending/daily/javascript/en` 返回 HTTP 200。

### YouTube 频道条目输出详细描述

官方 YouTube 频道 feed 使用 `<media:description>` 提供每个视频的描述。本地 RSSHub feed 原先只把视频播放器写入 RSS `<description>`，没有单独输出视频的详细描述。

RSSHub 的 YouTube 路由优先使用 YouTube Data API（配置 `YOUTUBE_KEY` 时），否则使用 `youtubei.js` 的 Innertube。Innertube 频道视频列表中的 `LockupView` 不包含完整描述，因此代码复用带缓存的 `getVideoInfo(videoId)`，通过同一个 Innertube 客户端的 `getBasicInfo(videoId)` 逐条取描述。首次请求会为每条视频额外请求一次详情，后续命中缓存；描述请求失败时会省略该条的媒体描述，不影响频道 feed 返回。Google API 路径直接使用 `snippet.description`。

内部字段名和本地 RSS XML 节点名均为 `description_detail`。它将展示用的 `<description>` 与视频原始描述分开。内容取自与官方 feed 相同的视频描述；节点名是 RSSHub 本地扩展。

验证结果：本地频道 feed 的 30 条视频都包含 `<description_detail>`；与官方 feed 重叠的 15 条视频，描述均逐字一致。`pnpm typecheck` 通过。

## 验证

```bash
curl -i http://localhost:1200/healthz
curl -i http://localhost:1200/youtube/channel/UC08YjBHjjPFu8T1KbHGomnQ
curl -i http://localhost:1200/github/trending/daily/javascript/en
```

本次验证中，健康检查和两个路由均成功；路由返回 RSS XML。构建期间曾出现缓存不可用提示，但构建完成且不影响上述路由运行。
