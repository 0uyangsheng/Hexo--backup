---
title: Build github pages with Hexo+NexT
date: 2018-04-17 16:45:50
tags:
- web
categories:
- web
---
{% asset_img hexo-github.png %}

主要记录在Windows下搭建博客系统的一些简要步骤，具体参考相关链接。
> *PS：为了建这个博客，Search了网络上的一些信息，很多都是只言片语且不够全面，甚至过时。所以最好的做法是参考原始出处、原始文档，一是原始的更全面，二是会时时更新。*

### 系统环境配置（Windows）
主要需要以下三个软件，按顺序安装(点击进官网查看相关)：
- [git-scm](http://git-scm.com/download/)
- [Nodejs](https://nodejs.org/en/)
- [Hexo](https://hexo.io)

### Hexo相关

- 安装Hexo

{% codeblock %}
$ cd d:/hexo
$ npm install hexo-cli -g
$ hexo init blog
$ cd blog
$ npm install
$ hexo g # 或者hexo generate
$ hexo s # 或者hexo server，可以在http://localhost:4000/ 查看
{% endcodeblock %}

- [Hexo常用命令及用法-点击进官网](https://hexo.io/docs/)

    - `hexo generate (hexo g)` 生成静态文件，会在当前目录下生成一个新的叫做public的文件夹
    - `hexo server (hexo s)` 启动本地web服务，用于博客的预览
    - `hexo deploy (hexo d)` 部署播客到远端（比如github, heroku等平台）
    - `hexo new "postName"` #新建文章
    - `hexo new page "pageName"` #新建页面
    - `hexo clean`  清除静态资源

### [Next主题](http://theme-next.iissnan.com/)
- clone [NexT主题](https://github.com/iissnan/hexo-theme-next)

 > git clone https://github.com/iissnan/hexo-theme-next

其中有个问题，如何将站点及主题的配置整合到一起，并合理的保存，参考NexT的 README中描述。
- 相关配置（大多在主题_config.yaml中可配置）
    - 添加categories、tags（参考官方github README）
    - 站点访问次数
        - 不蒜子 http://busuanzi.ibruce.info/ 
        - busuanzi_count: true
    - 站内搜索
        - algolia_search
        - local_search
    - 文章访问次数
        - leancloud_visitors
    - 评论添加（有多种配置，参考_config.yaml）
        - youyan_uid （http://www.uyan.cc）


### 项目托管到github
- 创建github pages
    - 在github上新建仓库，且仓库的名字必须是username/username.github.io
- 部署
    - 配置ssh-key:`ssh-keygen -t rsa -C "注册git的邮箱"`；
    - 打开https://github.com/settings/ssh ->new SSH key 添加密钥，title随便写，key为id_rsa.pub中所有内容；
    - 验证是否能否连接到github: `ssh -T git@github.com`
    - 同步到github
        - 首先安装：`npm install hexo-deployer-git --save`
        - 在blog目录执行： `hexo d`

按以上配置，基本上一个github pages基本完成，接下来要考虑如何编辑。

### 编辑相关
- Markdown 编辑器选择
    - Visual studio Code
    - [HexoEditor](https://github.com/zhuzhuyule/HexoEditor/releases)

- 如何插入图片（[Link](https://hexo.io/docs/asset-folders.html)）
    - post_asset_folder: true
    - asset_img

- 如何插入代码（[Link](https://hexo.io/docs/tag-plugins.html)）
    - 采用 codeblock
    - 3个`

{% note default %} 

\`\`\`[language] [title] [url] [link-text]

`代码`

\`\`\`

{% endnote %}

[language] 是代码语言的名称，用来设置代码块颜色高亮，非必须；

[title] 是顶部左边的说明，非必须；

[url] 是顶部右边的超链接地址，非必须；

[link text] 如它的字面意思，超链接的名称，非必须。

亲测这 4 项应该是根据空格来分隔，而不是[]，故请不要加[]。除非如果你想写后面两个，但不想写前面两个，那么就必须加[]了，要这样写：`[] [] [url] [link text]`。

- 如何添加categories&tags（[Link](https://hexo.io/docs/front-matter.html)）

### 总结
为什么选择github pages+hexo方案，最大的原因是免费，且有清新简约的Hexo引擎加上各种主题，然后可以在github上面保存Tracking。

在实施的过程中，最好是通读原始的文档，然后多实际尝试，一步一步会达到想要的结果。
