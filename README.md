# Jekyll-Jacman

**中文 | [English](/README_en.md)**

本项目是从[jekyll-jacman](https://github.com/Simpleyyt/jekyll-jacman) fork 而来，主要用于记录日常遇到的一些问题。

 * [主题演示](http://simpleyyt.github.io/jekyll-jacman/)
 * [如何使用 Jacman 主题](http://simpleyyt.github.io/jekyll-jacman/jekyll/2015/09/20/how-to-use-jacman)

## 本地搭建

下面的安装内容来自[jekyll官网](https://jekyllrb.com/docs/installation/ubuntu/)

Before we install Jekyll, we need to make sure we have all the required dependencies.
```sh
sudo apt-get install ruby-full build-essential zlib1g-dev
```
It is best to avoid installing Ruby Gems as the root user. Therefore, we need to set up a gem installation directory for your user account. The following commands will add environment variables to your ~/.bashrc file to configure the gem installation path. Run them now:
```sh
echo '# Install Ruby Gems to ~/gems' >> ~/.bashrc
echo 'export GEM_HOME="$HOME/gems"' >> ~/.bashrc
echo 'export PATH="$HOME/gems/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```
Finally, install Jekyll:

```sh
gem install jekyll bundler
```
That’s it! You’re ready to start using Jekyll.

到此，需要的环境安装完成.

下载本repo代码
```sh
git clone https://github.com/aimuke/aimuke.github.io.git
cd aimuke.github.io
```
安装依赖：

```sh
bundle install
```

运行 Jekyll：

```sh
./run.sh
```

更多细节可以参考：[Setting up your GitHub Pages site locally with Jekyll](https://help.github.com/articles/setting-up-your-github-pages-site-locally-with-jekyll/)

## 功能

- **菜单 menu**  
 主导航菜单
- **控件 widget**  
 侧边栏的控件。包括：Github 名片	、分类、标签、RSS、友情链接、微博秀。
- **图片相关 Image**  
 设置网站图标、网站logo、作者头像、博客顶部大图等。还提供了多种图片样式`img-logo`,`img-topic`,`img-center`等。
- **首页模式 index**  
 主题提供了两种首页展示模式。
- **作者 author**  
 作者信息，主要用于展示网站右下角的社交网络链接。包括：微博、豆瓣、知乎、邮箱、GitHub、StackOverflow、Twitter、Facebook、Linkedin、Google+。
- **目录 toc**  
 在文章中和侧边栏可以显示目录。
- **评论 comments**  
 支持 [多说](http://duoshuo.com/) & [disqus](https://disqus.com/) 评论。
- **分享 jiathis**  
 启用 内建分享工具 或 [加网](http://www.jiathis.com/) 分享系统。
- **网站统计 Analytiscs**  
 支持 [谷歌统计](http://www.google.com/analytics/) & [百度统计](http://tongji.baidu.com/) & [CNZZ站长统计](http://www.cnzz.com/)。
- **Search**  
 支持 [谷歌自定义搜索](https://www.google.com/cse/ ) & [百度站内搜索](http://zn.baidu.com/)  &[微搜索](http://tinysou.com/)。 &[Simple Jekyll Search](https://github.com/christian-fei/Simple-Jekyll-Search)
- **totop**  
 回到顶部。
- **rss**  
 RSS 订阅链接。
- **fancybox**  
 图片查看的 [Fancybox](http://fancyapps.com/fancybox/) 工具。
- **其他**
 你可以设置侧边栏在博文页面中不显示。

## 协议

[MIT](/LICENSE)

