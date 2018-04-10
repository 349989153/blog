---
title: echarts源码阅读修改
date: 2018-04-10 14:13:04
tags: [echarts, js]
categories: 技术
---
产品经理一个蛋疼的需求，着实让我研究了半天。
<!-- more -->
---
问题描述：

echarts有个bug，比如如果用下面这套配置（可以到echarts官网=>作品=>官方示例=>随便打开一个，把配置粘进去就能重现。如果你要弄明白我下面在说什么，请重现这张图，下面说的东西都是以这个option生成的图做例子）：
```javascript
var option = {
    tooltip: {
        trigger: 'axis',
        axisPointer: {
            type: 'cross',
            crossStyle: {
                color: '#999'
            }
        }
    },
    toolbox: {
        feature: {
            dataView: { show: true, readOnly: false },
            magicType: { show: true, type: ['line', 'bar'] },
            restore: { show: true },
            saveAsImage: { show: true }
        }
    },
    legend: {
        data: ['蒸发量', '平均温度']
    },
    xAxis: [{
        type: 'category',
        data: ['1月', '2月', '3月', '4月', '5月', '6月', '7月', '8月', '9月', '10月', '11月', '12月'],
        axisPointer: {
            type: 'shadow'
        }
    }],
    yAxis: [{
            type: 'value',
            name: '水量',
            // min: 50,
            // max: 250,
            // interval: 50,
            axisLabel: {
                formatter: '{value} ml'
            }
        },
        {
            type: 'value',
            name: '温度',
            // min: 20,
            // max: 25,
            // interval: 5,
            axisLabel: {
                formatter: '{value} °C'
            }
        }
    ],
    series: [{
            name: '蒸发量',
            type: 'bar',
            data: [2.0, 4.9, -7.0, 23.2, 25.6, 76.7, 135.6, 162.2, 32.6, 20.0, 6.4, 3.3]
        },
        {
            name: '平均温度',
            type: 'line',
            yAxisIndex: 1,
            data: [22.0, 22.2, 23.3, 24.5, 26.3, 20.2, 20.3, 23.4, 23.0, 26.5, 22.0, 26.2]
        }
    ],
    dataZoom: [{
        yAxisIndex: 1,
        start: 0,
        end: 100
    }]
};
````


在放大缩小之前，一切都是正常的，但是一旦你缩放了，x轴的位置就变了，蒸发量有正有负的数据，也不再在x轴的上下显示。

这很奇怪，我明明放大缩小的是y2轴的数据，凭什么y1轴的样子也要跟着变。

目标：

研究echarts代码，看看是否需要重写部分代码，如果需要，怎么重写，能够解决这个问题。

过程：

首先在浏览器中打断点了之后，一路跟踪下来，发现echarts画图是至少分两步的（就目前知道的情况来说），它会先画出坐标轴，再画数据图像，那我们先看坐标轴怎么画的。

`this._doPaintList(list, paintAll);` 这一句是画坐标轴的函数，其中，参数list是一个array，里面的元素是object，应该是坐标轴上点的各种参数。这个list是通过之前一系列复杂的步骤生成的，在下面的文章中，用list指代list，el指代list的元素。

对list进行循环遍历，首先根据el的Zlevel取出指定的Layer（理解为图层）。这里为啥要先取出Layer，是因为echarts支持多层绘制，每一层是一个单独的canvas，所以el要绘制到指定的层的canvas上去。

//TODO:elFrame是干什么的？elFrame == -1是正常的画图，elFrame >= 0是什么?

接着，在循环遍历里，走到最后一步 `this._doPaintEl(el, currentLayer, paintAll, scope);`  el不用解释了，currentLayer是前面取出来的el指定的图层，//TODO:paintAll和scope不知道是啥

进到 this._doPaintEl(el, currentLayer, paintAll, scope); ，发现最终绘制el的语句是 el.brush(ctx, scope.prevEl || null); ，所以准备进到这个函数里看下，但是看完这个，应该也没必要再深究绘制坐标轴的。

为啥呢？因为很明显，这一部分只是忠实的绘制list里面的el，list是什么样，就会画成什么样，而对于“为什么echarts会这样画这个图？”这个问题，现在可以等价转换为“为什么echarts会生成这个list？”，所以后面我们需要深入研究生成list的算法。但在这之前，先看看el哪些东西影响了画图。

进到 `el.brush(ctx, scope.prevEl || null);` 里面才发现，即使都是list中的el，它们之间也是互不相同的。如果这个el负责画直线，那么它是Line的实例；如果这个el要显示文字，那么它是Text的实例。el可能是不同的类，它们的相似之处是都有一个叫brush的方法，接收相同的参数，完成差不多的任务。

根据list来画图的部分先看到这里，就像之前说的，看看怎么生成的list。

生成list是靠这一句 `var list = this.storage.getDisplayList(true);` ，那么这个 `getDisplayList(true);` 是啥？请看：

```javascript
getDisplayList: function (update, includeIgnore) {
    includeIgnore = includeIgnore || false;
    if (update) {
        this.updateDisplayList(includeIgnore);
    }
    return this._displayList;
},
```

复制代码
在 `getDisplayList(true);` 里，`this`指向的是`Storage`对象，而`Storage._displayList`被直接返回了，那么看看里面的 `this.updateDisplayList(includeIgnore);`是怎么影响`Storage._displayList`的：
```javascript
updateDisplayList: function(includeIgnore) {
  this._displayListLen = 0;
  var roots = this._roots;
  var displayList = this._displayList;
  for (var i = 0, len = roots.length; i < len; i++) {
    this._updateAndAddDisplayable(roots[i], null, includeIgnore);
  }
  displayList.length = this._displayListLen;
  env.canvasSupported && timsort(displayList, shapeCompareFunc);
},
```

在上面这段代码中，this指向Storage对象。而`this._roots`则是一个数组（貌似长度确定是10？），`this._roots`里面的元素是一个Group对象（也就是代码里的roots[i]）传入到了 `this._updateAndAddDisplayable(roots[i], null, includeIgnore);` 。

在`this._updateAndAddDisplayable(roots[i], null, includeIgnore);` 这个函数里不再进行其他调用，说明在这里面对`Storage._displayList`造成了直接影响，而这个`Group`作为直接传入 `this._updateAndAddDisplayable `的参数，肯定有关系。



这个Group是什么呢？

单步调试的时候，研究第一个Group（也就是roots[0]）,发现Group._children是一个数组，有6个元素，5个是Path类，1个是Sub类，其中Sub类是在canvas里绘制基本图形的类，Sub.type如果为'line',那么就会画一条直线，如果为'rect',那么会画矩形。

Path类又是什么？点进去详细调查，发现Path.__title的值，和echarts上面toolbox的内容一一对应，于是恍然大悟，原来第一个Group里装着的是toolbox的零件。

接着调查，第二第三个Group的_children都是空数组，看第四个Group。第四个Group的_children只有一个元素，也是一个Group，它的_children,也就是绝对位置roots[3]._children[0]._children里面有9个元素，里面8个是Sub第一条y轴的横线（就是浅灰色，与x轴等长的那几条），剩下一个又是Group，它的_children里面有18个元素，9个Text9个Sub。调查之后发现，9个Text中有8个是第一条y轴的刻度上文字，剩下1个是第一条y轴的名字；9个sub中有8个是第一条y轴的刻度短横线，非常短的一条，位于刻度上文字的右边，剩下1个是第一条y轴本身。所以我们明白了，第四个Group就是第一条y轴的所有零件，包含8条y轴横线，8个刻度上文字，和对应的8条刻度短刻线，1条y轴和1个对应的y轴文字。

那么，第5个Group我们就能猜想一下了，应该是第2条y轴的零件吧。调查了一下，果不其然，确实是第2条y轴的零件。
第6个Group的_children也是1个元素的数组，这个元素是Group类；这个Group类的_children还是一个1元素的数组，看它的_children，里面有26个元素，12个Text类，14个Sub类，看了一眼，12个Text是x轴刻线上的文字，也就是1月、2月...12月，而13个Sub类是13条x轴的短刻线，还剩下1个Sub类就是x轴本身了。至此，我们第一阶段的目标应该相当明白了，第4个Group和第5个Group，也就是第2条y轴和x轴的零件生成的方式，是我们达成目的的主体。它俩这个Group是怎么生成的，使我们下一阶段的研究对象。

这一段概括一下剩下的Group都是什么，毕竟不是主体，一笔带过就好。第7个Group没有_children；第8个Group是图例的零件，也就是“蒸发量”和“平均温度”那一块；第9个Group是12个蒸发量的小红块儿；第10个Group是平均温度的那条线，包含了12个圆点和把12个圆点连起来的一条折线。

哦，原来整张图在最开始就已经生成好了，那为什么先出现坐标轴再出现的内容呢？有待探究。


----------


好的，那么我们下一阶段的工作就很明确了，假设我们给第2条y轴加上一个新属性叫`origin`，用来表示这条y轴和x轴的交点：

```javascript
yAxis: [{
  type: 'value',
  name: '水量',
  axisLabel: {
    formatter: '{value} ml'
  }
}, {
  type: 'value',
  name: '温度',
  origin: 20,
  axisLabel: {
    formatter: '{value} °C'
  }
}]

