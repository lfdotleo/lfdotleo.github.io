---
title: "Build_blog_by_hugo"
date: 2021-10-17T11:46:41+08:00
draft: false
hideToc: false
enableToc: true
enableTocContent: true
author: dotleo
tags:
- hugo
categories:
- hugo

---

## hugo 安装

Mac 安装 hugo 命令如下：

```
brew install hugo
```

运行如下命令验证：

```
hugo version
```

## 创建一个新的站点

可以先创建一个文件夹，然后 `cd` 到该文件夹，运行 `hugo new site .`；也可以直接运行 `hugo new site <hugo_dir>`，这样可以在当前目录下创建一个名为 `<hugo_dir>` 的 hugo 文件夹。

## 添加一个主题

主题可以在 <https://themes.gohugo.io/> 中找，最终我选择了 Zzo 这个主题。

```
cd <hugo_dir>
git init
git submodule add https://github.com/zzossig/hugo-theme-zzo.git themes/zzo
```

其中，`git submodule add https://github.com/zzossig/hugo-theme-zzo.git themes/zzo` 使用自己选择的主题提供的命令。

## 修改配置

这步其实不同的主题差异性很大，以 Zzo 为例，首先需要将 `<hugo_dir>/themes/zzo/exampleSite` 目录下的 config, content, resources, static 文件夹拷贝到 `<hugo_dir>` 下进行覆盖替换。主要修改的就是这几个目录。

config，修改最多的目录，主要存各种配置文件；content，里面存放这发布后的内容，发布前需要清除默认的内容，换成自己的内容即可；static 里面主要修改一些 logo 等图片资源。

config 中的修改主要参考 <https://zzo-docs.vercel.app/zzo/configuration/configfiles/>

static 主要修改 logo 和 favicon,主要参考 <https://zzo-docs.vercel.app/zzo/userguide/favicon/>

*注意，`themes/zzo/` 下的文件不要改动，如果确实需要改动，可以在 `<hugo_dir>` 下找到对应的文件修改，如果不存在可以 copy 一份进行修改。*

## 本地运行

`cd <hugo_dir>` 后，运行 `hugo server`，然后访问 `http://localhost:1313/` 就可以看到修改效果了。

## 托管

修改效果满意后，可以托管到如 GitHub Pages 上，主流的托管方式有两种：

1. 源目录在本地，构建的静态文件托管到远端
2. 源目录和构建的静态文件都在远端，可以通过 GitHub Actions 进行自动构建。

不论哪种方式托管，首先需要在 GitHub 上创建一个 `<yourname>.github.io` 名字的仓库，将构建好的静态文件 push 到这个仓库才可以搭建成功。

### 只托管静态文件

将 public 目录作为 submodule 提交的 GitHub Pages 仓库。

```
cd <hugo_dir>
rm -rf public
git submodule add -b master https://github.com/<yourname>/<yourname>.github.io.git public
hugo -t <yourtheme>
cd public
git add .
git commit -m "first commit"
git push origin main
```

这些操作有点麻烦，可以采用官方提供的脚本，将下面内容保存到 `deploy.sh` 中。

```
#!/bin/sh

# If a command fails then the deploy stops
set -e

# Print out commands before executing them
set -x

printf "\033[0;32mDeploying updates to GitHub...\033[0m\n"

# Build the project.
hugo -t <yourtheme>

# Go To Public folder
cd public

# Add changes to git.
git add .

# Commit changes.
msg="rebuilding site $(date)"
if [ -n "$*" ]; then
    msg="$*"
fi
git commit -m "$msg"

# Push source and build repos.
git push origin main

# Back to the origin folder
# cd ..

# rm -rf public
```

接下来就可以使用 `./deploy.sh "Your optional commit message"` 提交静态页面到 <yourname>.github.io 上。

### 使用 GitHub Actions 构建

#### 配置 github_token

- 生成个人令牌：GitHub 账号下的 Setting -> Developer setting -> Personal access tokens-> Generate，生成并记录下来。
- 添加 secret ：回到 GitHub 账号 yourname.github.io 仓库下的 settings -> secret -> add。添加进刚才生成 token，要特别注意变量名 Name 要设置为"GITHUB_TOKEN" 的格式并记录下来，以便后面配置 Action 时 yaml 文件的调用。

#### 配置 GitHub Actions

在 yourname.github.io 仓库下，选择 "Actions"，新建 Action -> Simple workflow ，之后会跳转到一个编辑 yaml 文件的页面，修改 yml 文件名为 gh-pages.yml。会在仓库的 main 分支下生成 `.github/workflows/gh-pages.yaml` 文件。

