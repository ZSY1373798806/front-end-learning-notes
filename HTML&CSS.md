## DOCTYPE

`<!DOCTYE>` 声明一般位于文档的第一行，它的作用主要是告诉浏览器以什么样的模式来解析文档

## html

### BFC
- 概念
	块级格式化上下文；BFC是CSS布局的一个概念；
	是一个独立的渲染区域，规定了内部box如何布局， 并且这个区域的子元素不会影响到外面的元素，
	其中比较重要的布局规则有内部 box 垂直放置，计算 BFC 的高度的时候，浮动元素也参与计算。
- 布局规则
    - 内部的Box会在垂直方向，一个接一个地放置
    - Box垂直方向的距离由margin决定。属于同一个BFC的两个相邻Box的margin会发生重叠
    - 每个元素的margin box的左边， 与包含块border box的左边相接触(对于从左往右的格式化，否则相反
    - BFC的区域不会与float box重叠
    - BFC是一个独立容器，容器里面的子元素不会影响到外面的元素
    - 计算BFC的高度时，浮动元素也参与计算高度
    - 元素的类型和display属性，决定了这个Box的类型。不同类型的Box会参与不同的	Formatting Context。
- 如何创建
    - 根元素，即HTML元素
    - float的值不为none
    - position为absolute或fixed
    - display的值为inline-block、table-cell、table-caption
    - overflow的值不为visible
- 使用场景
    - 去除边距重叠现象
    - 清除浮动（让父元素的高度包含子浮动元素）
    - 避免某元素被浮动元素覆盖
    - 避免多列布局由于宽度计算四舍五入而自动换行
    - 解决margin塌陷和margin合并问题

## CSS
### flex
- flex:1（flex-grow: 1;flex-shrink: 1;flex-basis: 0%;）
	- flex-grow 定义项目的放大比例，默认为0，即如果存在剩余空间，也不放大
	- flex-shrink 定义了项目的缩小比例，默认为1，即如果空间不足，该项目将缩小
	- flex-basis给上面两个属性分配多余空间之前, 计算项目是否有多余空间, 默认值为 auto, 即项目本身的大小
### position: sticky
- 条件
	- 父元素不能overflow:hidden或者overflow:auto属性。
    - 必须指定top、bottom、left、right4个值之一，否则只会处于相对定位
    - 父元素的高度不能低于sticky元素的高度
    - sticky元素仅在其父元素内生效
### 动画
### 瀑布流
### 重排（回流） & 重绘
- 重排
	dom的变化影响元素的几何信息，浏览器要重新计算生成布局；
- 重绘
	元素外观发生变化，没有改变布局；
- 重排一定会重绘，重绘不一定会重排；
### 盒子模型
- 标准的盒子模型
	width 指 content 部分的宽度。
- IE 盒子模型
	width 表示 content+padding+border 这三个部分的宽度
### rem & em & px
- px与rem的关系，如何单位换算
    - rem
        是指对于根元素（html）的字体大小单位，就是一个相对单位
    - em
        相对父元素的字体大小单位
### sass
- 常用变量

