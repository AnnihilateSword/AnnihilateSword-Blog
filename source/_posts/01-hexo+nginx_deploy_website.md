---
title: 【起始之章】Hexo+Nginx部署网站
tags: [hexo]
date: 2024-12-12
updated: 2024-12-12
cover: /res/img/post/01-hexo+nginx_deploy_website/17.png
top_img: /res/img/site/background.png
---

![](/res/img/post/01-hexo+nginx_deploy_website/17.png)

# 前言

以 Butterfly 主题为例，使用 nginx 进行部署到云服务器上

可以参考 Butterfly 文档的 👉[Butterfly快速开始](https://butterfly.js.org/posts/21cfbf15/)

# 一、安装依赖

## 1.下载 Git 和 Nodejs

git: 👉https://git-scm.com/
nodejs: 👉https://nodejs.org/en

<br>

## 2.局部安装 hexo

找一个用于存放 hexo 开发的位置

```shell
npm install hexo
```

<br>

## 3.hexo 常用命令

```shell
# 清理数据
npx hexo cl
# 测试
npx hexo s
# 生成
npx hexo g
```

<br>
<br>

# 二、开始建站

## 2.1 初始化

```shell
npx hexo init AnnihilateSword
cd AnnihilateSword
npm install
```

PS：AnnihilateSword 作为名称创建文件夹，这里称为 hexo 的根目录

<br>

## 2.2 下载并应用 butterfly 主题

进入刚刚创建的文件夹 AnnihilateSword

```shell
git clone -b master https://github.com/jerryc127/hexo-theme-butterfly.git themes/butterfly
```

PS：这会将主题 clone 在 `AnnihilateSword/themes/` 目录下

修改 Hexo 根目录下的 `_config.yml`，把主题改为 butterfly

```yaml
theme: butterfly
```

<br>

## 2.3 安装插件

在 AnnihilateSword 目录即你的 hexo 根目录

```shell
npm install hexo-renderer-pug hexo-renderer-stylus --save
```

<br>

## 2.4 升级建议

为了减少升级主题后带来的不便，请使用以下方法（建议这样做，但也是可选的）

在 hexo 的根目录创建一个文件 `_config.butterfly.yml`，并把主题目录的 `_config.yml` 内容复制到 `_config.butterfly.yml` 去( 注意：复制的是主题的 `_config.yml`，而不是 hexo 的 `_config.yml`)

> &nbsp;
> 注意： 不要把主题目录的 `_config.yml` 删掉
>
> 注意： 以后只需要在 `_config.butterfly.yml` 进行配置就行。如果使用了 `_config.butterfly.yml`， 配置主题的 `_config.yml` 将不会有效果。
> &nbsp;

hexo 会自动合并主题中的 `_config.yml` 和 `_config.butterfly.yml` 里的配置，如果存在同名配置，会使用 `_config.butterfly.yml` 的配置，其优先度较高。

![](/res/img/post/01-hexo+nginx_deploy_website/1.png)

<br>

## 2.5 主题配置

### 2.5.1 目录自动编序号（不能容忍啊…）

修改 `主题配置文件`，这里以及后面的主题配置文件指 `_config.butterfly.yml`

```yml
# toc (目录)
toc:
  post: true
  page: false
  number: false
  expand: false
  # Only for post
  style_simple: false
  scroll_percent: true
```

PS：把目录中的 number 设置为 false，这样就不会自动给目录标题编号了，按自己的编号来

接下来就是主题配置，文档比我更详细：👉https://butterfly.js.org/posts/4aa8abbe/

可以按自己想要的配置，下面列出我目前所初步修改的内容：

1.站点配置文件 `_config.yml` 修改

```yml
# Site
title: AnnihilateSword
subtitle: Home
description: ''
keywords:
author: Jinkun Ou
language: zh-CN
timezone: ''

# URL
## Set your site url here. For example, if you use GitHub Page, set url as 'https://username.github.io/project'
url: http://annihilatesword.com/
```

2.菜单 | 目录

修改 `主题配置文件`

```yml
menu:
  Home: / || fas fa-home
  Archives: /archives/ || fas fa-archive
  Tags: /tags/ || fas fa-tags
  Categories: /categories/ || fas fa-folder-open
  List||fas fa-list:
    Music: /music/ || fas fa-music
  #   Movie: /movies/ || fas fa-video
  Link: /link/ || fas fa-link
  About: /about/ || fas fa-heart
```

3.高亮主题

修改 `主题配置文件`，这里设置的代码块 macStyle 是 false，开启代码高度限制 height_limit: 520

```yml
code_blocks:
  # Code block theme: darker / pale night / light / ocean / false
  theme: darker
  macStyle: false
  # Code block height limit (unit: px)
  # height_limit: false
  height_limit: 520
  word_wrap: false
```

4.社交图标

修改 `主题配置文件`

butterfly 支持 👉[font-awesome v6](https://fontawesome.com/icons?from=io) 图标

书写格式 `图标名：url || 描述性文字 || color`

```yml
# Social Settings (社交图标设置)
# formal:
#   icon: link || the description || color
social:
  # fab fa-github: https://github.com/xxxxx || Github || '#24292e'
  # fas fa-envelope: mailto:xxxxxx@gmail.com || Email || '#4a7dbe'
  fa-brands fa-zhihu: https://www.zhihu.com/people/AnnhiliateSword || Zhihu || '#4a7dbe'
  fa-brands fa-bilibili: https://space.bilibili.com/445428089 || Bilibili || '#4a7dbe'
```

5.网站图标 | 头像

修改 `主题配置文件`

```yml
# Favicon（网站图标）
favicon: /res/img/site/avatar.jpg

# Avatar (头像)
avatar:
  img: /res/img/site/avatar.jpg
  effect: false
```

> 这里的根目录相对于 hexo init 创建的项目目录，我这里是 `AnnihilateSword/source/res/img/site`

6.顶部图

修改 `主题配置文件`

```yml
# Disable all banner images
disable_top_img: false

# If the banner of page not setting, it will show the default_top_img
default_top_img: /res/img/site/background.png

# The banner image of index page
index_img: /res/img/site/background.png

# The banner image of archive page
archive_img: /res/img/site/background.png

# Note: tag page, not tags page
tag_img: /res/img/site/background.png

# The banner image of tag page, you can set the banner image for each tag
# Format:
#  - tag name: xxxxx
tag_per_img: /res/img/site/background.png

# Note: category page, not categories page
category_img: /res/img/site/background.png

# The banner image of category page, you can set the banner image for each category
# Format:
#  - category name: xxxxx
category_per_img: /res/img/site/background.png
```

PS：不要以下划线开头，比如 _res，会无法识别图片

7.TOC

在文章页，会有一个目录，用于显示 TOC。

修改 `主题配置文件`

```yml
toc:
  post: true
  page: true
  number: false
  expand: false
  # Only for post
  style_simple: false
  scroll_percent: true
```

设置 page 为 true

8.相关文章

PS：当文章封面设置为 false 时，或者没有获取到封面配置，相关文章背景将会显示主题色。

相关文章推荐的原理是根据文章 tags 的比重来推荐

修改 `主题配置文件`

```yml
# Related Articles
related_post:
  enable: true
  limit: 6 # Number of posts displayed
  date_type: created # or created or updated 文章日期显示创建日或者更新日
```

9.Footer 设置

博客年份

`since` 是一个来展示你站点起始时间的选项。它位于页面的最底部。

修改 `主题配置文件`

```yml
# --------------------------------------
# Footer Settings
# --------------------------------------
footer:
  owner:
    enable: true
    since: 2024
  custom_text:
  # Copyright of theme and framework
  copyright: true
```

页脚自定义文本

custom_text 是一个给你用来在页脚自定义文本的选项。通常你可以在这里写声明文本等，支持 HTML。

```yml
custom_text: annihilatesword.com 版权所有
```

10.侧边栏

修改 `主题配置文件`

```yml
# --------------------------------------
# Aside Settings
# --------------------------------------

aside:
  enable: true
  hide: false
  # Show the button to hide the aside in bottom right button
  button: true
  mobile: true
  # Position: left / right
  position: right
  display:
    archive: true
    tag: true
    category: true
  card_author:
    enable: true
    description:
    button:
      enable: true
      icon: fab fa-github
      text: Follow Me
      link: https://github.com/AnnihilateSword
  card_announcement:
    enable: true
    content: 🗡️幸得识君桃花面，从此阡陌多暖春。<br>🗡️天道无常，事在人为。<br>🗡️临渊慕鱼，不如退而结网。<br>🗡️ (2024) 逐渐死去的心已经复活。
```

11.运行时间

网页已运行时间

修改 `主题配置文件`

```yml
# Time difference between publish date and now
# Formal: Month/Day/Year Time or Year/Month/Day Time
# Leave it empty if you don't enable this feature
runtime_date: 2/19/2024 21:13:14
```

12. 右下角按钮

简体繁体互换

右下角会有简繁转换按钮。

修改 `主题配置文件`

```yml
# Conversion between Traditional and Simplified Chinese
translate:
  enable: true
  # The text of a button
  default: 简
  # the language of website (1 - Traditional Chinese/ 2 - Simplified Chinese）
  defaultEncoding: 2
  # Time delay
  translateDelay: 0
  # The text of the button when the language is Simplified Chinese
  msgToTraditionalChinese: '繁'
  # The text of the button when the language is Traditional Chinese
  msgToSimplifiedChinese: '簡'
```

13. 夜间模式

右下角会有夜间模式按钮

修改 `主题配置文件`，这里就默认 dark 主题了，不让切其他主题 button: false

```yml
# Dark Mode
darkmode:
  enable: true
  # Toggle Button to switch dark/light mode
  button: false
  # Switch dark/light mode automatically
  # autoChangeMode: 1  Following System Settings, if the system doesn't support dark mode, it will switch dark mode between 6 pm to 6 am
  # autoChangeMode: 2  Switch dark mode between 6 pm to 6 am
  # autoChangeMode: false
  autoChangeMode: false
  # Set the light mode time. The value is between 0 and 24. If not set, the default value is 6 and 18
  start:
  end:
```

<br>

### 2.5.2 搜索

> &nbsp;
> 记得运行 hexo clean
> 
> 把主题配置文件中 search 的 use 配置为 local_search
> &nbsp;

- 安装 👉[hexo-generator-searchdb](https://github.com/next-theme/hexo-generator-searchdb) 或者 👉[hexo-generator-search](https://github.com/PaicHyperionDev/hexo-generator-search)，根据文档去做配置

```shell
npm install hexo-generator-search --save

或者

npm install hexo-generator-searchdb
```

修改 `主题配置文件`

```yml
search:
  # Choose: algolia_search / local_search / docsearch
  # leave it empty if you don't need search
  use: local_search
  placeholder:

  # Algolia Search
  algolia_search:
    # Number of search results per page
    hitsPerPage: 6

  # Local Search
  local_search:
    enable: true
    # Preload the search data when the page loads.
    preload: false
    # Show top n results per article, show all results by setting to -1
    top_n_per_article: 1
    # Unescape html strings to the readable one.
    unescape: false
    CDN:
```

<br>

### 2.5.3 Mathjax（数学公式支持）

修改 `主题配置文件`

```yml
mathjax:
  # Enable the contextual menu
  enableMenu: true
  # Choose: all / ams / none, This controls whether equations are numbered and how
  tags: none
```

使用 Mathjax 前，你需要卸载 hexo 的 markdown 渲染器，然后安装 👉[hexo-renderer-kramed](https://www.npmjs.com/package/hexo-renderer-kramed)

以下操作在你 hexo 博客的目录下 (不是 Butterfly 的目录):

1.安装插件

```shell
npm uninstall hexo-renderer-marked --save
npm install hexo-renderer-kramed --save
```

2.配置 hexo 根目录的配置文件 `_config.yml`

```yml
kramed:
 gfm: true
 pedantic: false
 sanitize: false
 tables: true
 breaks: true
 smartLists: true
 smartypants: true
```

<br>

### 2.5.4 评论系统

1. 在 LeanCloud 官网（👉https://console.leancloud.cn/）免费注册一个账号，然后实名认证一下
2. 创建应用
   a. 点击左上角 `LeanCloud` 后选择 `创建应用`
   b. 填写 `应用名称`
   c. 默认选择 `开发版`
   d. 点击 `创建`

下面修改配置

PS: avatar 是 MasterKey

```yml
# Valine
# https://valine.js.org
valine:
  appId: jinkuniPUjkzrzdZhcBnFRCQSphAUZ-gzGzoHszjinkun # leancloud application app id
  appKey: jinkunOhh5rx3HNbo5pPT6D9zrvrymjinkun # leancloud application app key
  avatar: retro # gravatar style https://valine.js.org/#/avatar
  # This configuration is suitable for domestic custom domain name users, overseas version will be automatically detected (no need to manually fill in)
  serverURLs:
  bg:
  # Use Valine visitor count as the page view count
  visitor: false
  option:
```

```yml
# --------------------------------------
# Comments System
# --------------------------------------

comments:
  # Up to two comments system, the first will be shown as default
  # Leave it empty if you don't need comments
  # Choose: Disqus/Disqusjs/Livere/Gitalk/Valine/Waline/Utterances/Facebook Comments/Twikoo/Giscus/Remark42/Artalk
  # Format of two comments system : Disqus,Waline
  use: Valine
  # Display the comment name next to the button
  text: true
  # Lazyload: The comment system will be load when comment element enters the browser's viewport.
  # If you set it to true, the comment count will be invalid
  lazyload: false
  # Display comment count in post's top_img
  count: true
  # Display comment count in Home Page
  card_post_count: true
```

重新生成文章即可看到效果

```shell
npx hexo cl
npx hexo s
```

删除一条评论

![](/res/img/post/01-hexo+nginx_deploy_website/18_删除评论.png)


&nbsp;

<hr>

**2025更新**：使用 gitalk 评论系统

遵循 👉[gitalk](https://github.com/gitalk/gitalk) 的指示去获取你的 github Oauth 应用的 client id 和 secret 值。以及查看它的相关配置説明。

```yml
gitalk:
  client_id:
  client_secret:
  repo:
  owner:
  admin:
  option:
```

|参数|解释|
|-|-|
|client_id|GitHub 应用的 client ID|
|client_secret|GitHub 应用的 client secret|
|repo|存储 issues 的 repo|
|owner|存储 issues 的 repo 的拥有者|
|admin|GitHub repository 的所有者和合作者 (对这个 repository 有写权限的用户)|
|option|可选配置|

创建 `OAuth Application`

需要点击右上角头像【Settings】->【Developer settings】->【OAuth Apps】->【New OAuth App】 创建【GitHub Application】进行基本配置 ，如果没有，👉[点击这里申请](https://github.com/settings/applications/new)

Authorization callback URL 填写当前使用插件页面的域名。( 跟 Homepage URL 一样 )

示例：

```yml
# Gitalk
# https://github.com/gitalk/gitalk
gitalk:
  client_id: Ov23liyxWK7876ejoLVlsuQ
  client_secret: f0b0hgd980296f45b379786903c224536ef1f3aca
  repo: 20250119_Hexo_Gitalk
  owner: AnnihilateSword
  admin: AnnihilateSword
  option:
```

PS：repo 填仓库名就行，owner 和 admin 填你的 Github 名称

```yml
comments:
  # Up to two comments system, the first will be shown as default
  # Leave it empty if you don't need comments
  # Choose: Disqus/Disqusjs/Livere/Gitalk/Valine/Waline/Utterances/Facebook Comments/Twikoo/Giscus/Remark42/Artalk
  # Format of two comments system : Disqus,Waline
  use: Gitalk
  # Display the comment name next to the button
  text: true
  # Lazyload: The comment system will be load when comment element enters the browser's viewport.
  # If you set it to true, the comment count will be invalid
  lazyload: false
  # Display comment count in post's top_img
  count: true
  # Display comment count in Home Page
  card_post_count: true
```

这个仓库记得初始化，按照 Github 步骤来就行：

```shell
git init
git add README.md
git commit -m "first commit"
git branch -M main
git remote add origin git@github.com:AnnihilateSword/20250119_Hexo_Gitalk.git
git push -u origin main
```

&nbsp;

**反向代理**

**使用 netlify 部署服务**

👉https://proxy-gitalk-api.netlify.app/

👉https://app.netlify.com/

> **反向代理我没成功~~哈哈~**

&nbsp;

<hr>

**2025更新**：使用 Twikoo 评论系统

👉https://twikoo.js.org/
👉[Twikoo Vercel 部署教程](https://www.bilibili.com/video/BV1Fh411e7ZH)
👉[Vercel 部署](https://twikoo.js.org/backend.html#vercel-%E9%83%A8%E7%BD%B2)

`评论通知QQ邮箱：`

SMTP_PASS 获取

登录你的 qq 邮箱或 163 邮箱，点击设置，选择账号，你就可以在页面下看到 SMTP 配置项。

`POP3/IMAP/SMTP/Exchange/CardDAV/CalDAV服务`

```txt
SENDER_EMAIL：邮件通知邮箱地址（必填）。
SENDER_NAME：邮件通知标题。
SMTP_SERVICE：邮件通知邮箱服务商（必填）。
SMTP_HOST：自定义 SMTP 服务器地址（必填）, 根据你选择的邮箱决定，如 QQ 为 smtp.qq.com。
SMTP_PORT：自定义 SMTP 端口，QQ 端口可配置为 465
SMTP_SECURE：自定义 SMTP 是否使用 TLS，请填写 true 或 false（选择 true）。
SMTP_USER：邮件通知邮箱用户名。
SMTP_PASS：邮件通知邮箱密码，QQ、163 邮箱请填写授权码（见下图）。
```

常见问题：👉[收不到邮箱提醒](https://twikoo.js.org/faq.html#%E6%94%B6%E4%B8%8D%E5%88%B0%E6%8F%90%E9%86%92%E9%82%AE%E4%BB%B6)

> 这个国内不用翻墙，考虑到还是国人交流比较多吧哈哈~国风第一！

<br>

### 2.5.5 Pjax（提升加载速度）

当用户点击链接，通过 ajax 更新页面需要变化的部分，然后使用 HTML5 的 pushState 修改浏览器的 URL 地址。

这样可以不用重复加载相同的资源（css/js）， 从而提升网页的加载速度。

```yml
# Pjax
# It may contain bugs and unstable, give feedback when you find the bugs.
# https://github.com/MoOx/pjax
pjax:
  enable: true
  # Exclude the specified pages from pjax, such as '/music/'
  exclude:
    # - /xxxxxx/
    - /music/
    - /no-pjax/
```

<br>

### 2.5.6 音乐

安装插件

```shell
npm install --save hexo-tag-aplayer
```

插件会在每一个文件都插入 js 和 css，为了避免这一情况，3.0 版本内置了 aplayer 需要的 css 和 js。

首先在 Hexo 根目录 `_config.yml` 里配置 asset_inject 为 false

```yml
# music
aplayer:
  cdn: https://cdn.jsdelivr.net/npm/aplayer/dist/APlayer.min.js  # 引用 APlayer.js 外部 CDN 地址 (默认不开启)
  style_cdn: https://cdn.jsdelivr.net/npm/aplayer/dist/APlayer.min.css
  meting: true       # MetingJS 支持
  meting_cdn: https://cdn.jsdelivr.net/npm/meting/dist/Meting.min.js # 引用 Meting.js 外部 CDN 地址 (默认不开启)
```

我后面放到国内的 Gitee 上了，防止失效

```yml
# music
aplayer:
  cdn: https://gitee.com/yunmiezhijian/blog-resources/raw/master/js/APlayer.min.js  # 引用 APlayer.js 外部 CDN 地址 (默认不开启)
  style_cdn: https://gitee.com/yunmiezhijian/blog-resources/raw/master/css/APlayer.min.css
  meting: true       # MetingJS 支持
  meting_cdn: https://gitee.com/yunmiezhijian/blog-resources/raw/master/js/Meting.min.js # 引用 Meting.js 外部 CDN 地址 (默认不开启)
  enable: true  # 实现全局音乐
  asset_inject: false
```

开启 `在主题配置文件` 中的 aplayerInject

```yml
# Inject the css and script (aplayer/meting)
aplayerInject:
  enable: true # 开启音乐播放器
  per_page: true # 每个页面都有
```

然后在你需要使用 aplayer 的页面 Front-matter 添加

```markdown
aplayer: true
```

这样只会在需要 aplayer 的页面插入 js 和 css。

参考：👉https://easyhexo.com/3-Plugins-use-and-config/3-1-hexo-tag-aplayer/#%E9%85%8D%E7%BD%AE

<br>

### 2.5.7 美化 | 特效

#### 2.5.7.1 默认显示模式

修改 `主题配置文件`

```yml
# Default display mode (网站默认的显示模式)
# light (default) / dark
display_mode: dark
```

<br>

#### 2.5.7.2 自定义主题颜色

可以修改大部分 UI 颜色

修改 `主题配置文件`，比如：

> 颜色值必须被双引号包裹，就像 “#000” 而不是 #000。否则将会在构建的时候报错！

```yml
theme_color:
  enable: true
  # main: "#49B1F5"
  # main: "#f549b3"  # 紫色
  main: "rgb(80, 80, 220)"  # 蓝色
  paginator: "#00c4b6"
  # button_hover: "#FF7242"
  button_hover: "#1e3472"  # 按钮 hover
  # text_selection: "#00c4b6"
  text_selection: "#3367d1"  # 文本选中
  # link_color: "#99a9bf"
  # link_color: "#B96D00"  # 链接颜色
  link_color: "#e1a43c"  # 链接颜色
  meta_color: "#858585"
  # hr_color: "#A4D8FA"
  hr_color: "#eaeaea"  # 分割线颜色
  # code_foreground: "#F47466"
  code_foreground: "rgb(220, 80, 80)"  # `` 前景颜色
  code_background: "rgba(27, 31, 35, .05)"
  # toc_color: "#00c4b6"
  toc_color: "rgb(80, 80, 220)"  # 目录颜色
  # blockquote_padding_color: "#49b1f5"
  # blockquote_background_color: "#49b1f5"
  blockquote_padding_color: "#d66829"  # 引用块颜色
  blockquote_background_color: "#d66829"  # 引用块颜色
  scrollbar_color: "#49b1f5"
  meta_theme_color_light: "ffffff"
  meta_theme_color_dark: "#0d0d0d"
```

<br>

#### 2.5.7.3 footer 背景

修改 `主题配置文件`

```yml
# The background image of footer
footer_img: true
```

<br>

#### 2.5.7.4 打字效果

修改 `主题配置文件`

```yml
# Typewriter Effect (打字效果)
# https://github.com/disjukr/activate-power-mode
activate_power_mode:
  enable: true
  colorful: true # open particle animation (冒光特效)
  shake: false #  open shake (抖动特效)
  mobile: false
```

<br>

#### 2.5.7.5 鼠标点击效果

修改 `主题配置文件`

```yml
# Mouse click effects: Heart symbol (鼠标点击效果: 爱心)
click_heart:
  enable: true
  mobile: false
```

<br>

#### 2.5.7.6 网站副标题

修改 `主题配置文件`

可设置主页中显示的网站副标题或者喜欢的座右铭。

```yml
# the subtitle on homepage (主页subtitle)
subtitle:
  enable: true
  # Typewriter Effect (打字效果)
  effect: true
  # Customize typed.js (配置typed.js)
  # https://github.com/mattboldt/typed.js/#customization
  typed_option:
  # source 调用第三方服务
  # source: false 关闭调用
  # source: 1  调用一言网的一句话（简体） https://hitokoto.cn/
  # source: 2  调用一句网（简体） https://yijuzhan.com/
  # source: 3  调用今日诗词（简体） https://www.jinrishici.com/
  # subtitle 会先显示 source , 再显示 sub 的内容
  source: false
  # 如果关闭打字效果，subtitle 只会显示 sub 的第一行文字
  sub:
    - 不要轻言放弃
    - Don't give up.
```

<br>

#### 2.5.7.7 加载动画

当进入网页时，因为加载速度的问题，可能会导致 top_img 图片出现断层显示，或者网页加载不全而出现等待时间，开启 preloader 后，会显示加载动画，等页面加载完，加载动画会消失。

主题支持 pace.js 的加载动画，具体可查看 👉[pace.js](https://codebyzach.github.io/pace/)

修改 `主题配置文件`

```yml
# Loading Animation (加載動畫)
preloader:
  enable: true
  # source
  # 1. fullpage-loading
  # 2. pace (progress bar)
  source: 1
  # pace theme (see https://codebyzach.github.io/pace/)
  pace_css_url:
```

PS：这里我就用默认盒子加载动画了

<br>

#### 2.5.7.8 字数统计

要为 Butterfly 配上字数统计特性，你需要如下几个步骤：

打开 hexo 工作目录

```shell
npm install hexo-wordcount --save
```

修改 `主题配置文件`：

```yml
# wordcount (字数统计)
# see https://butterfly.js.org/posts/ceeb73f/#字数统计
# Need to install the hexo-wordcount plugin
wordcount:
  enable: true
  # Display the word count of the article in post meta
  post_wordcount: true
  # Display the time to read the article in post meta
  min2read: true
  # Display the total word count of the website in aside's webinfo
  total_wordcount: true
```

<br>

#### 2.5.7.9 nstantpage

当鼠标悬停到链接上超过 65 毫秒时，Instantpage 会对该链接进行预加载，可以提升访问速度。

修改 `主题配置文件`:

```yml
# https://instant.page/
# prefetch (预加载)
instantpage: true
```

<br>

#### 2.5.7.10 Pangu

如果你跟我一样，每次看到网页上的中文字和英文、数字、符号挤在一块，就会坐立难安，忍不住想在它们之间加个空格。这个外挂正是你在网路世界走跳所需要的东西，它会自动替你在网页中所有的中文字和半形的英文、数字、符号之间插入空白。

修改 `主题配置文件`:

```yml
# https://github.com/vinta/pangu.js
# Insert a space between Chinese character and English character (中英文之间添加空格)
pangu:
  enable: true
  field: site # site/post
```

<br>

#### 2.5.7.11 CSS 前缀

有些 CSS 并不是所有浏览器都支持，需要增加对应的前缀才会生效。

开启 css_prefix 后，会自动为一些 CSS 增加前缀。（会增加 20% 的体积）

修改 `主题配置文件`:

```yml
# Add the vendor prefixes to ensure compatibility
css_prefix: true
```

<br>

### 2.5.8 主题页面添加

#### 2.5.8.1 标签页

1. 前往你的 Hexo 博客的根目录
2. 输入 `npx hexo new page tags`
3. 你会找到 `source/tags/index.md` 这个文件
4. 修改这个文件：
   记得添加 `type: "tags"`

```markdown
---
title: Tags
date: 2024-02-20 01:23:14
type: "tags"
---
```

> Tips: 添加一个标签再删除需要 npx hexo clean 清理数据库才能生效

<br>

#### 2.5.8.2 分类页

1. 前往你的 Hexo 博客的根目录
2. 输入 `npx hexo new page categories`
3. 你会找到 `source/categories/index.md` 这个文件
4. 修改这个文件：
   记得添加 `type: "categories"`

```markdown
---
title: Categories
date: 2024-02-20 01:23:14
type: "categories"
---
```

<br>

#### 2.5.8.3 友链

为你的博客创建一个友情链接！

- 创建友链页面

1. 前往你的 Hexo 博客的根目录
2. 输入 `npx hexo new page link`
3. 你会找到 `source/link/index.md` 这个文件
4. 修改这个文件：
   记得添加 `type: "link"`

```yml
---
title: Link
date: 2024-02-20 01:23:14
type: "link"
---
```

`友链添加：`

在 Hexo 博客目录中的 `source/_data`（如果没有 _data 文件夹，请自行创建），创建一个文件 `link.yml`

```yml
- class_name: 友情链接
  class_desc: 好友
  link_list:
    - name: 都哥
      link: https://ahucd.cn/
      avatar: https://ahucd.cn/img/avatar.png
      descr: 天剑行风

- class_name: 网站
  class_desc: 值得推荐的网站
  link_list:
    - name: Youtube
      link: https://www.youtube.com/
      avatar: https://i.loli.net/2020/05/14/9ZkGg8v3azHJfM1.png
      descr: 视频网站
    - name: Bilibili
      link: https://www.bilibili.com/
      avatar: /res/img/site/bilibili.svg
      descr: 视频网站
```

class_name 和 class_desc 支持 html 格式书写，如不需要，也可以留空。

PS：主题支持友情链接随机排序，只需要在顶部 front-matter 添加 random: true

<br>

#### 2.5.8.4 子页面

子页面也是普通的页面，你只需要 `hexo n page xxxxx` 创建你的页面就行

例如创建 About 页面或 music 页面

```shell
npx hexo n page about
```

```shell
npx hexo n page music
```

<br>

### 2.5.9 其他功能支持

#### 2.5.9.1 文章打赏/赞助

在你每篇文章的结尾，可以添加贊助按钮。相关二维码可以自行配置。

对于没有提供二维码的，可配置一张软件的 icon 图片，然后在 link 上添加相应的贊助链接。用户点击图片就会跳转到链接去。

link 可以不写，会默认为图片的链接。

```yml
# Sponsor/reward
reward:
  enable: true
  text:
  QR_code:
    - img: /res/img/site/wx_qr_code.png
      link:
      text: wechat
    - img: /res/img/site/zfb_qr_code.jpg
      link:
      text: alipay
```

<br>

#### 2.5.9.1 相关文章

PS：当文章封面设置为 false 时，或者没有获取到封面配置，相关文章背景将会显示主题色。

相关文章推荐的原理是根据文章 tags 的比重来推荐

```yml
related_post:
  enable: true
  limit: 6 # 显示推荐文章数目
  date_type: created # or created or updated 文章日期显示创建日或者更新日
```

<br>

## 2.6 文章常用设置

```markdown
---
title: UE5 C++ 加载 Json 文件
tags: [UE5, C++]
date: 2023-12-12
updated: 2024-12-12
cover: /res/img/post/UE5 C++ 加载 Json 文件/1.png
top_img: /res/img/site/samurai.png
---
```

参考：👉https://butterfly.js.org/posts/dc584b87/

- title: 标题
- tags: 标签
- date：页面创建日期
- updated：页面更新日期
- cover: 设置封面
- top_img：设置顶部图片
- description：页面描述

<br>
<br>

# 三、部署

为了方便维护，我个人这里使用的是 Windows + Nginx 的方式在云服务器上部署

## 3.1 安装依赖

在云服务器上我们也需要安装 Git 和 Nodejs

git: 👉https://git-scm.com/
nodejs: 👉https://nodejs.org/en

<br>

## 3.2 网站项目上传 Git

这里我上传 Github

> 在上传前，建议先备份一下项目，避免损失

在真正上传前，我还要做一些操作

1. 删除主题 butterfly 文件夹中的 `.git` 文件夹，我打算手动更新，避免云服务器上 clone 时有问题
2. 将项目文件夹下的 `.gitignore` 文件删除，我打算全部上传，维护整个项目
3. 生成静态文件
   `npx hexo g`

做完这些操作后，就可以新建一个仓库，将整个项目上传上去了

![](/res/img/post/01-hexo+nginx_deploy_website/2.png)

```shell
git add .
git commit -m "<feat>: init"
git push
```

<br>

## 3.3 云服务器上应用项目

```shell
git config --global user.name xxx
git config --global user.email xxx
git clone xxx
```

<br>

## 3.4 配置 Nginx

打开 `nginx.conf` 在 http 下添加

```txt
server {
    listen 80;
    server_name 域名/IP;

    location / {
        root xxx/public;
        index index.html index.htm;
    }
}
```

PS：这里 `root xxx/public;` 是设置你的项目目录

- 验证配置是否 ok

```shell
./nginx.exe -t -c ./conf/nginx.conf
```

- 重启 nginx

```shell
./nginx.exe -s reload
```

> Tip：有时候 404 是端口被占用引起的，可以重启一下

有些服务器需要手动开启端口防火墙，以我这里为例

![](/res/img/post/01-hexo+nginx_deploy_website/3.png)

![](/res/img/post/01-hexo+nginx_deploy_website/4.png)

![](/res/img/post/01-hexo+nginx_deploy_website/5.png)

![](/res/img/post/01-hexo+nginx_deploy_website/6.png)

![](/res/img/post/01-hexo+nginx_deploy_website/7.png)

通过公网 IP 访问进行测试

![](/res/img/post/01-hexo+nginx_deploy_website/8.png)

<br>
<br>

# 四、域名配置

这里以腾讯云为例

- 需要先购买一个域名

![](/res/img/post/01-hexo+nginx_deploy_website/9.png)

![](/res/img/post/01-hexo+nginx_deploy_website/10.png)

![](/res/img/post/01-hexo+nginx_deploy_website/11.png)

- 刚购买好需要等待审核，这里以我之前买的域名为例，添加解析

![](/res/img/post/01-hexo+nginx_deploy_website/12.png)

![](/res/img/post/01-hexo+nginx_deploy_website/13.png)

![](/res/img/post/01-hexo+nginx_deploy_website/14.png)

![](/res/img/post/01-hexo+nginx_deploy_website/15.png)

![](/res/img/post/01-hexo+nginx_deploy_website/16.png)

<br>
<br>

# 五、SSL证书

购买证书服务，验证通过后一般会给u这两个格式的文件

![](/res/img/post/01-hexo+nginx_deploy_website/19.png)

![](/res/img/post/01-hexo+nginx_deploy_website/20.png)

补充：

![](/res/img/post/01-hexo+nginx_deploy_website/21.png)

最后还需要打开 `nginx.conf` 在 # HTTPS 下的 location 添加网页目录位置

```txt
server {
    listen 80;
    server_name 域名/IP;

    location / {
        root xxx/public;
        index index.html index.htm;
    }
}
```

- 最终效果

![](/res/img/post/01-hexo+nginx_deploy_website/17.png)

<br>
<br>

# 补充

新版本图片点击没有放大效果，需要更改 `主题配置文件`：
👉https://github.com/jerryc127/hexo-theme-butterfly/issues/1601

```yml
# Choose: fancybox / medium_zoom
# https://github.com/francoischalifour/medium-zoom
# https://fancyapps.com/fancybox/
# Leave it empty if you don't need lightbox
lightbox: fancybox
```

<br>

## 自定义样式

这里我在 `themes\butterfly\source\css` 目录下添加了一个 `custom.css` 文件例如：

```css
body,
.gitcalendar {
    /* font-family: tzy !important; */
    font-family: 'Microsoft Yahei' !important;
}
```

然后需要在 Inject 插入如下链接：

```yml
# Inject
# Insert the code to head (before '</head>' tag) and the bottom (before '</body>' tag)
inject:
  head:
    # - <link rel="stylesheet" href="/xxx.css">
    - <link rel="stylesheet" href="/css/custom.css" media="defer" onload="this.media='all'">
```

其中 `media="defer"` `onload="this.media='all'"` 是异步加载配置项，确保自定义样式会在页面加载完成后才继续渲染。如果没有需求或效果不好可以不加这个。

<br>

## Butterfly 添加全局吸底 Aplayer

👉https://butterfly.js.org/posts/507c070f/

<br>

## Hexo哔哩哔哩番剧页面插件

👉https://imszz.com/p/8422e92e/

<br>

## 图标库

👉https://fontawesome.com/icons?d=gallery

<br>

## Live2D 看板娘

👉https://github.com/stevenjoezhang/live2d-widget

&nbsp;
&nbsp;

The end.
