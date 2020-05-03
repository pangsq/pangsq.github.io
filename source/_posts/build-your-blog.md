---
title: 使用Hexo + Github Pages搭建个人博客
comments: true
date: 2020-05-03 16:17:59
tags:
- Hexo
- Github Pages
- Travis CI
- SEO
- Realm
categories:
- 鼓捣点儿什么
---

本文记录利用 `Hexo + Github Pages + Travis CI + 域名` 搭建个人博客的过程。

## Hexo作为博客框架

[Hexo](https://hexo.io/zh-cn/)是一个广泛使用的博客框架，上手简单，页面简洁，非常适合用于发布个人的一些文章。

Hexo项目可以直接跑在服务器上，或者依托于Github Pages之类的平台进行托管。无论最终网站运行在哪，在搭建Hexo项目和创作文章过程中还是需要有一台电脑或服务器能够运行Hexo。建议用mac或linux，配环境省事点；直接用win或者WSL也可以。

因为这几天手头只有台win10的笔记本，所以我用的阿里云的一台ecs(centos 7.2)运行Hexo，用vscode remote初始化项目和进行修改。

1. 安装或更新nodejs。对于centos最简单的做法是 `sudo yum install -y nodejs`，但是由于yum源的nodejs版本比较老，所以直接到 https://nodejs.org/en/download/ 下载二进制包(v14.0.0)，解压后配置PATH。
2. 安装hexo-cli `npm install hexo-cli -g`
3. 初始化hexo项目 `hexo init blog`。blog是项目的目录。刚初始化的项目中已存在一篇helloWorld文章。
4. 创建新文章 `hexo new page "<page name>"`，文章格式是Markdown的
    1. 注意`hexo new page`是以scaffolds/page.md为模板在source/_posts中创建相应Markdown文件，post/draft同理，所以可以根据自己需要事先修改这几个模板文件
    2. 在文章中插入图片需要注意，直接将图片放在与Markdown同级目录下，然后用相对地址，Hexo是渲染不出来的；需要在_config.yml中配置 `post_asset_folder: true`，那么`hexo new`创建文章时会一并创建一个同名的目录，将图片放置在该目录下，然后再用Markdown插入图片的方式就能生效了
5. 默认情况下hexo运行后，导航栏中会存在catagories和tags两个栏目，但点击跳转会显示404，需要事先进行简单配置
    1. 用`hexo new page categories`和`hexo new page tags`创建categories和tags，然后修改它们的type以及layout，分别修改为categories和tags
    2. 在_config.yml中配置
```yml
nav: 
  home: /
  about: /about
  tags: /tags
  catagories: /catagories
```
6. 默认用的`landscape`主题，可以在[themes](https://hexo.io/themes/)挑选主题，放到themes/目录下，并在_config.yml中配置theme启用
7. 运行Hexo `hexo server`，监听4000端口，此时浏览器可以进行访问
8. Hexo使用上的更多内容见[Hexo文档](https://hexo.io/zh-cn/docs/index.html)

## Github Pages托管博客

[Github Pages](https://pages.github.com/),可以对我们的博客进行托管，节省自己维护服务器的精力。

1. 在github上创建一个名为`<username>.github.io`的repo（用这个格式作为repo名字的原因是只有这个名字可以在启用Github Pages时直接拿来当url用，其他repo的url只能是`<username>.github.io/<repo name>`
2. 在repo的settings中开启Github Pages，默认情况（不花钱）下只能选择master作为发布分支（即想让Github Pages托管的网站文件都必须放在master分支，其他分支是无关的，想放什么都可以）
3. 将之前hexo init所在目录git init，并关联刚创建的repo（下一小节再推送）

## Travis CI自动发布

Travis CI我们一般用来做自动化集成，这里也可以用来做Hexo项目的自动化构建。

Hexo运行时真正起作用的是public/文件夹内的内容，在Hexo项目中使用`hexo generate`可生成public/，将其放到Github Pages关联的分支中，就能让Github Pages把博客网站跑起来了。

1. 参考https://hexo.io/zh-cn/docs/github-pages 配置travis-ci，将其与我们的github账号联动起来
2. 修改.travis.yml中的master为source，并添加target_branch: master。这是由于我们只能选择master作为发布分支，所以以source分支维护我们的项目。
```yml
sudo: false
language: node_js
node_js:
  - 10 # use nodejs v10 LTS
cache: npm
branches:
  only:
    - source # build master branch only
script:
  - hexo generate # generate static files
deploy:
  provider: pages
  skip-cleanup: true
  github-token: $GH_TOKEN
  keep-history: true
  on:
    branch: source
  local-dir: public
  target_branch: master
```
3. 将本地的分支推送到github中作为source分支。`git push origin master:source`
4. 关注travis-ci中的构建情况，如不出意外，几分钟后访问 `http://<username>.github.io` 就能看到博客效果了。

至此，个人博客已经成型，以后修改文章、创建文章都在source分支中进行，travis-ci会自动帮我们进行构建。

## 绑定个人域名

如果自己有域名的话，可以将自己的域名关联到个人博客上。

由于我之前有在阿里云上申请过域名，该域名直接关联ecs（为了平时敲敲ssh的时候不用背ip），所以打算用一个二级域名去关联博客。

1. 登录阿里云，添加一条DNS解析，给pangsq.cn域名（主机为blog）添加CNAME记录指向pangsq.github.io
2. source分支中，在source/目录下添加一个CNAME文件，内容为blog.pangsq.cn
3. 推送source分支到远端；travis ci构建成功后，在master分支也能看到CNAME文件
4. 访问[http://blog.pangsq.cn](http://blog.pangsq.cn)或[https://blog.pangsq.cn](https://blog.pangsq.cn)就能访问博客了

## 让搜索引擎对博客进行收录

完成了上面的工作，博客就能通过个人域名访问到了，但目前还不会被搜索引擎收录。

这里提个醒，Github Pages禁止百度的爬虫访问，所以百度搜索不到博客是正常的。

这里只考虑Google，以后再处理Bing或其他搜索引擎。

1. 使用sitemap，我们可以直接向Google提交我们的网站。首先生成网站的sitemap
    1. 在source分支的代码下，`npm install hexo-generator-sitemap --save`
    2. 在_config.yml中配置
    ```yml
    sitemap:
        path: sitemap.xml
    ```
    3. 推送分支到远端，构建后master分支下会生成sitemap.xml
2. 登录Google的[Search Console](https://search.google.com/search-console)
3. 按照指示验证域名。在域名控制台里，添加一条txt记录。
4. 在Search Console的`Index > Sitemaps`栏目中提交sitemap.xml的url，例如 `https://blog.pangsq.cn/sitemap.xml`
5. 在`URL inspection`中输入博客地址，再点击"REQUEST INDEXING"，会将博客地址加入Google的相对高优先级的队列中供爬虫获取
6. 等待几分钟后能在Google中就搜到博客了（可能是目前网页的权重还比较低，所以关键词是网站的url才容易搜得到；我直接搜博客名是显示在了第二页）
