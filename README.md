# clay-blog

基于 Astro 的个人博客模板，内容使用 Markdown 管理，包含文章、随笔、归档、分类、标签、搜索、RSS、站点地图和 Twikoo 后端评论。仓库自带示例文章，克隆后可直接构建预览。

## 项目结构

```
blog/
├── public/                # 静态资源
│   ├── avatars/           # 头像
│   ├── covers/            # 封面图加载失败时的兜底图
│   └── sw.js              # 旧版 service worker 的清理开关
├── src/
│   ├── components/        # 组件（导航、音乐播放器、评论等）
│   ├── layouts/           # 页面布局
│   ├── pages/             # 路由页面（首页、归档、搜索、RSS 等）
│   ├── content/
│   │   ├── posts/         # 文章（Markdown）
│   │   └── notes/         # 随笔（Markdown）
│   ├── data/
│   │   ├── site.config.json      # 关于页配置（见下文）
│   │   └── github-projects.json  # GitHub 项目缓存
│   ├── lib/               # 工具函数与 Markdown 插件
│   └── styles/            # 全局样式
├── astro.config.mjs       # Astro 配置
└── package.json
```

## 快速开始

**环境要求：只需要 Node.js（>= 18.20 或 20.3+ 或 22+）。不需要单独安装 Astro**——它是项目依赖，`npm install` 时会自动装好（npm 随 Node.js 一起安装）。

第一次使用，按顺序执行：

```bash
# 1. 安装 Node.js（未安装时）：到 https://nodejs.org/ 下载 LTS 版本安装

# 2. 下载项目
git clone https://github.com/laogou717/clay-blog.git
cd clay-blog

# 3. 安装依赖（自动安装 Astro）
npm install

# 4. 启动本地预览
npm run dev
```

浏览器打开 `http://localhost:4321` 即可看到博客。修改 `src/content/` 下的 Markdown 后保存，页面会热更新。

## 构建

```bash
npm run build
npm run preview
```

构建产物输出到 `dist/`，可部署到任意静态托管平台。

## 部署

1. 连接仓库（Vercel / Netlify / Cloudflare Pages / GitHub Pages 均可），构建命令 `npm run build`，输出目录 `dist`
2. 在平台环境变量中设置 `SITE_URL` 为正式域名
3. 自定义域名按平台指引添加 DNS 解析（CNAME 记录）
4. 若旧域名需要迁移，在平台侧配置 301 重定向

更详细的说明见示例文章《静态站点的部署实践》。

## 环境变量

复制 `.env.example` 为 `.env`，按需填写：

```bash
SITE_URL=https://example.com
PUBLIC_TWIKOO_ENV_ID=https://your-twikoo.example.com
PUBLIC_NETEASE_PLAYLIST_ID=8792942606
PUBLIC_MUSIC_API=https://meting.mikus.ink/api
```

- `SITE_URL`：站点正式域名，用于 RSS、sitemap、canonical URL 和结构化数据。
- `PUBLIC_TWIKOO_ENV_ID`：Twikoo 后端地址。未配置时，评论区和选中文字引用评论功能会自动隐藏。
- `PUBLIC_NETEASE_PLAYLIST_ID`：网易云音乐歌单 ID。未配置时使用博主歌单 `8792942606`。
- `PUBLIC_MUSIC_API`：Meting 兼容的音乐 API 地址，默认使用 `https://meting.mikus.ink/api`，也可换成自建服务。

## 音乐播放器

全站右下角的悬浮播放器会在浏览器中按需读取网易云歌单，不会自动播放。替换歌单时，从网易云歌单链接中复制 `id` 数字，写入 `.env` 的 `PUBLIC_NETEASE_PLAYLIST_ID` 后重新启动开发服务。

Meting 是非官方接入方式，受网易云版权、VIP 与地区限制影响，个别歌曲可能无法播放。生产环境若需要更稳定，建议将 `PUBLIC_MUSIC_API` 指向自建的 Meting 兼容服务。

## 内容

- 文章：`src/content/posts/`
- 随笔：`src/content/notes/`
- 内容字段定义：`src/content.config.ts`

文章和随笔会在构建时生成静态页面。若使用自定义 `slug`，需要保证唯一，避免内容集合覆盖。

仓库自带一组**示例文章与随笔**（`示例/`、`AI/`、`技术/` 分类），用于演示首页、分类、标签、归档、搜索与 RSS 等页面的完整效果；封面图引用图床占位图。使用前请直接删除这些示例文件，换成自己的内容。

## 更换成你的博客

克隆后需要改的地方，全部集中在这几个文件：

