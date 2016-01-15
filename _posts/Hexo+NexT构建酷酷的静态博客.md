title: Hexo+NexT构建酷酷的静态博客
date: 2015-12-28 23:02:07
categories: hexo
tags:
- hexo
- next
- github
- 静态博客

---

最近在网上浏览文章的时候，无意看到某个作者的博客简洁、清晰。顺着网站看到底部。`由Hexo强力驱动`和`主题 NexT`暴露了网站的技术，NexT带着链接，点进去看了下，进入了github的页面，文档写的很清晰，而且有作者自己博客地址，看起来真的很不错。加上NexT文档说的`5分钟快速安装`，于是心动了，准备捣鼓捣鼓。

之前也捣鼓过半天，想用github来写博客，当时网上搜索了下，Jekyll是那时的主流，搜索引擎搜出来的基本上和他相关的文章。于是用Jekyll搭了个博客。可是，我却始终对博客的主题并没有很满意，样式看起来太大了，也不知道怎么修改。然后如何让搜索引擎发现自己的文章，能在网上分享出来也一无所知，大半年过去了，搜索引擎直接搜索博客的标题，也无法找到。

直到看到NexT，我觉得有希望了。花了些碎片时间搭建了个博客。写下此文记录下整个搭建过程。

NexT是Hexo的一个主题，要学会用NexT，其实主要是要会用Hexo。

Hexo是一个简单、快速、强大的Node.js静态博客框架，用起来非常简单，只有简单的几个命令，就能安装各种组件，集成各种强大的功能。

安装Hexo前还需安装npm，一个Node.js的包管理。好在在若干年前，我的npm已经装好了。在mac下安装软件也非常简单，homebrew是mac下强大的软件管理工具，只需要一条`brew install node`即可。

npm安装好后，安装Hexo：

	npm install hexo -g

可以使用该命令升级hexo到最新版本：

	npm update hexo -g

先初始化博客目录

	hexo init <folder>
	
想要写篇新文章，可以执行下面命令，生成一个文件：

	hexo new "Hello World"
	
生成网站用如下命令：

	hexo generate

启动网站服务用如下命令：
	
	hexo server
	
这样默认就可以访问如下网址了：

	http://localhost:4000/	
	
这中间可能有各种插件需要安装，列出我装好的插件：

	hexo
	hexo-generator-baidu-sitemap
	hexo-generator-index
	hexo-renderer-ejs
	hexo-server
	hexo-deployer-git
	hexo-generator-category
	hexo-generator-sitemap
	hexo-renderer-marked
	hexo-generator-archive
	hexo-generator-feed
	hexo-generator-tag
	hexo-renderer-stylus
	
 安装插件使用类似下面的命令即可：
 
 	npm install hexo-deployer-git --save
 	
 说下我知道几个插件，`hexo-deployer-git`用来部署git，`hexo-generator-category`用来生成分类，`hexo-generator-sitemap`用来生成谷歌的网站地图，可以让搜索引擎发现自己的网站。`hexo-generator-baidu-sitemap`是百度的网站地图。`hexo-generator-feed`用来生成rss种子。`hexo-generator-tag`用来生成标签。`hexo-generator-archive`用来生成归档。关于这些插件和sitemap如何提交给搜索引擎，我从这篇文章参考了很多：[《Hexo搭建GitHub博客（三）- NexT主题配置使用》](http://zhiho.github.io/2015/09/29/hexo-next/)
 
 Hexo的目录结构可以参考这篇文章[hexo的目录结构及作用](http://www.tuicool.com/articles/fiYVbaY)，主结构如下：

 	|-- _config.yml
	|-- package.json
	|-- scaffolds
	|-- scripts
	|-- source
	   |-- _drafts
	   |-- _posts
	|-- themes
	
文章放在	source/_posts目录下，用`hexo new "Hello World"`生成的文件就放在该目录。NexT主题就放在themes下。
 
 安装好Hexo后，把从git下clone或者download下来的NexT文件夹，重命名为next，放到博客文件夹的theme目录下，然后修改博客_config.yml配置文件的theme为next，就使用上最基本的Hexo+NexT了。
 
Hexo+NexT强大的除了安装、部署简单，还提供了很多强大的第三方功能。

下面的这些配置在Hexo的配置文件中配置：

	plugins:
	- hexo-generator-sitemap
	- hexo-generator-baidu-sitemap
	
该配置可以配置google和baidu的sitemap，百度生成sitemap还需要使用下面一行配置：

	baidusitemap:
    	path: baidusitemap.xml
    	
接着要配置两个网站的标识：

	google_site_verification: 
	baidu_site_verification: 
    	
然后可以配置谷歌分析：

	google_analytics:
	
配置多说评论：

	duoshuo_shortname:
	
配置网站搜索引擎：

	swiftype_key:
	
配置社交网络：

	social:
  		github:
  		weibo:

配置分析服务:
	
	jiathis:
	
配置知识分析图标：

	creative_commons:

配置站点起始时间：

	since: 2015

配置部署地址，这个可以把网站自动部署到github或者gitcafe：

	deploy:
	  type: git
	  repository: 
	  branch: 

这里有个这个配置需要注意，因为我是从Jekyll迁移过来的，所以把默认的文件命令格式给改成了如下配置：

	# Writing
	new_post_name: :year-:month-:day-:title.md # File name of new posts
	
配置完了Hexo的配置后，接下来可以配置NexT的配置，NexT的配置名也为_config.xml，在NexT目录下。

menu配置可以定义显示那些目录：

	menu:
	  home: /
	  categories: /categories
	  about: /about
	  archives: /archives
	  tags: /tags
	  #commonweal: /404.html

menu_icons可以定义每个目录名称的icon。

favicon可以定义网站的ico。

keywords定义网站的关键字。

rss:定义rss功能，正常只要这样配置即可，不需要rss功能，使用false即可关闭。

highlight_theme定义代码的颜色，默认使用normal，更推荐使用night，可以很明显的区分代码块。

scheme默认是注释的，这里基本上大家都是用Mist。

sidebar据说是可以打开和隐藏边栏，可是我改成任何值都是隐藏的效果。。。

toc可以开启目录功能。

auto_excerpt表示在首页显示的文章是否压缩，即只显示部分内容。

可以在这个配置文件里面设置头像：

	avatar:
	
然后设置多说热评文章：
	
	duoshuo_hotartical:
	
上面就是我的Hexo+NexT的配置了，我只是初次体验，还有很多第三方功能可以增加的。


