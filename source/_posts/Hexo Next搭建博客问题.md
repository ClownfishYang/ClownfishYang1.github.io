---
title: Hexo Next搭建博客问题
tags:
- Hexo
- Next
- blog
categories:
- 技术
date: 2020-11-03 17:45:16
---
之前已经搭建过了一次，后面一直没有使用，最近又心学来潮想弄起来记录点东西。
搭建网上已经很多了，我就不啰嗦，主要记录下我遇到的问题以及做的一些修改，后续也会持续更新。
<!--more-->

## 遇到的问题

### hexo 命令无法执行
``` bash
$ npm install -g hexo-cli
$ hexo
'hexo' 不是内部或外部命令，也不是可运行的程序
或批处理文件。
```
好多博客都没有说明，问题是因为找不到`hexo-cli`的执行文件，需要将`node_modules`目录下的`.bin/hexo`加入到环境变量中，我没有加，直接使用`./node_modules/.bin/hexo`执行。

### 分类页、标签页和关于页
``` bash
$ hexo new page categories
$ hexo new page tags
$ hexo new page about
# 分别对生成的categories、tags和about文件夹下的index.md文件添加type
# type: categories
# type: tags
# type: about
# 后面文章使用是只需要在head部添加categories和tags属性即可(yml语法)
---
title: 搭建博客
tags:
- Hexo
- Cxo
- blog
categories:
- 技术
date: 2020-11-03 17:45:16
---
```

### 首页显示摘要（不显示全文）
默认会显示全文，查看时很不方便，需要只显示摘要或者文章部分。
首先需要将`themes/_config.yml`中的`excerpt_description`属性启用，默认是true。
```
# Automatically excerpt description in homepage as preamble text.
excerpt_description: true
```
方法有两种，一种是摘要，另一种是截取文章部分。
```
# 在文章头部添加description，内容就会被显示在首页上。
---
title:
date:
description: 这是显示在首页的概述，正文内容均会被隐藏。
---

# 在文章中加<!--more-->标签，标签前的内容就会被显示在首页上。
显示的内容
<!--more-->
隐藏的内容
```


### 动画效果
添加动画效果时需要安装额外的依赖，但是添加完以后会有依赖版本不兼容，所以我使用的CDN。
错误如下:
```
$ hexo s
INFO  Validating config
INFO  Start processing
FATAL {
  err: TypeError: Cannot read property 'enable' of undefined
      at \themes\next\scripts\filters\comment\changyan.js:11:23
      at Filter.execSync (\node_modules\hexo\lib\extend\filter.js:84:30)
      at Hexo.execFilterSync (\node_modules\hexo\lib\hexo\index.js:485:31)
      at module.exports (\themes\next\scripts\events\lib\injects.js:58:8)
      at Hexo.<anonymous> (\themes\next\scripts\events\index.js:9:27)
      at Hexo.emit (events.js:327:22)
      at Hexo._generate (\node_modules\hexo\lib\hexo\index.js:452:10)
      at \node_modules\hexo\lib\hexo\index.js:356:19
      at tryCatcher (\node_modules\bluebird\js\release\util.js:16:23)
      at Promise._settlePromiseFromHandler (\node_modules\bluebird\js\release\promise.js:517:31)
      at Promise._settlePromise (\node_modules\bluebird\js\release\promise.js:574:18)
      at Promise._settlePromise0 (\node_modules\bluebird\js\release\promise.js:619:10)
      at Promise._settlePromises (\node_modules\bluebird\js\release\promise.js:699:18)
      at Promise._fulfill (\node_modules\bluebird\js\release\promise.js:643:18)
      at Promise._resolveCallback (\node_modules\bluebird\js\release\promise.js:437:57)
      at Promise._settlePromiseFromHandler (\node_modules\bluebird\js\release\promise.js:529:17)
} Something's wrong. Maybe you can find the solution here: %s https://hexo.io/docs/troubleshooting.html
```
动画启用配置：
```
# 主题配置文件(themes/_config.yml)
# 页面加载动画(页面顶部的加载进度条)
pace: true
# Themes list:
# pace-theme-big-counter | pace-theme-bounce | pace-theme-barber-shop | pace-theme-center-atom
# pace-theme-center-circle | pace-theme-center-radar | pace-theme-center-simple | pace-theme-corner-indicator
# pace-theme-fill-left | pace-theme-flash | pace-theme-loading-bar | pace-theme-mac-osx | pace-theme-minimal
pace_theme: pace-theme-minimal

# 顶部阅读进度条
reading_progress:
  enable: true
  # Available values: top | bottom
  position: top
  color: "#37c6c0"
  height: 2px

# three 背景动画
# three_waves
three_waves: true
# canvas_lines
canvas_lines: true
# canvas_sphere
canvas_sphere: true

# Canvas-nest 背景动画
canvas_nest:
  enable: true
  onmobile: true # display on mobile or not
  color: "0,0,255" # RGB values, use ',' to separate
  opacity: 0.5 # the opacity of line: 0~1
  zIndex: -1 # z-index property of the background
  count: 99 # the number of lines

# canvas-ribbon 背景动画
canvas_ribbon:
  enable: true
  size: 300
  alpha: 0.6
  zIndex: -1

# 图片弹出效果
fancybox: true
```