```
ok整理一下，我们有Storage._roots这个Array，里面存放着画一张echarts的所有零件，那么这个Storage._roots是在什么阶段生成的，又是怎么生成的?

那么怎么样修改源码才能让这条y轴的各个零件，还有它的图形（在这个例子中，是第10个Group里的点）都能按照图例来生成

接着上次的文章，我们这次需要知道Storage._roots是在什么时候生成的，以及怎么生成的。

我们watch`this._zr.storage._roots`的变化（为什么是这个？参考echarts的类依赖图），在`setOption`里单步调试：
```javascript
echartsProto.setOption = function (option, notMerge, lazyUpdate) {
    if (__DEV__) {
        zrUtil.assert(!this[IN_MAIN_PROCESS], '`setOption` should not be called during main process.');
    }

    var silent;
    if (zrUtil.isObject(notMerge)) {
        lazyUpdate = notMerge.lazyUpdate;
        silent = notMerge.silent;
        notMerge = notMerge.notMerge;
    }

    this[IN_MAIN_PROCESS] = true;

    if (!this._model || notMerge) {
        var optionManager = new OptionManager(this._api);
        var theme = this._theme;
        var ecModel = this._model = new GlobalModel(null, null, theme, optionManager);
        ecModel.init(null, null, theme, optionManager);
    }

    // FIXME
    // ugly
    this.__lastOnlyGraphic = !!(option && option.graphic);
    zrUtil.each(option, function (o, mainType) {
        mainType !== 'graphic' && (this.__lastOnlyGraphic = false);
    }, this);

    this._model.setOption(option, optionPreprocessorFuncs, this.__lastOnlyGraphic);

    if (lazyUpdate) {
        this[OPTION_UPDATED] = { silent: silent };
        this[IN_MAIN_PROCESS] = false;
    } else {
        updateMethods.prepareAndUpdate.call(this);
        // Ensure zr refresh sychronously, and then pixel in canvas can be
        // fetched after `setOption`.
        this._zr.flush();

        this[OPTION_UPDATED] = false;
        this[IN_MAIN_PROCESS] = false;

        flushPendingActions.call(this, silent);
        triggerUpdatedEvent.call(this, silent);
    }
};
```
发现执行完`updateMethods.prepareAndUpdate.call(this);`这一句之后，`this._zr.storage._roots`从`Array(0)`变成了`Array(10)`
在`updateMethods.prepareAndUpdate.call(this);`里：

```javascript
  prepareAndUpdate: function(payload) {
    var ecModel = this._model;

    prepareView.call(this, 'component', ecModel);

    prepareView.call(this, 'chart', ecModel);

    // FIXME
    // ugly
    if (this.__lastOnlyGraphic) {
      each(this._componentsViews, function(componentView) {
        var componentModel = componentView.__model;
        if (componentModel && componentModel.mainType === 'graphic') {
          componentView.render(componentModel, ecModel, this._api, payload);
          updateZ(componentModel, componentView);
        }
      }, this);
      this.__lastOnlyGraphic = false;
    } else {
      updateMethods.update.call(this, payload);
    }
  }
