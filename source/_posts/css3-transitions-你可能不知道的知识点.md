title: CSS3 Transitions 你可能不知道的知识点
date: 2013-11-02 20:28:49
tags:
---


## 如何临时让transition失效

我们给一个element设置了transition效果，但某些特殊情况，我们希望让transition临时失效。
我们第一反应就是先移除transition设置，等其他属性设置完成之后再还原transition设置。
但浏览器有时候会让我们感觉事与愿违
看下面这段代码，你觉得会不会有transition动画效果？

```
<div id="test"></div>
<script>
    window.onload = function(){
        var div = document.getElementById("test");
        div.style.width="500px";
        div.style.transition="all 2s ease";
    };
</script>
```

答案是有的。
之所以会出现这种情况，与javascript单线程有关，给dom设置style，只是一个赋值的过程，浏览器引擎不会立即去渲染，所以会看到动画效果。
那么遇到这种情况，如何去解决呢？
大家可能首先会想到 setTimeout()，但感觉不是那么自然。

还有另外一种更好的方法，实用getComputedStyle()方法强制让当前设置生效

```
<div id="test"></div>
<script>
    window.onload = function(){
        var div = document.getElementById("test");
        div.style.width="500px";
        //获取计算后的宽度
        window.getComputedStyle(div).width;
        div.style.transition="all 2s ease";
    };
</script>
```






## transitionend 事件

大家都知道，KISSY1.4中支持了transition动画，用法如下：

```
KISSY.add(function(S,Node,Anim){
    Node.all("#test").animate({
            transform: "translate3d(-100px,0,0)"
        }, {
            duration: .3,
            //增加useTransition配置即可
            useTransition:true,
            easing: "ease-out",
            complete: function(){
                //your code
            }
        });
},{
    requires:['node','anim']
})
```
刚开始很好奇，觉得肯定需要不少代码才能实现支持原生动画，看了[源码](https://github.com/kissyteam/kissy/blob/master/src/anim/sub-modules/transition/src/transition.js)之后才发现其实挺简单，关键点是transitionend事件

```
<style>
#test {
    width:200px;
    height: 200px;
    padding:10px;
    background-color: #8bb8f3;
    transition: all 1s ease;
}
#test:hover {
    background-color: #ff5500;
}
</style>
<div id="test">touch me</div>
<script>
    document.getElementById("test").addEventListener("transitionend",function(ev){
        console.log(ev);
        alert(1);
    })
</script>
```



## transition与visibility

![http://gtms01.alicdn.com/tps/i1/T1GjgcFm4eXXbWpdou-1000-852.png](http://gtms01.alicdn.com/tps/i1/T1GjgcFm4eXXbWpdou-1000-852.png)

```
-webkit-transition: visibility 0s linear .2s;
```

在研究google phone版导航菜单效果的时候，无意中发现css中有上面这么一段。
visibility不就是用来实现显示、隐藏效果的吗，与过度有什么关系呢？
不过直觉告诉我，google的工程师不会这么无聊，写一段毫无用处的代码

用相关的关键字搜索了一下，果然暗藏玄机
不过用一两句话说不明白，直接看这篇文章 [http://www.zhangxinxu.com/wordpress/2013/05/transition-visibility-show-hide/](http://www.zhangxinxu.com/wordpress/2013/05/transition-visibility-show-hide/)

看来任何细节深挖下去都会有收获。



## 启用硬件加速

这个大家可能都知道，方法也有好几种，介绍的文章也多，这里顺便记录一下。

变化某些属性，比如 width，left，浏览器会重新计算每一帧的显示效果，这严重影响速度，尤其是在移动设备上。解决办法就是让 GPU 来做这些运算，简单的说，就是将元素转化为图片再制作变化效果，而不是重新计算每一帧的 CSS 样式。

```
.test{
    //触发GPU加速
    transform: translate3d(0,0,0);
}
```