| 要改的内容 | 位置 |
| --- | --- |
| 站点名称 / 简介 / 作者 | [src/data/site.config.json](src/data/site.config.json) 的 `siteName` / `siteDescription` / `siteAuthor` |
| 头像 | 替换 `public/avatars/avatar.png` 与 `avatar-bw.png` |
| GitHub 用户名与仓库链接 | [src/data/site.config.json](src/data/site.config.json) 的 `githubUser` / `githubRepo`（不想要导航栏的 GitHub 按钮，把 `githubRepo` 留空即可） |
| 正式域名 | 复制 `.env.example` 为 `.env`，设置 `SITE_URL` |
| 评论后端 | `.env` 的 `PUBLIC_TWIKOO_ENV_ID`（不配则评论自动隐藏） |
| 音乐播放器歌单 | `.env` 的 `PUBLIC_NETEASE_PLAYLIST_ID` |
| 文章与随笔 | 删除 `src/content/posts/` 下的 `示例/`、`AI/`、`技术/` 目录和 `src/content/notes/` 下的示例随笔，换成自己的 Markdown |

改完 `site.config.json` 后重新 `npm run dev` 即可生效。

## 关于页自定义配置

关于页的开源项目与资源下载均通过 [src/data/site.config.json](src/data/site.config.json) 配置：

```jsonc
{
  // GitHub 用户名：构建时自动拉取该账号的公开仓库作为「开源项目」
  "githubUser": "laogou717",
  // 按仓库名覆盖卡片细节：icon 为卡片图标（见 src/components/Icon.astro 的图标名），
  // article 为指向博客内相关文章的「笔记」链接（不填则不显示该按钮）
  "projectOverrides": {
    "md-wechat": { "icon": "wechat" },
    "Steam-game-cover-gets": { "icon": "download", "article": "/posts/ai-era/github/steamcovr/" }
  },
  // 静态项目组（如「资源下载」），可直接增删条目或整组
  "projectGroups": [
    {
      "title": "资源下载",
      "description": "文章里提到的网盘、脚本和可复用资料。",
      "items": [
        {
          "title": "Bookmarklet 小书签",
          "owner": "Baidu Pan",
          "description": "浏览器小书签源码与使用说明，适合做轻量自动化。",
          "icon": "bookmark",
          "href": "https://pan.baidu.com/s/1olHsMYzcOtGCYiY6nUs6eQ?pwd=6666",
          "article": "/posts/book/",
          "tags": ["书签", "脚本", "code:6666"]
        }
      ]
    }
  ]
}
```

「开源项目」组在构建时从 GitHub API 拉取（自动排除 fork，按 Star 数排序），新增仓库重新构建即可自动出现；API 不可达时回退到 [src/data/github-projects.json](src/data/github-projects.json) 缓存，可用以下命令手动刷新（将 `laogou717` 替换为你的 GitHub 用户名）：

```bash
curl -s "https://api.github.com/users/laogou717/repos?per_page=100" -o src/data/github-projects.json
```

## 评论

评论前端没有使用 Twikoo 的默认 UI，而是通过 `src/lib/comments.js` 调用 Twikoo 后端接口并渲染自定义样式。

文章正文支持选中文字后引用到底部评论区：

1. 确认 `.env` 已配置 `PUBLIC_TWIKOO_ENV_ID`，否则评论区和引用按钮都会隐藏。
2. 在文章正文或随笔正文中拖选至少 2 个字符。
3. 选区上方会出现“引用评论”按钮。
4. 点击按钮后页面会滚动到底部评论区，并把选中的文字以 Markdown blockquote 写入评论框。
5. 发送成功后页面会回到刚才的阅读位置。

## 常见问题

**没装过 Node.js 怎么办？**
到 https://nodejs.org/ 下载 LTS 版本安装，npm 会一并装好，然后回到「快速开始」从第 3 步继续。

**Astro 需要单独安装吗？**
不需要。Astro 是 `package.json` 里的依赖，`npm install` 自动安装，无需全局安装任何东西。

**`npm install` 很慢 / 失败？**
国内网络可切换到镜像源：

```bash
npm config set registry https://registry.npmmirror.com
```

**提示 Node 版本太低？**
Astro 5 要求 Node 18.20.8+ / 20.3.0+ / 22.0.0+，升级 Node.js 后重试。

**端口被占用？**
`npm run dev -- --port 4322` 指定其他端口。

## 许可证

本项目基于 [MIT License](LICENSE) 开源。仓库自带的示例文章与随笔仅用于演示，可直接删除替换。