```
在这里面，当运行过`prepareView.call(this, 'component', ecModel);`之后，storage._roots变成了Array(8)，但是这8个Group里面都没有_children；同样的，当运行过`prepareView.call(this, 'chart', ecModel);`之后，storage._roots变成了Array(10)，新加的两个Group也没有实质的内容。可以认为，这两个语句其实是初始化了storage._roots里面的Group。

而当执行了`updateMethods.update.call(this, payload);storage._roots里面的Group开始有了内容。老规矩，还是进到这个函数里面，它太长了，我截取了一部分显示出来：

```javascript
// TODO
// Save total ecModel here for undo/redo (after restoring data and before processing data).
// Undo (restoration of total ecModel) can be carried out in 'action' or outside API call.

// Create new coordinate system each update
// In LineView may save the old coordinate system and use it to get the orignal point
coordSysMgr.create(this._model, this._api);

processData.call(this, ecModel, api);

stackSeriesData.call(this, ecModel);

coordSysMgr.update(ecModel, api);

doVisualEncoding.call(this, ecModel, payload);

doRender.call(this, ecModel, payload);
```
发现执行过了`doRender.call(this, ecModel, payload);`之后storage._roots发生实质性变化，于是打算再深入调查。有点令人不安的是这排成一排的6个方法，前5个是干什么的？

