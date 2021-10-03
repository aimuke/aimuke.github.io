---
title: "网页中最常用的JS代码（js禁止右键、禁止复制）" 
tags: [html, "禁用复制"]
---

经常遇到网页中需要禁用复制或右键等功能。本文介绍了常见的处理方法以及如何破解的方法。

# 禁用事件

禁止复制
```html
<body oncopy="return false">  
```

防止剪切
```html
<input oncut=”return false;” >
```
禁用右键
```html
<body oncontextmenu="return false"></body>
```

```html
<!– 禁用右键: ->
<script>
function stop(){
return false;
}
document.oncontextmenu=stop;
</script>
```

```html
<script>
function document.onmousedown()
{
    if(event.button==2||event.button==3){
        alert( “右健被禁止 “)
        return   false
    }
}
</script>
```

```
onselectstart //取消选取、防止复制
onpaste //不准粘贴
onMouseDown
```

图片下载限制
```html
<script language=”javascript”>
    function Click(){
        if(window.event.srcElement.tagName==”IMG”){
            alert(‘图片直接右键’);
            window.event.returnValue=false;
        }
    }
    document.oncontextmenu=Click;
</script>

<META HTTP-EQUIV=”imagetoolbar” CONTENT=”no”>  
插入图片时加入galleryimg属性
<img galleryimg=”no” src=””>
```


网页防复制代码 禁止查看网页源文件代码

```html
<body oncontextmenu=”return false” ondragstart=”return false” onselectstart =”return false” onselect=”document.selection.empty()” oncopy=”document.selection.empty()” onbeforecopy=”return false” onmouseup=”document.selection.empty()”>

<noscript><iframe src=”/blog/*>”;</iframe></noscript>
```

5. //防止被人frame
```html
<SCRIPT LANGUAGE=javascript>
if (top.location != self.location)top.location=self.location;
</SCRIPT>
```
6. 网页将不能被另存为
```html
<noscript><iframe src=”/blog/*.html>”;</iframe></noscript> //
```

9. //页面禁止刷新完全
最好在pop出来的窗口里用,没工具栏的
```html
<body onkeydown=”KeyDown()” onbeforeunload=”location=location”
oncontextmenu=”event.returnValue=false”>

<script language=”Javascript”><!–
function KeyDown(){
if ((window.event.altKey)&&
((window.event.keyCode==37)||
(window.event.keyCode==39))){ alert(“请访问我的主页”);
event.returnValue=false;
}
if ((event.keyCode==8)|| (event.keyCode==116)){ //屏蔽 F5 刷新键
event.keyCode=0;
event.returnValue=false;
}
if ((event.ctrlKey)&&(event.keyCode==78)){ //屏蔽 Ctrl+n
event.returnValue=false;
}
if ((event.shiftKey)&&(event.keyCode==121)){ //屏蔽 shift+F10
event.returnValue=false;
}
}
</script>
</body
```

# 取消禁用
就是将禁用的事件重新响应即可
```html
<script>
function start() {
    return true
}
document.oncontextmenu = start
</script>
```
# 参考文献

- [网页中最常用的JS代码（js禁止右键、禁止复制）](https://blog.csdn.net/Lpandeng/article/details/53586913)
