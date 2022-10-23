---
title: hexo踩坑记
date: 2018-08-10 22:12:44
tags: other
categories: 挖坑
---

>`hexo`的搭建和使用十分的方便，搭配`github page`食用更佳。`github`上有很多绚丽的`hexo`主题，大家可以按需搭配。虽说`hexo`+`github page`搭建个人博客很方便快捷，但如果你需要较多的个性化定制，坑总是会踩的。本文主要遇到的各种坑以及记录简单的操作步骤，如果需要详细的安装部署教程请<a href="https://www.baidu.com/s?ie=UTF-8&wd=hexo" target="_self">点击传送门离开</a>。

### 安装`hexo`
>如果你不知道或者未安装 npm ，<a href="https://www.npmjs.com/" target="_blank">点击此处</a>了解以及下载。

```bash
npm i -g hexo-cli
```

说到`npm`我在后面挖到了一个坑，这个坑以前已经挖过，这里又遇到了，有必要记下来。如果你跟我一样是在使用`Windows 10 + VirtualBox (VBox) + Vagrant + Laravel Homestead + 共享目录`这样的环境做开发的，需要注意。

在上述环境中使用`npm`会有比较多意料之外的状况，大部分`stackoverflow`上都有解决方案，但这次遇到的`error “ETXTBSY: text file is busy” on npm install`弄了一段时间仍然没解决，把`npm`搞坏后还是换到`windows`环境去了。

虽然没解决，但还是推荐一下比较靠谱的<a href="https://stackoverflow.com/questions/45678817/error-etxtbsy-text-file-is-busy-on-npm-install" target="_blank">解决方案</a>。

### 快速搭建
```bash
hexo init [folder] //初始化，新建一个网站。如果没有设置 folder ，Hexo 默认在目前的文件夹建立网站
cd folder //进入网站根目录，如果没有设置 folder ，则不需要该步骤
hexo g //generetor的缩写，生成静态文件。
hexo s //server的缩写，启动服务器。默认情况下，访问网址为： http://localhost:4000/。
```

### 主题
`hexo`有很多主题可以提供选择，而我这里则使用了<a href="https://github.com/viosey/hexo-theme-material">material</a>这个主题，最坑的是他的文档网站ssl证书居然过期了，想看都看不了=。=那就没办法啦，咱一起逐个配置试一下，再问一问`Google`，应该也不会有啥大问题。

首先我们把`material`的项目`clone`下来放到`hexo`目录下的`themes`中，根据说明为了避免配置文件被覆盖，我们需要复制一份`_config.template.yml`并命名为`_config.yml`。接下来我们的就开始定制我们自己的主题风格。`_config.yml`中大多数的配置项都有简单的说明，配置完各种基本信息后，我们想要配置一些个性化的东西，例如代码的高亮样式、选择自己喜欢熟悉的评论系统等等。

#### 高亮
首先说说配置代码高亮样式，在备注`Code highlight`下有两个配置项

```
# Code highlight
# You can only enable one of them to avoid issues.
# Also you need to disable highlight option in hexo's _config.yml.
#
#    Prettify
#        theme: # Available value in /source/css/prettify/[theme].min.css
prettify:
    enable: false
    theme: "github-v2"

#    Hanabi (https://github.com/egoist/hanabi)
#        line_number: [true/false] # Show line number for code block
#        includeDefaultColors: [true/false] # Use default hanabi colors
#        customColors: This value accept a string or am array to setting for hanabi colors.
#                    - If `includeDefaultColors` is true, this will append colors to the color pool
#                    - If `includeDefaultColors` is false, this will instead default color pool
hanabi:
    enable: true
    line_number: true
    includeDefaultColors: true
    customColors:
```

上面是各种高亮的样式选择，而下面则是`hanabi`，一个很骚的高亮风格（没错就是我用着的这个）。这里需要注意的是这两个配置项你只能设置其中一个enable为`true`，而且`hexo`的`_config.yml`中有一个配置项跟它是冲突的

```
# Writing
new_post_name: :title.md # File name of new posts
default_layout: post
titlecase: false # Transform title into titlecase
external_link: true # Open external links in new tab
filename_case: 0
render_drafts: false
post_asset_folder: false
relative_link: false
future: true
highlight:
  enable: false
  line_number: false
  auto_detect: false
  tab_replace:
```
这里的`highlight`设置为`true`会让`hanabi`不起作用。如果你选用`prettify`的话，可以在路径`material/source/css/prettify/[theme].min.css`找喜欢的高亮样式，其中`[theme]`就是`prettify.theme`中填写的样式名称。

#### 评论系统

大家可以看到，集成服务里面，包含了评论系统的配置项，如此方便的配置就能够加入一个评论系统到博客里了，何乐而不为呢？