现在能知道，倒数第2个，也就是`doVisualEncoding.call(this, ecModel, payload);`会给ecModel新加两个属性：`ec_colorIdx:2`和`ec_colorNameMap:{平均温度:"#2f4554",蒸发量:"#c23531"}`。

不过现在管不了这么多了，先试试看只关注`doRender.call(this, ecModel, payload);`能不能得到想要的结果，在`doRender.call(this, ecModel, payload);`里面：

```javascript
function doRender(ecModel, payload) {
  var api = this._api;
  // Render all components
  each(this._componentsViews, function(componentView) {
    var componentModel = componentView.__model;
    componentView.render(componentModel, ecModel, api, payload);

    updateZ(componentModel, componentView);
  }, this);

  each(this._chartsViews, function(chart) {
    chart.__alive = false;
  }, this);

  // Render all charts
  ecModel.eachSeries(function(seriesModel, idx) {
    var chartView = this._chartsMap[seriesModel.__viewId];
    chartView.__alive = true;
    chartView.render(seriesModel, ecModel, api, payload);

    chartView.group.silent = !!seriesModel.get('silent');

    updateZ(seriesModel, chartView);

    updateProgressiveAndBlend(seriesModel, chartView);

  }, this);

  // If use hover layer
  updateHoverLayerStatus(this._zr, ecModel);

  // Remove groups of unrendered charts
  each(this._chartsViews, function(chart) {
    if (!chart.__alive) {
      chart.remove(ecModel, api);
    }
  }, this);
}
```

