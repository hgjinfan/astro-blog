---
title: 把博客嵌进首页后，点击导航让整个标签页崩了
description: Astro 的 ClientRouter 在跨站 iframe 里做客户端导航时，会直接把 Chromium 渲染器搞崩——一次完整的排查记录。
pubDatetime: 2026-10-19T18:00:00+08:00
tags:
  - astro
  - debug
draft: true
---

## 背景

博客（AstroPaper 搭的，部署在 Cloudflare Pages）用 Glance 的 `iframe` 小组件嵌进了个人首页。看着挺美，直到我在 iframe 里点了博客导航栏的「Posts」——整个浏览器标签页直接崩了（Aw, Snap!）。

## 现象

- iframe 内点任何导航链接 → 标签页渲染进程死亡
- 直接在新标签页打开 `blog.kyriexu.com/posts/` → 一切正常

## 排查过程

1. **复现**：两次独立复现，点击动作返回成功，随后标签页内所有读取全部失败，确认是渲染进程崩溃，不是跳错页面。
2. **查源码**：博客没有开 ViewTransitions，链接也没有奇怪的 `target`，一时看不出问题。
3. **隔离测试**：拿掉 Glance，用一个纯 HTML 页面只放一个裸 `<iframe src="https://blog.kyriexu.com">`，点 Posts——**照样崩**。说明问题在博客本身，不在 Glance。
4. **抓包看 HTML**：在 `/posts/` 的 HTML 里发现了一个可疑的脚本：`/_astro/ClientRouter.astro_astro_type_script_index_0_lang.*.js`——博客开了 Astro 的 `<ClientRouter />`（`astro:transitions` 的客户端路由，SPA 式导航）。

## 根因

Astro 的 ClientRouter 会拦截站内链接点击，做客户端 DOM 替换式导航。在**跨站 iframe** 里做这件事时，会触发 Chromium 渲染器崩溃。直接访问没问题，是因为顶层文档走客户端导航是正常的。

## 修复

`src/layouts/Layout.astro` 里加一段内联脚本，检测是否在 iframe 中：

```js
var inIframe = false;
try {
  inIframe = window.self !== window.top;
} catch (e) {
  inIframe = true; // 跨域 iframe 里读 window.top 会抛异常，这本身就是信号
}
if (inIframe) {
  document.addEventListener("DOMContentLoaded", function () {
    document.querySelectorAll("a[href]").forEach(function (a) {
      var href = a.getAttribute("href") || "";
      if (href.charAt(0) === "/" || href.indexOf("blog.kyriexu.com") !== -1) {
        a.setAttribute("data-astro-reload", "");
      }
    });
  });
}
```

`data-astro-reload` 是 Astro 的官方属性，会让 ClientRouter 对该链接强制整页跳转。效果：**直接访问博客保留 SPA 过渡动画，被 iframe 嵌入时自动降级为整页跳转，不再崩溃**。

## 顺手踩的另一个坑

修博客要推代码到 GitHub，用的 deploy key。结果推到一半开始报 `Permission denied`——排查发现运行环境的 shell 是 root 身份但 `HOME=/home/hatch`，ssh 读的是 `/root/.ssh/`，而 key 放在 `/home/hatch/.ssh/`。更坑的是，云主机的 `/root` 在 VM 重置后会被清空，之前配好的 key 和 known_hosts 全丢了。

根治：给仓库配 `git config core.sshCommand`，显式指定 `-i /home/hatch/.ssh/id_ed25519 -o UserKnownHostsFile=/home/hatch/.ssh/known_hosts`，所有东西都放在持久化的 home 目录里，VM 怎么重置都不怕。

## 教训

1. 第三方嵌入（iframe）场景下，SPA 客户端路由是高危项，出问题先怀疑它。
2. `window.top` 读不到本身就是一种检测手段——异常即信号。
3. 云主机上任何放在 `/root`、`/etc`、`/tmp` 里的东西都当成一次性的，重要配置进 home 目录。
