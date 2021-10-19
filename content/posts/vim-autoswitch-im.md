---
title: Vim 自动切换中英文
date: 2021-10-16 16:40:00
draft: false
hideToc: false
enableToc: true
enableTocContent: true
author: dotleo
tags:
- vim
categories:
- vim
series:
- vim
---

使用 vim 最大的痛点就是使用时中英文切换的问题，但插件可以帮我们自动切换，进入 normal 模式则自动切换为英文，进入 insert 模式恢复之前的输入法。

目前了解有 3 种方案：`smartim` 、`vim-xkbswitch` 和 `vim-barbaric`

## smartim

[https://github.com/ybian/smartim](https://github.com/ybian/smartim)

该插件支持 Windows 和 MacOS，需要依赖 `im-select` ：

```bash
curl -Ls https://raw.githubusercontent.com/daipeihust/im-select/master/install_mac.sh | sh
```

然后安装该插件即可：

```bash
Plug 'ybian/smartim'
```

`im-select` 使用 [`com.apple.keylayout.US`](http://com.apple.keylayout.us/) 作为默认输入法，因此需要确保有该英文输入法。

该插件会在 CPU 较高时延迟较大，表现为当从 normal 模式进入 insert 模式后，需要等待 < 1s 才能进行中文输入。我经常碰到这种情况，并为此困惑。

## vim-xkbswitch

[https://github.com/lyokha/vim-xkbswitch](https://github.com/lyokha/vim-xkbswitch)

该插件暂时没有使用，朋友推荐并且 star 数比 smartim 还要高，等 smartim 出现问题时作为备选方案。

## vim-barbaric

[https://github.com/rlue/vim-barbaric](https://github.com/rlue/vim-barbaric)

同上

## 参考

- [https://jdhao.github.io/2021/02/25/nvim_ime_mode_auto_switch/](https://jdhao.github.io/2021/02/25/nvim_ime_mode_auto_switch/)