方法一：安装依赖文件
```
# 进入主题目录
$ cd themes/next

# 从GitHub下载依赖文件
# 页面加载动画
$ git clone https://github.com/theme-next/theme-next-pace source/lib/pace

# 顶部阅读进度条
$ git clone https://github.com/theme-next/theme-next-reading-progress source/lib/reading_progress

# three 背景动画
$ git clone https://github.com/theme-next/theme-next-three source/lib/three

# canvas-nest 背景动画
$ git clone https://github.com/theme-next/theme-next-canvas-nest source/lib/canvas-nest

# canvas-ribbon 背景动画
$ git clone https://github.com/theme-next/theme-next-canvas-ribbon source/lib/canvas-ribbon

# 图片弹出效果
$ git clone https://github.com/theme-next/theme-next-fancybox3 source/lib/fancybox
```

方法二：使用CDN
```
# 从GitHub下载依赖文件
# 页面加载动画(页面顶部的加载进度条)
vendors:
  # Internal version: 1.0.2
  # pace: //cdn.jsdelivr.net/npm/pace-js@1/pace.min.js
  # pace: //cdnjs.cloudflare.com/ajax/libs/pace/1.0.2/pace.min.js
  # pace_css: //cdn.jsdelivr.net/npm/pace-js@1/themes/blue/pace-theme-minimal.css
  # pace_css: //cdnjs.cloudflare.com/ajax/libs/pace/1.0.2/themes/blue/pace-theme-minimal.min.css

# 顶部阅读进度条
vendors:
  # reading_progress: //cdn.jsdelivr.net/gh/theme-next/theme-next-reading-progress@1/reading_progress.min.js

# three 背景动画
vendors:
  # Internal version: 1.0.0
  # three: //cdn.jsdelivr.net/gh/theme-next/theme-next-three@1/three.min.js
  # three_waves: //cdn.jsdelivr.net/gh/theme-next/theme-next-three@1/three-waves.min.js
  # canvas_lines: //cdn.jsdelivr.net/gh/theme-next/theme-next-three@1/canvas_lines.min.js
  # canvas_sphere: //cdn.jsdelivr.net/gh/theme-next/theme-next-three@1/canvas_sphere.min.js

# canvas-nest 背景动画
  # Canvas-nest:
  canvas_nest: //cdn.jsdelivr.net/gh/theme-next/theme-next-canvas-nest@1/canvas-nest.min.js
  canvas_nest_nomobile: //cdn.jsdelivr.net/gh/theme-next/theme-next-canvas-nest@1/canvas-nest-nomobile.min.js

# canvas-ribbon 背景动画
  # Internal version: 1.0.0
  # canvas_ribbon: //cdn.jsdelivr.net/gh/theme-next/theme-next-canvas-ribbon@1/canvas-ribbon.js

# 图片弹出效果
  # FancyBox
  # jquery: //cdn.jsdelivr.net/npm/jquery@3/dist/jquery.min.js
  # fancybox: //cdn.jsdelivr.net/gh/fancyapps/fancybox@3/dist/jquery.fancybox.min.js
  # fancybox_css: //cdn.jsdelivr.net/gh/fancyapps/fancybox@3/dist/jquery.fancybox.min.css
```

