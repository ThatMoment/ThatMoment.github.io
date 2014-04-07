---
layout: post
title: "使用Octopress搭建GitHub Pages"
date: 2014-04-04 15:32:26 +0800
comments: true
categories: 技术
---

###搭建Ruby发布环境

1.可以使用`ruby --version`查看本机`Ruby`版本号。我是用的**OS X 10.9.2**系统，自带2.0.0版本的`Ruby`。

2.如果本机没有安装，可以使用`RVM(Ruby Version Manager)`来负责安装和管理`Ruby`的环境。 
		 	
安装`RVM`
	
```
curl -L https://get.rvm.io | bash -s stable --ruby
```
安装`Ruby`
		
```
rvm install 2.0.0
rvm use 2.0.0
rvm rubygems latest
```
  参考[Installing Ruby With RVM](http://octopress.org/docs/setup/rvm/)

###安装Octopress

1.首先把**Octopress**从**GitHub**上下载下来

``` 
git clone git://github.com/imathis/octopress.git octopress
cd octopress
```

2.安装各安装包的依赖关系	

```
gem install bundle
bundle install
```
	
在进行`bundle install`时出现`An error occured while installing RedCloth (4.2.9), and Bundler cannot continue.
	Make sure that gem install RedCloth -v '4.2.9' succeeds before bundling.`错误。

最后通过强制重新安装`Ruby`解决了这个问题。
	
```
rvm --force install 2.0.0
gem install bundle --no-ri --no-rdoc
bundle install
```
参考[Stackoverflow](http://stackoverflow.com/a/12161114/2436229)

3.安装**Octopress**主题样式

可以通过`rake install`安装默认主题样式	
	
这里安装的是一个**GitHub**上面的一个[开源样式](https://github.com/shashankmehta/greyshade)
	
```	
$ git clone git@github.com:shashankmehta/greyshade.git .themes/greyshade
$ echo "\$greyshade: color;" >> sass/custom/_colors.scss //Substitue 'color' with your highlight color
$ rake "install[greyshade]"
$ rake generate
```

###配置Octopress

1.进入**Octopress**目录打开`_config.yml`进行修改。
具体可以参考[Configuring Octopress](http://octopress.org/docs/configuring/)。
2.配置完成之后可以生成静态网页并通过[http://127.0.0.1:4000]()预览

```
rake generate
rake preview
```
	


