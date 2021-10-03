---
title: 常见问题
---
记录在构建和使用过程中遇到的有一些常见问题.

# 原始输出

有时需要展示一些与Liquid产生冲突的内容时（如，我们需要展示liquid语句），我们需要暂时性的禁用标签的解析，这时就需要Raw标签了。
```
{% raw %}
{% raw %}
    In Handlebars, {{ this }} will be HTML-escaped, but {{{ that }}} will not. {% if aaa %}
{ % endraw %}  
{% endraw %}  
// 在使用过程中注意去掉`{`和`%`之间的空格，因为我不这样写，我也没有办法输出它...很无奈。
```

# 注释
注释是最简单的标签，它只是把内容包含起来。
```
{% raw %}
{% comment %} 注释内容 {% endcomment %}
{% endraw %}  
```

# Liquid 没有被解析

项目中要被 `Liquid` 解析的文档头部必须以两行 `---` 三横线开头, 否则将不会被解析。

# Liquid 默认值

```
{% raw %}
{{ varA | default: aa}}
{% endraw %}  
```

liquid 语法中使用上述结构获取变量值，和设置默认值。

但是 liquid 认为 当`varA` 的值为 `nil`, `false` 或 empty 时，会设置默认值。

所以，当变量为 `bool` 类型时， `{{ varA | default: true }}`获取到的值永远为 true

可参见 http://dfolio.free.fr/articles/2018/12/liquid-assign-boolean/

# 参考文献
- [搭建一个免费的、无限流量的Blog](https://myzerone.com/posts/2017/08/25/%E6%90%AD%E5%BB%BA%E4%B8%80%E4%B8%AA%E5%85%8D%E8%B4%B9%E7%9A%84-%E6%97%A0%E9%99%90%E6%B5%81%E9%87%8F%E7%9A%84Blog/)
