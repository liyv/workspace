# Viewpager

```xml
viewpager 作为 viewgroup,本质上仍是处理和子View的关系.
分2种情况理解 Viewpager 的实现原理:1.静态;2.动态
```

## 静态

```xml
初始化阶段,viewgroup 的声明周期方法.所不同之处在于 Viewpager 在 onMeasure()阶段实例化子View 并将其添加到 Viewpager
```

### 静态情况下的populate()

```xml
Viewpager 的内部类 Iteminfo 就代表着 单个 子View的重要信息;在当前情况下 mItems 为空,通过adapter的 instantiateItem 实例化第一个 子View 将其包装成1个 iteminfo,并添加到 mitems中,自此 Viewpager 有了第一个 子View / mItems 有了1个iteminfo(iteminfo 封装着 这个item 的positon,widthfactor,offset,object等信息),接下来就是填充 3倍item宽度的空间,即 当前显示的 item 加上左右各1个的宽度,当前是 position=0的item在展示,所以左边没有,右边有1个item(若是 Viewpager 有paddingright则右边会有多个item被实例化),自此填充3倍空间的任务完成;接下来就是计算当前 item(position=0)的offset(offset起着关键作用,offset 是item 左边距Viewpager 左边的的距离(不是像素,是item width的倍数),当前 item 的offset=0,右边的item 的offset=1
```

### layout()

```xml
根据各个item的offset 计算出各child的(l,t,r,b)
```

## 动态

```xml
主要是分析 Viewpager 如何处理手势滑动,在滑动过程中是如何处理各个 item 的,各item 的offset值会变化吗?
```

### onTouchEvent

```xml
1.action_down:记录x,y坐标
2.action_move:调用 scrollTo实现 View UI的滑动,触发 onpagechange等监听方法
3.action_up:此为事件处理的关键所在,首先得到当前滑动位置的 iteminfo,暂且为position=0(即第一个)的,就可得到currentpage(0),pageoffset(0--1之间),totaldelta(滑动距离)等信息,根据这些信息计算出下一个将要展示的 item 的position,知道了position 就可调用 populate(item),scrollToItem等方法展示 下一个View了,
```

### position从 0滑动到1 发生了什么

```xml
主要是分析 populate(1)做了什么
1.定位当前展示的 item : curItem=item_1,curIndex=1;
2.填充 3倍空间:左边是滑过去的 item_0,右边 需要调用 adapter的 instantiateItem 实例化 item_2;
3.计算各 item 的offset: 各item到头来offset都没有变吗?
```

### position 从1 到2 发生了什么

```xml
mItems 和 position ,offset 之间的
```

```java
mGutterSize 的作用是什么
 addNewItem(mCurItem, curIndex);//mCurItem,curIndex分别代表了什么
```