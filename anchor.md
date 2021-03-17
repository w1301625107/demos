# 简单带大家写一个锚点高亮点代码

## 起因
前几天公司要求写一个锚点自动高亮功能，页面滚动到哪里锚点高亮到哪里。

所以这里简单记录一下，因为遇到了自动高亮判定的问题，所以发在这里，__如果有更好的方法，希望能教教我～～__

## 效果图
![](imgs/Snipaste_2019-11-30_18-39-44.png)


## __[点击这里看运行效果](https://w1301625107.github.io/demos/anchor.html)__



## 分析一下实现步骤

这是大概的dom结构
```html
<html> 
  <body>
    <ul>
      <li><a href="#a1">a1</a></li>
      <li><a href="#a2">a2</a></li>
      <li><a href="#a3">a3</a></li>
      <li><a href="#a4">a4</a></li>
      <li><a href="#a5">a5</a></li>
    </ul>
    <div id="container">
      <div id="a1"></div>
      <div id="a2"></div>
      <div id="a3"></div>
      <div id="a4"></div>
      <div id="a5"></div>
    </div>
  </body>
</html>
````

### 需要处理的元素

* 页面中需要处理的有 `a` 标签 即图中 `anchor`
* 锚点的位置即各种带 `id` 的元素，如图中 `a1 element` 和 `a2 element`
* 包裹这些锚点位置的`容器`，如果是整个页面那就是 `document` ，或者就是自定义的元素，这里统一叫滚动容器`container`

![](imgs/Snipaste_2019-11-30_18-50-41.png)


### 如何手动高亮 a 标签

高亮效果，这里就是简单的给对应的 `a` 标签添加一个`class`即可

``` js
// 获取a所有元素
function getAllAnchorElement(container) {
    const target = container ? container : document
    return target.querySelectorAll('a')
}

// 对应id的添加高亮类名，非对应id移除之前添加的高亮类名
function highLightAnchor(id){
  getAllAnchorElement().forEach(element => {
    element.classList.remove('highLight')
    if (element.hash.slice(1) == id) {
      element.classList.add('highLight')
    }
  });
}

