---
title: 隐藏的博客文章还在首页显示？排查 Astro draft 机制问题
published: 2026-07-15
description: 排查 Astro 博客中 draft: true 文章仍显示预览和分类计数错误的经历，以及 HTML 缓存策略的优化
tags: [Astro, 调试, 缓存, GitHub Pages, nginx]
category: 技术实践
draft: false
---

## 问题现象

把博客里的示例文章全部设为 `draft: true` 后，出现了两个问题：

1. **首页仍能看见隐藏文章的预览卡片**
2. **分类（"文章示例""博客指南"）后面的文章数字还是旧的**

## 排查过程

### 第一层：draft 过滤逻辑

查看 `src/utils/content-utils.ts` 中获取文章列表的函数：

```typescript
const allBlogPosts = await getCollection("posts", ({ data }) => {
    return import.meta.env.PROD ? data.draft !== true : true;
});
```

问题出在这里。`import.meta.env.PROD` 在本地开发环境（`pnpm dev`）下是 `false`，此时过滤函数恒返回 `true`——**所有文章包括草稿全部显示**。所以本地预览时永远看到全部文章，分类计数也是错的。

修复很简单：去掉环境判断，始终过滤草稿。

```typescript
return data.draft !== true;
```

### 第二层：为什么部署到服务器后还不对？

代码改完、部署成功后，用户说还是看到旧内容。这就不是 draft 逻辑的问题了——应该是浏览器缓存。

检查服务器返回的响应头：

```
Cache-Control: no-cache
```

`no-cache` 的语义是"可以缓存，但使用前必须向服务器验证"。大部分浏览器会遵守这个规则，但 Safari 等浏览器的 **bfcache（后退缓存）** 在回退导航时会直接使用页面快照，根本不发验证请求。

将 HTML 的缓存策略改为更严格的 `no-store`：

```nginx
# 旧
location ~* \.html$ {
    expires -1;
    add_header Cache-Control "no-cache";
}

# 新
location ~* \.html$ {
    add_header Cache-Control "no-store, must-revalidate";
}
```

`no-store` 告诉浏览器**完全不要缓存页面**，每次导航都从服务器获取最新内容。

### 第三层：配置怎么每次部署都丢了？

配置写好后，部署一次博客——配置又丢了，容器退回到默认 nginx 配置。

查 `deploy-server.yml` 的 SCP 步骤：

```yaml
- uses: appleboy/scp-action@v0.1.7
  with:
    source: "dist/"
    target: "/path/to/blog"
    rm: true
```

`rm: true` 每次部署**清空整个目标目录**再上传。而我把 nginx 配置放在了 `/path/to/blog/conf/` 下——正好被清掉。

修复：把 nginx 配置文件移出 SCP 的打击范围。

```
/path/
├── blog/          # SCP 目标（rm:true 清空这里）
│   └── dist/      # 博客静态文件
└── nginx/
    └── conf/
        └── default.conf   # nginx 配置（SCP 碰不到）
```

## 经验总结

| 问题 | 教训 |
|------|------|
| draft 过滤 | `import.meta.env.PROD` 的判断会让草稿在开发环境可见，计数也受影响 |
| HTML 缓存 | `no-cache` 不如 `no-store` 严格，bfcache 会绕过验证 |
| 配置文件被删 | SCP 的 `rm: true` 清空整个目标目录，配置必须放在外面 |

三个问题单独看都不复杂，但串在一起会导致"明明部署成功了，为什么还是旧内容"的困惑——每个环节都以为自己没做错，合起来效果就不对。
