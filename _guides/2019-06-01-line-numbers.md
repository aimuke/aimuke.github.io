---
title: "JEKYLL LINE NUMBERS "
tags: [jekyll, 'line numbers']
list_number: false
---

**June 28, 2015**

In this post I’ll describe how to add line numbers to a Jekyll 3 site using Rouge. While this should be a simple task, it actually took me a bit of effort to allow code to be copied without line numbers also being copied. Previously I used highlight.js, which was super fast, but I decided to speed things up even more by switching to Rouge and have the syntax highlighting pre-generated as part of the Jekyll static compilation.

# Demo
Here’s an example with line numbers:

{% highlight ruby %}
def foo
  puts 'foo'
end
{% endhighlight %}

Here’s the same example without line numbers:
{% highlight ruby linenos %}
def foo
  puts 'foo'
end
{% endhighlight %}

Notice that selecting and copying the code in both examples does not also copy the line numbers.

# Pygments Issues
Initially I added line number with Pygments but had issues copying code within the generated code blocks because it would also copy the line numbers. Here’s an example of the code I used:

{% raw %}
```liquid
{% highlight python linenos %}
def fib(n):
 a,b = 1,1
 for i in range(n-1):
  a,b = b,a+b
 return a
print fib(5)
{% endhighlight %}
```
{% endraw %}
I found some documentation describing how adding linenos=table would separate the code from the line numbers. Alas, I had no luck with this either because it jacked up my layout.

# Tutorial
The solution that worked for me was switching to Rouge and doing a bit of CSS.

## Step 1

If you’re using Jekyll 3 then install [Rouge](https://github.com/jneen/rouge) and modify your `_config.yml` to use Rouge:

```yml
highlighter: rouge # or pygments or null
markdown: kramdown # my examples assume you're using kramdown
```
Similarly to Pygments, you can highlight code like the following example:

{% raw %}
```
<figure class="lineno-container">
{% highlight js linenos %}
var recursive = function(n) {
    if(n <= 2) {
        return 1;
    } else {
        return this.recursive(n - 1) + this.recursive(n - 2);
    }
};
{% endhighlight %}
</figure>
```
{% endraw %}
This should generate markup similar to the following:

```html
<figure class="lineno-container">
    <figure class="highlight">
        <pre>
            <code class="language-js" data-lang="js">
                <table>
                    <tbody>
                        <tr>
                            <!-- line numbers -->
                            <td class="gutter gl">...</td>
                            <!-- code -->
                            <td class="code">...</d>
                        </tr>
                    </tbody>
                </table>
            </code>
        </pre>
    </figure>
</figure>
```

## Step 2

It should hopefully be easier to style the line numbers and code now that they’re separate columns (Pygments kept it as a single pre element with spans intermixed with line numbers and code, which made things tricky.

Specifically for my SCSS/CSS, I used the following code:
```scss
// so that the line numbers are not selectable
@mixin unselectable() {
  -webkit-touch-callout: none;
  -webkit-user-select: none;
  -khtml-user-select: none;
  -moz-user-select: none;
  -ms-user-select: none;
  user-select: none;
}
@mixin opacity($opacity) {
  opacity: $opacity;
  $opacity-ie: $opacity * 100;
  filter: alpha(opacity=$opacity-ie); //IE8
}

// having a lineno-container made it easier to style line number 
// code blocks without impacting normal code blocks
.lineno-container > figure > pre {
  padding: 0px;
}

.highlight {
  pre.lineno {
    border: none;
    @include opacity(0.6);  
  }

  .lineno {
    @include unselectable();
  }

  .gutter {
    border-right: 1px solid #ccc;
  }

  pre {
    pre {
      border: none;
      margin-bottom: 0px;
    }
  }
}
```
# 参考文献

- [原文](https://www.minh.io/blog/2015/06/28/jekyll-line-numbers/)
- [Line Numbers in Jekyll Code Blocks](https://www.richwerden.com/2017/line-numbers-in-jekyll-code.html)

