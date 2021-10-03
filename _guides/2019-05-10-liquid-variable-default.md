---
title: "Jekyll/Liquid assign boolean issues"
tags: [Liquid, variable, default, bool]
---
 **Written by © FolioDavid**

 **Published on December 13, 2018, at 13:44:09**

`Liquid` is not a programming language! Take care on the use of boolean values…

When dealing with Jekyll template and use of Liquid language one may take care when assing boolean expression to a variable: its does not work!!!!:

The assign tag only supports raw booleans (ie. `true` or `false`). Trying to assign a boolean expresion such as:
```
{% raw %}
{% assign vars = 2 == 1 %} 
{% endraw %}
```
inexplicably assigns “2” to vars.
To my knowledge, the only way is to use conditional statement to assign boolean values, such as follows:
```
{% raw %}
{% assign enable_A=false %}
{% assign enable_B=true %}
{% if page.param1==true or page.param2==true  %}{%assign enable_A=true%}{% endif %}
{% if page.param2==false or enable_A==true    %}{%assign enable_B=false%}{% endif %}
{% endraw %}
```

Secondly, when mixing the use of boolean variable with the Liquiddefault filter, one may also take care on the ‘basic’ behavior of the filter. Basically in the following snippet code:
```
{% raw %}
{% assign varA = false %}
{% assign vars = varA | default: true %}
{% endraw %}
```
vars is set to the default true value as the Liquiddefault filter will set the value only if “the left side is nil, false, or empty”.

# 参考文献

- [Jekyll/Liquid assign boolean issues](http://dfolio.free.fr/articles/2018/12/liquid-assign-boolean/)
