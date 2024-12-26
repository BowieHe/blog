---
title: Hexo GitHub Pages部署流程
date: 2024-03-26
categories:
    - 教程
tags:
    - Hexo
    - GitHub Pages
comments: true
---

最近可能是因为 AI 时代的崛起, 越来越多的个人开发者开始在这个市场上崭露头角. 同时也在这些个人开发者身上看到了很多充满个人气息的博客信息. 因此也打算趁这个机会吧以前弄到一半荒废的 GitHub Pages 页面重新升级一遍. 同时发现一两年前大家强推的 `Hugo` 好像今年很少提及, 搜索结果关联的都是 `Hexo`.

因此在这里记录一下 Hexo 搭建的过程中碰到的问题, 以及一些简单的教程

<!--more-->

文中的一些配置参考了下面两位的文章

> [Hexo 的 Next 主题优化教程](https://cloud.tencent.com/developer/article/2317861) >[从零开始搭建 Hexo 博客简明教程](https://www.philoli.com/building-a-blog-from-scratch/)

# 安装

### NVM 安装

如果在这个之前没有安装过 npm 的话,需要先通过下面的命令来安装 npm

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.1/install.sh | bash
或者
wget -qO- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.1/install.sh | bash
```

安装脚本会将 NVM 仓库克隆到 \~/.nvm 目录，并更新您的配置文件（\~/.bash_profile、\~/.zshrc、\~/.profile 或 \~/. bashrc）。

通过命令 `nvm --version` 来验证是否安装成功 (比如我的返回就是 `0.35.3`)

### 安装 NPM

NPM 通常和 Node. Js 一起安装, 在安装完成 nvm 之后, 可以通过如下的命令安装最新版本 NPM

```bash
nvm install node

nvm install <version>
```

## 安装 Hexo

首先需要安装 `hexo` 的框架

```bash
$ npm install -g hexo-cli
```

然后新建一个文件夹, 比如 blog (**注意即使是通过 GitHub Pages 发布,这里的文件夹不需要和 github. Io 名字相同, 可以是任意名字, 不然 GitHub Pages 发布的静态文件结构和当前项目完全不同**)

```bash
mkdir blog
cd blog
# 初始化 hexo模板
hexo init
# 安装和运行必须的npm包
npm install
# 安装完成之后执行
hexo server
```

按照上面的步骤安装完成之后, 可以通过执行命令 `hexo server` 来看是不是正确安装.

此时命令行会返回下面的命令, 或者访问 `localhost:4000` 来查看页面

```bash
INFO  Validating config
INFO  Start processing
INFO  Hexo is running at http://localhost:4000/ . Press Ctrl+C to stop.
```

# 安装 NexT 主题

因为 NexT 主题截止目前已经长期不更新, 而且为了方便简单维护主题相关信息, 我选择直接复制 next 文件到当前项目, 而不是通过 Git 版本控制的方式来维护

```bash
cd ~/Downloads/
git clone https://github.com/next-theme/hexo-theme-next.git
cp -r hexo-theme-next blog/themes/next
rm -rf hexo-theme-next
```

修改根目录下的 `_config.yml` 文件中 theme. 从 `landscape` 修改为 `next`

打开 `theme/next/_config.yml` 文件, 找到 `scheme` 字段, 修改成自己喜欢的, 我选择的是 `Gemini`

然后可以执行命令 `hexo server` 查看是否正确设置了主题

## 配置

### 站点基本信息

这一块在根目录的 `_config.yml` 文件夹下面. 首先配置默认的基本用户信息

```yaml
title: Bowie
subtitle: "小何的岛🏝️"
description: "前端/后端/大数据/AI开发,记录日常所思所想,阅读感悟"
keywords:
author: Bowie
language: zh-CN
timezone: "Asia/Shanghai"
```

### 头像

同时可以支持添加头像. 可以添加图片到 `source/images` 文件夹中 (文件夹不存在可以新建).

寻找到 `avatar` 配置, 修改为

```yaml
avatar:
# Replace the default image and set the url here.
	url: /images/avatar.png
```

### 完善页面

在 `themes/next/_config.yml` 文件中, 找到 `menu`, 把需要添加的菜单前面取消注释

```yaml
menu:
	home: / || fa fa-home
	#about: /about/ || fa fa-user
	#tags: /tags/ || fa fa-tags
	categories: /categories/ || fa fa-th
	archives: /archives/ || fa fa-archive
	#schedule: /schedule/ || fa fa-calendar
	#sitemap: /sitemap.xml || fa fa-sitemap
	#commonweal: /404/ || fa fa-heartbeat
```

同时执行下面的命令来创建对应的页面

```bash
hexo new page categories

hexo new page about
```

对于 `categories` 和 `tags` 页面, 需要设置对应的类型, 分别为 `categories` 和 `tags`. 同时注意 `:` 后面的空格

```markdown
---
title: categories
date: 2024-12-26 14:48:30
type: "categories"
comments: false
---
```

### 设置社交链接

通过编辑 `themes/next/_config.yml` 文件中字段 `social`, 对于想要添加的取消注释, 并且在下面的 `social_icons` 添加对应的信息

```yaml
# Social links
social:
  GitHub: https://github.com/your-user-name || fab fa-github
  E-Mail: mailto:yourname@gmail.com || fa fa-envelope
  #Weibo: https://weibo.com/yourname || fab fa-weibo
  #Google: https://plus.google.com/yourname || fab fa-google
  Twitter: https://x.com/your-user-name || fab fa-twitter

social_icons:
	enable: true
	Twitter: twitter
	GitHub: github
	E-Mail: envelope
```

### 创建新文章

在根目录下执行命令 `hexo new "Article"`, 会在 `source/_posts` 文件夹下面自动生成一个 `Article.md` 文件 (其中 article 的名字是文章的名字)

文章遵从 markdown 的规则, 同时可以设置文章的起始字段

```markdown
title 文章的标题
date 创建日期 （文件的创建日期 ）
updated 修改日期 （ 文件的修改日期）
comments 是否开启评论 true
tags 标签
categories 分类
permalink url 中的名字（文件名）
```

同时因为 hexo 中文章并不会自动的折叠, 因此如果希望有一个 readMore 的折叠按钮, 可以通过在文章中间添加文本 `<!--more-->` 来实现

在安装完成上述之后, 可以通过命令 `hexo s` 查看

### 部署

好了! 终于到可以部署到 GitHub Pages 了, 这里也很简单, 只需要几个设置

1. 安装 hexo 部署工具

运行下面的命令来安装

```bash
npm install hexo-deployer-git --save
```

2. 创建一个 Github. Io 的仓库目录, 这个仓库名字需要是用户名字开头, 比如 `username.github.io`
3. 修改根目录的 `_config.yml` 文件中最下面的 deploy, 修改成如下所示 (**因为 GitHub 取消了密码认证,因此这里需要是 git 的地址** )

```yaml
deploy:
	type: git
	repository: git@github.com:username/username.github.io.git
	branch: main
```

4. 然后执行下面的命令, 就可以推送到 GitHub pages 了. 如果用 GitHub 来管理配置和内容的话, 记得添加和提交到 GitHub

```bash
hexo generate #生成静态文件
hexo deploy #推送到服务器

git add .
git commit -m "docs:message"
git push
```