```
# Comment Systems
# Available value of "use":
#     disqus | disqus_click | changyan | livere | gitment | gitalk | valine | wildfire
# If you want to use gitment or gitalk,you should get the client_id and client_secret from https://github.com/settings/applications/new
# If you want to use valine,you should get the app_id and app_key from https://leancloud.cn ,more setting please see https://valine.js.org
comment:
    use: gitalk
    shortname: # duoshuo or disqus shortname
    changyan_appid:
    changyan_conf:
    changyan_thread_key_type: path
    livere_data_uid:
    gitment_repo:   # git repo of the hexo
    gitment_owner:  # git repo's owner
    gitment_client_id:  # github app client id
    gitment_client_secret :  # github app client secret
    valine_leancloud_appId:  # leancloud application app id
    valine_leancloud_appKey:  # leancloud application app key
    valine_notify: false # valine mail notify (true/false) https://github.com/xCss/Valine/wiki
    valine_verify: false # valine verify code (true/false)
    valine_pageSize: 10 # comment list page size
    valine_avatar: identicon # gravatar style https://valine.js.org/#/avatar
    valine_lang: zh-CN # i18n
    valine_placeholder: Just go go # valine comment input placeholder(like: Please leave your footprints )
    valine_guest_info: nick,mail,link #valine comment header info
    gitalk_repo: gitTalk
    gitalk_owner: tomhaoye
    gitalk_client_id: ksjhdfkhfksdhfjsdhf
    gitalk_client_secret: jd12809j890sjhdosihdoaishd9o21d9081h2yd0
    wildfire_database_provider: firebase # firebase or wilddog
    wildfire_wilddog_site_id:
    wildfire_firebase_api_key:
    wildfire_firebase_auth_domain:
    wildfire_firebase_database_url:
    wildfire_firebase_project_id:
    wildfire_firebase_storage_bucket:
    wildfire_firebase_messaging_sender_id:
    wildfire_theme: light # light or dark
    wildfire_locale: zh-CN # en or zh-CN
```

大家可以看到，集成的评论系统有：`disqus`，`disqus_click`，`changyan`，`livere`，`gitment`，`gitalk`，`valine`，`wildfire`，大家喜欢哪个就就配置对应的配置项就可以了。例如我使用的`gitalk`，只需要在个人设置中选择开发者设置，然后选择`Oauth Apps`，新建一个授权应用，将得到的`Client ID`和`Client Secret`填到上面的`gitalk_`前缀配置项中，`gitalk_repo`填写你的应用名称，`gitalk_owner`填写你的账号名即可。

当然配置不是重点，这篇文章主要是记录踩过的坑，没趴下过的地方我也不会特别提出来。我完完整整的配置好了我的的`gitalk`评论系统后，先新建了一片文章并打开链接，评论系统自动请求并在`github`的仓库里面新增了一个`issue`，据我猜测这个`issue`就是用来记录当前文章的所有评论的。果不其然，我在发表了一条评论，`issue`里面便新增了留言一段记录。嗯，一切都十分的顺利，于是我又新建了一篇名字比较长的文章并打开链接，咦？怎么请求发生了错误？后来在`Google`查了半天，终于发现`gitalk`的初始化`id`参数最长只能传50个字符，而它默认是直接拿当前的`url`作为参数的，所以请求会返回客户端错误。知道了问题所在解决就比较简单了，只需要在`theme/material/layout/_widget/comment/gitalk/main.ejs`中加入：
```
<script>
    var gitalk = new Gitalk({
            clientID: '<%= theme.comment.gitalk_client_id %>',
            clientSecret: '<%= theme.comment.gitalk_client_secret %>',
            repo: '<%= theme.comment.gitalk_repo %>',
            owner: '<%= theme.comment.gitalk_owner %>',
            admin: ['<%= theme.comment.gitalk_owner %>'],
            id: document.title.substr(0,50),
            // facebook-like distraction free mode
            distractionFreeMode: false
        })
   gitalk.render('gitalk-container')
</script>
```

这里我是将`title`截取最多50个字符，大家也可以结合自身情况传入自己需要的参数。


### Emoji表情渲染

`Hexo`默认的`markdown`渲染引擎不支持`emoji`表情的渲染，为了美化博客，我们可以更换一个支持`emoji`的渲染引擎，并添加一个`emoji`插件。方法很简单，只需要键入以下三行命令：

```
npm un hexo-renderer-marked --save
npm i hexo-renderer-markdown-it --save
npm install markdown-it-emoji --save
```

全部安装成功后在`Hexo`的`_config.yml`配置文件中加入以下的配置：

```
## markdown 渲染引擎配置，默认是hexo-renderer-marked，这个插件渲染速度更快，且有新特性
markdown:
  render:
    html: true
    xhtmlOut: false
    breaks: true
    linkify: true
    typographer: true
    quotes: '“”‘’'
  plugins:
    - markdown-it-footnote
    - markdown-it-sup
    - markdown-it-sub
    - markdown-it-abbr
    - markdown-it-emoji
  anchors:
    level: 2
    collisionSuffix: 'v'
    permalink: true
    permalinkClass: header-anchor
    permalinkSymbol: ''
```

在<a href="https://www.webpagefx.com/tools/emoji-cheat-sheet/">emojy-cheet-sheet</a>中找到你想要的表情编码，例如笑脸对应的`emoji`编码是`:smile:`，清除旧文件并重新构建后在你的文章中就可以会出现:smile:了。



### 没有结束

由于博客还在搭建美化中，该文章会持续更新。。。
