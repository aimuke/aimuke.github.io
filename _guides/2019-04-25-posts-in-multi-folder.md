---
categories: guide
layout: post
title: 将log按文件夹分类
---

为 Jekyll 设置多个 post 文件夹
将博客迁移到 Jekyll 大半年后，对它基本熟悉了起来。Jekyll 最基本的目录结构是：
```
.
├── _config.yml
├── _data
|   └── members.yml
├── _includes
|   ├── footer.html
|   └── header.html
├── _layouts
|   ├── default.html
|   └── post.html
├── _posts
|   ├── 2017-01-19-hello-world.md
├── _sass
|   ├── _base.scss
|   └── _layout.scss
└── index.html  # Or index.md
└── about.html # Or about.md
```
这其中，比较重要的文件/文件夹有：
- _config.yml：配置文件，填写一些配置信息。
- _includes：在其中添加一些可以在 _layout 和 _posts 中复用的文件。
- _layouts：模板文件，_post 中使用 YAML Front Matter 使用这些模板文件。
- _posts：所有文章都放在这里，注意 文章有特定的文件命名规则，使用 Markdown 格式，并以 YAML Front Matter 作为开头。

但是在这种基本配置中，所有的文章都放在_posts这一个文件夹中。不过，我一直很想将我的文章分成多个文件夹，这样可以实现在不同的页面，显示不同文件夹中的文章，方便管理、归类和查看。尝试了很多方法后，参考了一篇博文[Create a Multi Blog Site with Jekyll](https://www.garron.me/en/blog/multi-blog-site-jekyll.html)，终于成功配置了出来。方法是这样的：

# 新建目录
首先，重新整理目录。如果我想设置两个分类文件，分别是 blog1 和 blog2，那么目录结构变成：
```
.
├── _config.yml
├── _data
|   └── members.yml
├── _includes
|   ├── footer.html
|   └── header.html
├── _layouts
|   ├── default.html
|   └── post.html
|── blog1
|   ├──index.html
|   ├──_posts
|   |   ├──2017-01-19-hello-world.md
|── blog2
|   ├──index.html
|   ├──_posts
|   |   ├──2017-01-19-another-blog.md
├── _sass
|   ├── _base.scss
|   └── _layout.scss
└── index.html
└── about.html
```
# 修改category
接着，在 Yaml Front Matter 制定不同分类的 category，比如将 category 设为 blog1 或者 blog2。
```
---
layout: post
category: blog1
title: 'Hello World'
date: 2017-01-19
---
```
# 设置index.html
然后，在 /blog1 和 /blog2 路径以及根目录下的 index.html 中添加逻辑判断，变量的使用可以参见这篇及之后的文档。这一部分有很多内容，不过没太有时间仔细写了。

```
{% raw %}
  {{ this }} is this, {{ that }} is that.
   {% for post in site.posts %}
      {% if post.categories contains 'blog' %}
          {% capture y %}{{post.date | date:"%Y"}}{% endcapture %}
        <li><span>{{ post.date | date_to_string }} &raquo;</span><a href="{{ post.url }}">{{ post.title }}</a></li>
      {% endif %}
    {% endfor %}
{% endraw %}
```

**PS**
 > `{ % raw %}{ % endraw %}` 是 `Liquid` 的标示符(去除`%`前的空格)，在这中间的内容不会被 `Liquid`解析。在 `jekyll` 生成静态文件时使用 `Liquid`解析，如果不加，那么所有的`{% raw %}{%%}{% endraw %}` 中的内容都将被解析。如上面这个md文档中的内容就会被错误的解析为 `Liquid`

# 完成

最后，打开 ‘xxx.com/blog1’ 和 ‘xxx.com/blog2’，就可以显示不同的文件文章了，根目录下的 index.html 也可以按心愿进行配置。可以查看我的根目录，分类1，和分类2简单查看配置结果。


# 参考文献
[为 Jekyll 设置多个 post 文件](https://dinglisa.com/blog/2017/01/19/jekyll-seperate-blog/)

[Create a Multi Blog Site with Jekyll](https://www.garron.me/en/blog/multi-blog-site-jekyll.html)