```
name: github pages

on:
  push:
    branches:
      - main  # Set a branch to deploy
  pull_request:

jobs:
  deploy:
    runs-on: ubuntu-20.04
    steps:
      - uses: actions/checkout@v2
        with:
          submodules: true  # Fetch Hugo themes (true OR recursive)
          fetch-depth: 0    # Fetch all history for .GitInfo and .Lastmod

      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v2
        with:
          hugo-version: 'latest'
          # extended: true

      - name: Build
        run: hugo --minify

      - name: Deploy
        uses: peaceiris/actions-gh-pages@v3
        if: github.ref == 'refs/heads/main'
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./public
```

如果需要自定义域名指向，在最后一行 `public_dir` 下面加上 `cname: <yourname.github.io>` 即可。

#### 将 GitHub Pages 分支设置为 gh_pages

再到 Settings -> Pages 选项下，修改 Source 源分支为 gh-pages，点击 save。

#### 添加 .gitignore 文件

为了防止将 public 目录也 commit 到代码库中，需要在 `<hugo_dir>` 中添加一个 `.gitignore` 文件。

```
.DS_Store
*/.DS_Store
public/
```

加 DS_Store 相关是因为 MacOS 会自动生成这些文件，如果不是 MacOS 可以删掉。

#### 将 <hugo_dir> 下的仓库提交到 GitHub

*这里假定 GitHub 上新建的 yourname.github.io 仓库中没有任何内容。*

```bash
cd <hugo_dir>
rm -rf public/
git commit -m "first commit"
git branch -M main
git remote add origin git@github.com:<yourname>/<yourname.github.io>
git push -u origin main
```

提交到 main 分支后，脚本会自动构建到 gh-pages 分支，访问 <https://yourname.github.io> 就可以看到 Blog。

## 搭建遇到的问题

在搭建中遇到了一些问题，记录下避坑。

### 主题问题

主要是在本地配置中，遇到的一些与修改主题相关的问题。

#### 多语言问题

因为 Zoo 主题支持多语言，所以打算弄一个中文，进行下面设置：

- 在 config.toml 中设置 `defaultContentLanguage = "cn"`
- 在 languages.toml 中配置了 `cn`
- 在 content 目录下简历 cn 目录

但是折腾半天，启动后可以选择到该语言也能正常显示内容，就是样式和 js 显示异常。

后来发现 `theme/zzo/i18n/` 下面关于中文的一些文件将中文定义为 `zh`，将上面这些配置中的 cn 改成 zh 后，解决了该问题。

#### 评论问题

Zzo 主题支持多种评论系统，只需要配置下就可以，朋友推荐使用 utterances，在 params.toml 有配置，如下:

```
[utterances]       # https://utteranc.es/
  owner = "codekeeperjava"              # Your GitHub ID
  repo = "https://codekeeperjava.github.io"               # The repo to store comments
```

这样配置无法显示 utterances, 需要将 repo 中的 `https://` 去掉。

### 托管问题

托管的选型问题

#### 选择 GitHub Actions 还是只托管静态文件

GitHub Actions 的方式，可以把源文件和构建好的文件都 push 到 GitHub 上备份还是很方便的，去另一台电脑 clone 下仓库就可以进行编辑。但同时，如果自己内容较多时，可能被别人直接 download 将所有“财产”拖走。

只托管静态文件，一般需要将源文件用 dropbox 等同步，并且比 GitHub Actions 还要多很多步骤：构建，将 publich push 到 GitHub。当然这种方式比较安全，别人不能直接 download 自己的“财产”。

综合考虑，目前我自己只有几篇博文，所以别人可能也没有兴趣，所以选择了方便的方式(GitHub Actions)。

## 参考

- [Hugo+GitHub Action+GitHub Pages搭建个人博客](https://pingyangblog.com/setup-hugo-blog/#%E4%BD%BF%E7%94%A8github-pages%E5%AE%9E%E7%8E%B0%E8%AE%BF%E9%97%AE)
- [把博客从 Hexo 迁移到 Hugo](https://jdhao.github.io/2018/10/10/hexo_to_hugo/)
- [给 Hugo even 主题添加 utterances 评论系统](https://jdhao.github.io/2021/08/15/hugo_even_add_utterance/)
- [使用Hugo和GitHub搭建博客](https://zhuanlan.zhihu.com/p/150095964)









