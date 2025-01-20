---
title: Hexo Butterfly 星空配置
date: 2025-01-20 13:17:33
categories:
    - 教程
tags:
    - Hexo
comments: true
description: Hexo 星空背景效果配置教程
top_img: https://anne416wu.github.io/gallery/thumbnails/hexo-cover.png
cover: https://anne416wu.github.io/gallery/thumbnails/hexo-cover.png
---

> 这一篇文章主要是作为记录, 备份星空效果的添加方式. 其中借鉴了博主 [Barry-Flynn](https://blog.meta-code.top/2021/09/30/2021-7/) 的解决方案

## 前置条件

根据 [butterfly](https://butterfly.js.org/posts/dc584b87/) 配置中描述的, 确实在最新版本的 butterfly 中只要在 hexo 项目的根目录添加文件 `_config.butterfly.yml` 即可实现 butterfly 的主题配置. 但是因为在设置星空特效的时候依旧需要涉及到一些 `css` 和 `javascript` 代码的编写, 所以还是需要吧 butterfly 的项目代码通过下面的命令复制到项目中

```bash
# stable
git clone -b master https://github.com/jerryc127/hexo-theme-butterfly.git themes/butterfly
# dev
git clone -b dev https://github.com/jerryc127/hexo-theme-butterfly.git themes/butterfly

cp _config.butterfly.yml themes/butterfly/_config.yml
```

上面的操作缺失. 导致在一开始的时候思考了好久为什么所有的教程中都有类似源码的文件, 但是我在执行文档中的 npm 安装 `npm install hexo-theme-butterfly` 之后并没有对应的 butterfly 配置文件.

## 编辑 JS 和插入 Canvas 标签

在文件目录 `themes/butterfly/js` 文件夹中新建文件 `universe.js`, 并且插入如下代码

```javascript
function dark() {
    window.requestAnimationFrame =
        window.requestAnimationFrame ||
        window.mozRequestAnimationFrame ||
        window.webkitRequestAnimationFrame ||
        window.msRequestAnimationFrame;
    var n,
        e,
        i,
        h,
        t = 0.05,
        s = document.getElementById("universe"),
        o = !0,
        a = "180,184,240",
        r = "226,225,142",
        d = "226,225,224",
        c = [];

    function f() {
        (n = window.innerWidth),
            (e = window.innerHeight),
            (i = 0.216 * n),
            s.setAttribute("width", n),
            s.setAttribute("height", e);
    }

    function u() {
        h.clearRect(0, 0, n, e);
        for (var t = c.length, i = 0; i < t; i++) {
            var s = c[i];
            s.move(), s.fadeIn(), s.fadeOut(), s.draw();
        }
    }

    function y() {
        (this.reset = function () {
            (this.giant = m(3)),
                (this.comet = !this.giant && !o && m(10)),
                (this.x = l(0, n - 10)),
                (this.y = l(0, e)),
                (this.r = l(1.1, 2.6)),
                (this.dx =
                    l(t, 6 * t) +
                    (this.comet + 1 - 1) * t * l(50, 120) +
                    2 * t),
                (this.dy =
                    -l(t, 6 * t) - (this.comet + 1 - 1) * t * l(50, 120)),
                (this.fadingOut = null),
                (this.fadingIn = !0),
                (this.opacity = 0),
                (this.opacityTresh = l(0.2, 1 - 0.4 * (this.comet + 1 - 1))),
                (this.do = l(5e-4, 0.002) + 0.001 * (this.comet + 1 - 1));
        }),
            (this.fadeIn = function () {
                this.fadingIn &&
                    ((this.fadingIn = !(this.opacity > this.opacityTresh)),
                    (this.opacity += this.do));
            }),
            (this.fadeOut = function () {
                this.fadingOut &&
                    ((this.fadingOut = !(this.opacity < 0)),
                    (this.opacity -= this.do / 2),
                    (this.x > n || this.y < 0) &&
                        ((this.fadingOut = !1), this.reset()));
            }),
            (this.draw = function () {
                if ((h.beginPath(), this.giant))
                    (h.fillStyle = "rgba(" + a + "," + this.opacity + ")"),
                        h.arc(this.x, this.y, 2, 0, 2 * Math.PI, !1);
                else if (this.comet) {
                    (h.fillStyle = "rgba(" + d + "," + this.opacity + ")"),
                        h.arc(this.x, this.y, 1.5, 0, 2 * Math.PI, !1);
                    for (var t = 0; t < 30; t++)
                        (h.fillStyle =
                            "rgba(" +
                            d +
                            "," +
                            (this.opacity - (this.opacity / 20) * t) +
                            ")"),
                            h.rect(
                                this.x - (this.dx / 4) * t,
                                this.y - (this.dy / 4) * t - 2,
                                2,
                                2
                            ),
                            h.fill();
                } else
                    (h.fillStyle = "rgba(" + r + "," + this.opacity + ")"),
                        h.rect(this.x, this.y, this.r, this.r);
                h.closePath(), h.fill();
            }),
            (this.move = function () {
                (this.x += this.dx),
                    (this.y += this.dy),
                    !1 === this.fadingOut && this.reset(),
                    (this.x > n - n / 4 || this.y < 0) && (this.fadingOut = !0);
            }),
            setTimeout(function () {
                o = !1;
            }, 50);
    }

    function m(t) {
        return Math.floor(1e3 * Math.random()) + 1 < 10 * t;
    }

    function l(t, i) {
        return Math.random() * (i - t) + t;
    }
    f(),
        window.addEventListener("resize", f, !1),
        (function () {
            h = s.getContext("2d");
            for (var t = 0; t < i; t++) (c[t] = new y()), c[t].reset();
            u();
        })(),
        (function t() {
            document
                .getElementsByTagName("html")[0]
                .getAttribute("data-theme") == "dark" && u(),
                window.requestAnimationFrame(t);
        })();
}
dark();
```

根据上面代码的最后一小段, 这个主题目前应该只是在 `dark` 模式下才会生效

```javascript
function t() {
    document.getElementsByTagName('html')[0].getAttribute('data-theme') == 'dark' && u(), window.requestAnimationFrame(t)
}()
```

之后打开 butterfly 主题的配置文件 (我的是 `_config.butterfly.yml`), 查找到 `inject` 部分插入下面代码

```yaml
inject:
	bottom:
	    - '<canvas id="universe"></canvas>' # 背景宇宙星光canvas
	    - '<script src="/js/universe.js"></script>' # 背景宇宙星光js文件
	      # - <script src="xxxx"></script>
```

## 配置 CSS 文件

在完成上面部分之后, 为了正确显示效果, 需要 css 文件来设定样式. 我写在目录 `themes/butterfly/themes/butterfly/source/css` 下面创建了文件 `universe.css`. 并添加如下内容

```javascript
/* 夜间模式下背景宇宙星光 */
#universe {
    display: block;
    position: fixed;
    margin: 0;
    padding: 0;
    border: 0;
    outline: 0;
    left: 0;
    top: 0;
    width: 100%;
    height: 100%;
    pointer-events: none;
    z-index: 1;
}
/* 亮色模式下隐藏 */
[data-theme='light'] #universe {
    display: none;
}
```

然后在上面的 `_config.yml` 文件中添加上这部分 css 的代码引用

```javascript
inject: head: -'<link rel="stylesheet" href="/css/universe.css">';
```

## 结束!

恭喜你也可以晚上在手机里看到星星了!