```javascript
 type: 'cartesianAxis',

   axisPointerClass: 'CartesianAxisPointer',

   /**
    * @override
    */
   render: function(axisModel, ecModel, api, payload) {

     this.group.removeAll();

     var oldAxisGroup = this._axisGroup;
     this._axisGroup = new graphic.Group();

     this.group.add(this._axisGroup);

     if (!axisModel.get('show')) {
       return;
     }

     var gridModel = axisModel.getCoordSysModel();

     var layout = cartesianAxisHelper.layout(gridModel, axisModel); //layout接收两个参数

     var axisBuilder = new AxisBuilder(axisModel, layout); //y轴各个零件在这里面构建的，接受两个参数，this._model和layout

     zrUtil.each(axisBuilderAttrs, axisBuilder.add, axisBuilder);

     this._axisGroup.add(axisBuilder.getGroup()); //这里add进去的就是y轴的细节，可见构建y轴是在axisBuilder里完成的。

     zrUtil.each(selfBuilderAttrs, function(name) {
       if (axisModel.get(name + '.show')) {
         this['_' + name](axisModel, gridModel, layout.labelInterval);
       }
     }, this);

     graphic.groupTransition(oldAxisGroup, this._axisGroup, axisModel);

     CartesianAxisView.superCall(this, 'render', axisModel, ecModel, api, payload);
   },

```

`componentView.render(componentModel, ecModel, api, payload);`

`return helper.intervalScaleGetTicks(this._interval, this._extent, this._niceExtent, this._intervalPrecision);`
 为什么`this._interval`是5，为什么`this._extent`是[0, 30],为什么`this._niceExtent`是[0,25]，为什么用了`this._niceExtent`?

```javascript
axisLine: function() {
  var opt = this.opt;
  var axisModel = this.axisModel;

  if (!axisModel.get('axisLine.show')) {
    return;
  }

  var extent = this.axisModel.axis.getExtent();

  var matrix = this._transform;
  var pt1 = [extent[0], 0];
  var pt2 = [extent[1], 0];
  if (matrix) {
    v2ApplyTransform(pt1, pt1, matrix);
    v2ApplyTransform(pt2, pt2, matrix);
  }

  this.group.add(new graphic.Line(graphic.subPixelOptimizeLine({

    // Id for animation
    anid: 'line',

    shape: {
      x1: pt1[0],
      y1: pt1[1],
      x2: pt2[0],
      y2: pt2[1]
    },
    style: zrUtil.extend({
      lineCap: 'round'
    }, axisModel.getModel('axisLine.lineStyle').getLineStyle()),
    strokeContainThreshold: opt.strokeContainThreshold || 5,
    silent: true,
    z2: 1
  })));
}

```
这是建造坐标轴线的函数，在我们这个例子中，x轴坐标轴线的位置是需要改变的。在这里面，pt1和pt2经过了变换：

```javascript
if (matrix) {
      v2ApplyTransform(pt1, pt1, matrix);
      v2ApplyTransform(pt2, pt2, matrix);
}
```
pt1从[0, 0]变成了[100, 643],pt2从[800, 0]变成了[900, 643]，这个matrix变量是：

```javascript
{
  0: 1,
  1: 0,
  2: 0,
  3: 1,
  4: 100,
  5: 642.8571166992188
}
```
可以看出matrix[4]和matrix[5]应该是和变换有关的。那么matrix1-4都有什么作用呢？也许是矩阵变换吧，目前暂时不知道。
Anyway，现在知道了，影响x轴画法的变量，是Echarts._componentsViews[0].__model.axis._extend和axisBuilder._transform, 其中axisBuilder是var axisBuilder = new AxisBuilder(axisModel, layout);弄出来的。

取到var grid = gridModel.coordinateSystem;，然后从grid中取到rect，这是画图区域的矩形，它有x起点有y起点，有width有height。

发现在helper.layout中，var axisPosition = axis.onZero ? 'onZero' : rawAxisPosition;是受axis.onZero影响的。尝试一下改axis.onZero会不会好点,也就是Echarts._componentsViews[5].__model.axis.onZero


改写gridProto.update方法就好了！！！！！！
```javascript
each(axesMap.x, function(xAxis) {
  // onZero can not be enabled in these two situations
  // 1. When any other axis is a category axis
  // 2. When any other axis not across 0 point
  if (ifAxisCanNotOnZero('y')) {
    xAxis.onZero = false;
  }
});
```
那么，怎么去改写呢
Echarts._componentsViews[0].__model.axis.scale
```
