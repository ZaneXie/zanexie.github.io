---
title: Github Pages & hexo 的安装与使用
date: 2017-01-24 16:09:03
tags:
---

Github Pages 提供了免费的静态文件服务，用来搭建简单的个人网站很是方便。下面介绍一下如何使用Github Pages制作个人博客。

由于Github Pages只能提供静态文件服务，因此不能使用类似wordpress的博客系统。一番搜索之后找到了[hexo](https://github.com/hexojs/hexo)，一个使用nodejs来生成静态文件的博客系统。

## Github Pages
在github上创建一个新的repository，名字叫username.github.io，比如我的是[https://github.com/zanexie/zanexie.github.io](https://github.com/zanexie/zanexie.github.io)。接下来clone这个repository

```
git clone https://github.com/username/username.github.io
```

创建一个简单的hello world页面。
```
cd username.github.io

echo "Hello World" > index.html
```

将修改push到github

```
git add --all

git commit -m "Initial commit"

git push -u origin master
```

然后打开浏览器，浏览[https://username.github.io](https://username.github.io)，一切正常将能看到hello world的页面。


## hexo
Github Pages测试正常之后，开始安装hexo。

#### 安装Node.js
hexo依赖[Node.js](https://nodejs.org/)，先到官网下载安装。windows系统在[官网](https://nodejs.org)下载安装包安装，mac和linux可以下载安装包，可以使用包管理器安装，具体的方法可以搜索一下。

安装好Nodejs后，在命令行执行

```
npm --version
```
能显示版本号表示安装成功，如果提示npm not found，则是安装失败了（或者npm不在path里），重试直到npm安装成功。

#### 安装hexo

```
npm install hexo-cli -g
```

国内的同学如果上面命令执行慢，先换国内的源，再执行
```
npm config set registry https://registry.npm.taobao.org
npm install hexo-cli -g
```
#### 创建一个blog

在刚才创建的username.github.io目录里
```
hexo init src
cd src
```
开启hexo服务
```
hexo server
```
新开一个终端，创建文章
```
cd path/to/username.github.io （修改为自己的路径）
hexo new "hello world!
```
到src/sources/_posts里找到hello world.md，写点文字。打开浏览器访问[http://localhost:4000](http://localhost:4000)（默认端口4000，执行hexo server时会有提示）。这时就能看到刚才创建的hello world了！

#### 修改配置

hexo配置文件在src/_config.yml，title、subtitle、description，url等根据自己的情况修改。public_dir默认是public，修改成../，接下来生成网站！

```
hexo generate 
```

成功之后，会在username.github.io目录里生成网站的静态文件，

```
cd ..
git add --all
git commit -m "first blog"
git push
```

Github Pages有缓存，稍等片刻，再访问[https://username.github.io](https://username.github.io)便能看到新的文章了。

#### 一些说明

##### 分离源码与静态页面
上面的步骤里，把文章源码(/src)和静态网页(/)放到了一起，这样可以使用一个repository同时保存源码与静态网页。也可以将文章源码新建一个repository单独保存，前面创建hexo项目的代码稍作修改。

- 创建
    ```
    cd ..
    hexo init myblog
    cd myblog
    ```

- 不用修改public_dir的值（使用默认的public），生成完成后将文件拷贝到username.github.io里并push到github，或者安装hexo-depoloy-git
    ```
    npm install hexo-deployer-git --save
    ```
- 修改配置_config.yml

    ```
    deploy:
      type: git
      repo: <repository url>
      branch: [branch]
      message: [message]
    ```

|Option|Description|
|--------|---------|
|repo|GitHub/Bitbucket/Coding/GitLab repository URL|
|branch|分支名. 如果使用GitHub或者GitCafe则不用配置，部署器会自行检测|
|message|自定义提交信息(默认值为Site updated: 当前时间)|

- 部署
    ```
    hexo deploy
    ```

##### 实用的插件

安装插件时，注意要在源码目录(/src)里，安装后重启hexo server

- 修改md文件后自动刷新浏览器，免去手动刷新网页的烦恼

```
npm install --save hexo-browsersync
```

##### 自定义域名

GitHub Pages支持自定义域名，配置方式：

1. 配置GitHub

    在网页端打开https://github.com/username/username.github.io，点击Settings，在下方GitHub Pages栏中设置Custom domain为自己的域名。
    
2. 配置DNS

    修改DNS记录，将自己域名的CNAME值设为username.github.io

配置好后可使用自己的域名访问GitHub Pages，更多说明见[官方文档](https://help.github.com/articles/quick-start-setting-up-a-custom-domain/)。
##### 其他

如果重新克隆了源码，记得安装一次依赖，在src目录中执行

```
npm install
```