```

### 如何自动高亮 a 标签
原理很简单，就是监听容器元素的滚动，在对应条件下去自动高亮a标签
```js
// 这里注意用 thorttle 处理一下handleScroll 函数，不然触发次数太多可能卡页面哦 🤪🤪
const throttleFn = throttle(handleScroll, 100)
// 如果你没有阻止滚动事件，可以加上 passive: true 可以提高滚动性能
ScrollContrainer.addEventListener('scroll', throttleFn, {passive: true})
```
__但是这个`对应条件`有点麻烦，我一共想了3种方法，各有缺点和优点，如果大家有完美的方法，不吝赐教__

我这里方案都是通过 `getBoundingClientRect` 来判断元素的位置。

这里贴一点MDN上的解释
> 返回值是一个 DOMRect 对象，这个对象是由该元素的 getClientRects() 方法返回的一组矩形的集合, 即：是与该元素相关的CSS 边框集合 。

> DOMRect 对象包含了一组用于描述边框的只读属性——left、top、right和bottom，单位为像素。除了 width 和 height 外的属性都是相对于视口的左上角位置而言的。

> ![](imgs/rect.png)

我这里3种方案，先说第一种
### __谁冒头就高亮谁__
```js
let highligthId;//需要高亮的id
const windowHeight = this.ScrollContrainer.offsetHeight //容器高度
this.anchors.forEach(element => {
  const id = element.hash.slice(1)
  const target = document.getElementById(id)
  if (target) {
    const {
      top
    } = target.getBoundingClientRect()
    // 当元素头部可见时
    if (top < windowHeight) {
      highligthId = id
    }
  }
})
if (highligthId) {
  // 调用高亮方法
  this.highLightAnchor(highligthId)
}
```
#### 优点
* 简单

#### 缺点
* 初始状态如果 `第一个元素` 没有满屏比如 `a1` ，屏幕下方漏出一点 `a2` 元素，那 `a1` 元素的高亮会里面跳走，这个时候 `a1` 再也高亮不到了
![](imgs/Snipaste_2019-11-30_19-23-53.png)


### __谁占据屏幕比例大就高亮谁__
```js
let highligthId;
let maxRatio = 0 // 暂居屏幕的比例
const windowHeight = this.ScrollContrainer.offsetHeight
this.anchors.forEach(element => {
  const id = element.hash.slice(1)
  const target = document.getElementById(id)
  if (target) {
    let visibleRatio = 0;
    let {
      top,
      height,
      bottom
    } = target.getBoundingClientRect();
    // 当元素全部可见时
    if (top >= 0 && bottom <= windowHeight) {
      visibleRatio = height / windowHeight
    }
    // 当元素就头部可见时
    if (top >= 0 && top < windowHeight && bottom > windowHeight) {
      visibleRatio = (windowHeight - top) / windowHeight
    }
    // 当元素占满屏幕时
    if (top < 0 && bottom > windowHeight) {
      visibleRatio = 1
    }
    // 当元素尾部可见时
    if (top < 0 && bottom > 0 && bottom < windowHeight) {
      visibleRatio = bottom / windowHeight
    }
    if (visibleRatio >= maxRatio) {
      maxRatio = visibleRatio;
      highligthId = id;
    }
  }
});
if (highligthId) {
  this.highLightAnchor(highligthId)
}
```

#### 优点
* 当每一个元素都大于半屏时，效果好

#### 缺点
* 在页面最顶部 如果 `a2` 的比例比 `a1` 大，那 `a1` 就无法高亮了
![](imgs/Snipaste_2019-11-30_19-53-20.png)
* 在页面最底部，如图中 `a5` 比 `a4` 小，那么 `a5` 就无法高亮，因为 `a5` 占据屏幕的比例是大不过 `a4` 的
![](imgs/Snipaste_2019-11-30_19-46-45.png)


### __谁显示的自身百分比大就高亮谁__
代码判断条件和 谁占据屏幕比例大就高亮谁 的方案一样，就是分母不一样
```js
let highligthId;
let maxRatio = 0
const windowHeight = this.ScrollContrainer.offsetHeight
this.anchors.forEach(element => {
  const id = element.hash.slice(1)
  const target = document.getElementById(id)
  if (target) {
    let visibleRatio = 0;
    let {
      top,
      height,
      bottom
    } = target.getBoundingClientRect();
    // 当元素全部可见时
    if (top >= 0 && bottom <= windowHeight) {
      visibleRatio = 1
    }
    // 当元素就头部可见时
    if (top >= 0 && top < windowHeight && bottom > windowHeight) {
      visibleRatio = (windowHeight - top) / height
    }
    // 当元素占满屏幕时
    if (top < 0 && bottom > windowHeight) {
      visibleRatio = windowHeight / height
    }
    // 当元素尾部可见时
    if (top < 0 && bottom > 0 && bottom < windowHeight) {
      visibleRatio = bottom / height
    }
    if (visibleRatio >= maxRatio) {
      maxRatio = visibleRatio;
      highligthId = id;
    }
  }
});
if (highligthId) {
  this.highLightAnchor(highligthId)
}
```
#### 优点
* 当每一个元素都大于半屏时，效果好
* 在有连续出现小元素的时候效果会比`谁占据屏幕比例大就高亮谁的方案`好一点

#### 缺点
* 如图所示，`a1` 和 `a2` 都全部显示，那 `a1` 的优先级就没有 `a2` 高，无法高亮
![](imgs/Snipaste_2019-11-30_19-58-16.png)

* 如图所示当 `a3` 很小，那么 `a3` 出现后就每次计算比例都是 `1` ，a4 就必须等 `a3` 开始消失，才有可能高亮
![](imgs/Snipaste_2019-11-30_20-05-28.png)

## 最后

通过以上可以看到我能想到的方案在某些情况下都有问题。

在我的工作中因为每一个元素都至少有半屏都大小，所以`谁占据屏幕比例大就高亮谁`这个方案效果是综合起来效果最好的。

其实一开始用的是`谁显示的自身百分比大就高亮谁`这个方案，但是这个方案在某一个元素`特别特别长`的时候效果就差点。

## 如果以上内容对你有帮助，请问可以骗个赞吗 👍👍👍👍

## 最后的最后附上全部代码
```html
<html>
<body>
  <nav>
    <ul>
      <li><a href="#a1">a1</a></li>
      <li><a href="#a2">a2</a></li>
      <li><a href="#a3">a3</a></li>
      <li><a href="#a4">a4</a></li>
      <li><a href="#a5">a5</a></li>
    </ul>
    <input type="radio"
           name="strategy"
           checked
           value="type1">冒头就高亮<br><br>
    <input type="radio"
           name="strategy"
           value="type2">占据屏幕比例大就高亮<br><br>
    <input type="radio"
           name="strategy"
           value="type3">显示的自身百分比大就高亮<br><br>
    <p>颜色右下角可以拖动改变对应元素的大小</p>
  </nav>

  <div id="container">
    <div id="a1"></div>
    <div id="a2"></div>
    <div id="a3"></div>
    <div id="a4"></div>
    <div id="a5"></div>
  </div>