### 头像动画
旧版本的动画效果需要修改CSS，新版本只需要配置就可以实现。
```
# Sidebar Avatar
avatar:
  # Replace the default image and set the url here.
  url: img_url #/images/avatar.gif
  # If true, the avatar will be dispalyed in circle.
  rounded: true
  # If true, the avatar will be rotated with the cursor.
  rotated: true
```
头像的大小可以通过修改主题`style`或者修改`themes/source/css/_variables/base.styl`的css。
```
# 修改base.styl
$site-author-image-width              = 128px;
$site-author-image-border-width       = 2px;
$site-author-image-border-color       = $black-dim;
```

### 网页图标
网页图标可以通过修改配置实现,图标地址存放在`themes/source/images`目录下。
```
favicon:
  small: /images/favicon-16x16-my.png
  medium: /images/favicon-32x32-my.png
  apple_touch_icon: /images/apple-touch-icon-my.png
  safari_pinned_tab: /images/logo.svg
```

### Github角落
[Github角落实现效果](https://tholman.com/github-corners/)
旧版本的动画效果需要修改`themes/next/layout/_layout.swig`文件，新版本只需要配置就可以实现。
```
# 旧版本，在<div class="headband"></div>下添加github-corners提供的svg代码
<div class="headband"></div>
<a ...></a>

# 新版本
# `Follow me on GitHub` banner in the top-right corner.
github_banner:
  enable: false
  permalink: https://github.com/yourname
  title: Follow me on GitHub
```

### 标签图标
默认标签使用#号，旧版本图片需要修改`/themes/next/layout/_macro/post.swig`文件，新版本只需要配置就可以实现。
```
# 旧版本
# <a href="{{ url_for(tag.path) }}" rel="tag"># {{ tag.name }}</a>
<a href="{{ url_for(tag.path) }}" rel="tag"><i class="fa fa-tag"></i> {{ tag.name }}</a>

# 新版本
# Use icon instead of the symbol # to indicate the tag at the bottom of the post
tag_icon: true
```
## 部署问题

### hexo deploy分支管理
我使用了`develop`和`master`两个分支,`develop`用于存储未发表的文章，`master`用于存储已发表的文章，但是使用`hexo deploy`命令部署的话，我的`master`分支将会被覆盖成静态文件，但如果不使用`hexo deploy`命令进行部署的话，就需要手动进行部署，并且我发现手动部署时文章中的更新时间不会被修改过来。
并且我还需要作为io的仓库，能够看到我现有维护的代码，能不能新增一个分支`pages`专门用来生成静态文件，作为io地址访问。
首先修改站点的配置文件`_config.yml`:
```
deploy:
  type: git
  repo: https://github.com/username/username.github.io.git
  branch: pages
```
然后执行`hexo deploy`命令，成功之后会创建一个`pages`的分支； 
然后到github仓库中的`setting`中找到[GitHub Pages](https://docs.github.com/cn/free-pro-team@latest/github/working-with-github-pages/configuring-a-publishing-source-for-your-github-pages-site)，将Source 的Branch 选为刚刚创建的`pages`，然后保存即可。 

## 参考博客
[<i class="fas fa-paperclip"></i> GitHub + Hexo](https://zhuanlan.zhihu.com/p/26625249)
[<i class="fas fa-paperclip"></i> Cxo Theme](https://github.com/Longlongyu/hexo-theme-Cxo)
[<i class="fas fa-paperclip"></i> Next Theme](https://github.com/next-theme/hexo-theme-next)
[<i class="fas fa-paperclip"></i> Hexo-NexT 主题个性优化](https://guanqr.com/tech/website/hexo-theme-next-customization/)