</body>
<script>
  function throttle(fn, interval = 1000) {
    let timer = null;
    return function(...args) {
      if (!timer) {
        timer = setTimeout(() => {
          timer = null
          fn.call(this, ...args)
        }, interval);
      }
    }
  }

  class AutoHighLightAnchor {
    anchors;
    ScrollContrainer;
    throttleFn;
    strategy;
    constructor(anchorsContainer, ScrollContrainer, strategy = AutoHighLightAnchor.Strategys.type1) {
      this.anchors = anchorsContainer.querySelectorAll('a')
      this.ScrollContrainer = ScrollContrainer;
      this.strategy = strategy;
      this.init()
    }

    init(strategy = this.strategy) {
      if (this.throttleFn) {
        this.remove()
      }
      this.throttleFn = throttle(this[strategy].bind(this), 100)
      this.throttleFn() // 初始执行一次更新位置
      this.ScrollContrainer.addEventListener('scroll', this.throttleFn, {
        passive: true
      })
    }
    remove() {
      this.ScrollContrainer.removeEventListener('scroll', this.throttleFn, {
        passive: true
      })
    }

    highLightAnchor(id) {
      this.anchors.forEach(element => {
        element.classList.remove('highLight')
        if (element.hash.slice(1) == id) {
          element.classList.add('highLight')
        }
      });
    }

    type1(e) {
      let highligthId;
      const windowHeight = this.ScrollContrainer.offsetHeight
      this.anchors.forEach(element => {
        const id = element.hash.slice(1)
        const target = document.getElementById(id)
        if (target) {
          const {
            top
          } = target.getBoundingClientRect()
          // 当元素头部可见时
          if (top < windowHeight) {
            highligthId = id
          }
        }
      })
      if (highligthId) {
        this.highLightAnchor(highligthId)
      }
    }

    type2(e) {
      let highligthId;
      let maxRatio = 0
      const windowHeight = this.ScrollContrainer.offsetHeight
      this.anchors.forEach(element => {
        const id = element.hash.slice(1)
        const target = document.getElementById(id)
        if (target) {
          let visibleRatio = 0;
          let {
            top,
            height,
            bottom
          } = target.getBoundingClientRect();
          // 当元素全部可见时
          if (top >= 0 && bottom <= windowHeight) {
            visibleRatio = height / windowHeight
          }
          // 当元素就头部可见时
          if (top >= 0 && top < windowHeight && bottom > windowHeight) {
            visibleRatio = (windowHeight - top) / windowHeight
          }
          // 当元素占满屏幕时
          if (top < 0 && bottom > windowHeight) {
            visibleRatio = 1
          }
          // 当元素尾部可见时
          if (top < 0 && bottom > 0 && bottom < windowHeight) {
            visibleRatio = bottom / windowHeight
          }
          if (visibleRatio >= maxRatio) {
            maxRatio = visibleRatio;
            highligthId = id;
          }
        }
      });
      if (highligthId) {
        this.highLightAnchor(highligthId)
      }
    }

    type3(e) {
      let highligthId;
      let maxRatio = 0
      const windowHeight = this.ScrollContrainer.offsetHeight
      this.anchors.forEach(element => {
        const id = element.hash.slice(1)
        const target = document.getElementById(id)
        if (target) {
          let visibleRatio = 0;
          let {
            top,
            height,
            bottom
          } = target.getBoundingClientRect();
          // 当元素全部可见时
          if (top >= 0 && bottom <= windowHeight) {
            visibleRatio = 1
          }
          // 当元素就头部可见时
          if (top >= 0 && top < windowHeight && bottom > windowHeight) {
            visibleRatio = (windowHeight - top) / height
          }
          // 当元素占满屏幕时
          if (top < 0 && bottom > windowHeight) {
            visibleRatio = windowHeight / height
          }
          // 当元素尾部可见时
          if (top < 0 && bottom > 0 && bottom < windowHeight) {
            visibleRatio = bottom / height
          }
          if (visibleRatio >= maxRatio) {
            maxRatio = visibleRatio;
            highligthId = id;
          }
        }
      });
      if (highligthId) {
        this.highLightAnchor(highligthId)
      }
    }

  }
  AutoHighLightAnchor.Strategys = {
    type1: 'type1',
    type2: 'type2',
    type3: 'type3'
  }

  const high = new AutoHighLightAnchor(document.querySelector('ul'), document.querySelector('#container'),
    AutoHighLightAnchor.Strategys.type1)

  document.querySelectorAll('input[type=radio]').forEach(element => {
    element.onchange = e => high.init(e.target.value)
  })
</script>
<style>
  body {
    margin: 0;
  }

  a {
    display: block;
    width: 100%;
    height: 100%;
    color: #898A95;
    line-height: 30px;
    border-radius: 0 50px 50px 0;
    text-decoration: none;
    text-align: center;
  }

  .highLight {
    color: #ffffff;
    background: #77b9e1;
  }

  nav {
    width: 120px;
    float: left;
  }

  ul {
    width: 100px;
    display: flex;
    flex-direction: column;
    margin: 0;
    padding: 0;
    list-style: none;
  }

  #container {
    height: 100%;
    overflow: scroll;
  }

  #container>div {
    position: relative;
    resize: vertical;
    overflow: scroll;
    /* color: #ffffff;
    font-size: 30px; */
  }

  #container>div::after {
    content: attr(id);
    display: block;
    font-size: 100px;
    color: #fff;
    text-align: center;
    width: 100%;
    position: absolute;
    top: 50%;
    transform: translateY(-50%);
  }

  #container>div:hover {
    outline: 1px dashed #09f;
  }

  #container>div::-webkit-scrollbar {
    width: 25px;
    height: 20px;
  }

  #a1 {
    height: 100vh;
    background-color: #77b9e1;
  }


  #a2 {
    height: 50vh;
    background-color: #9fc6e6;
  }

  #a3 {
    height: 33vh;
    background-color: #73a5d7;
  }

  #a4 {
    height: 33vh;
    background-color: #1387aa;
  }

  #a5 {
    height: 20vh;
    background-color: #0c5ea8;
  }
</style>

</html>
```


