---
title: native-echarts的问题汇总
date: 2018-04-16 13:39:17
tags: [js, react-native, echarts]
categories: 问题集
---

公司最近用react-native来开发一个app。在app里需要显示图表，我们选择了native-echarts这个库。这个库实际上是封装了一个webview，在webview里对echarts代码进行调用。应该说这个库坑还是不少的，在这里记录一下。

<!-- more -->
---

# 库的简介：

npm：https://www.npmjs.com/package/native-echarts

github：https://github.com/somonus/react-native-echarts

# 换行的问题

这个问题是我进行调查并且发现的，所以可能会说得多一些。

## 描述：

在echarts的option中，需要显示各种文字，legend，label，tooltips等等。为了适应大家对文字样式的需求，echarts提供了一个formatter给大家做一些文字的变换：

```javascript
formatter:function(val){
    return val.split("").join("\n");
}
```

但是在`native-echarts`这个库中，写了上面的`formatter`，`\n`换行并不能生效。其他的替代方法（详见该库github上的issue，搜索"\n"）如`\\n`，`\\nn`也都没有用。

## 调查：

后面发现，问题出在`native-echarts/src/util/toString.js`里（请看注释）：

```javascript
export default function toString(obj) {
  let result = JSON.stringify(obj, function(key, val) {
        if (typeof val === 'function') {
            return `~--demo--~${val}~--demo--~`;
        }
        return val;
    });

    do {
        result = result.replace('\"~--demo--~', '').replace('~--demo--~\"', '').replace(/\\n/g, '').replace(/\\\"/g,"\"");
//while循环体内，多次执行了replace(/\\n/g, ''),导致无论你写多少个\\nn，都会被替换掉
    } while (result.indexOf('~--demo--~') >= 0);
    return result;
}
```

解决方案是在循环体之外，替换一次`replace(/\\n/g, '').replace(/\\\"/g,"\"");`

## 解决方案：

1.用下面的代码替换`native-echarts/src/util/toString.js`:

```javascript
export default function toString(obj) {
  let result = JSON.stringify(obj, function(key, val) {
    if (typeof val === 'function') {
      return `~--demo--~${val}~--demo--~`;
    }
    return val;
  });

  do {
    result = result.replace('\"~--demo--~', '').replace('~--demo--~\"', '');
  } while (result.indexOf('~--demo--~') >= 0);
  result = result.replace(/\\n/g, '').replace(/\\\"/g,"\"");//最后一个replace将release模式中莫名生成的\"转换成"
  return result;
}
```

2.formatter写成这样：

```javascript
formatter: function (name) {
        var maxLength = 40;
        if (name.length > maxLength) {
          return name.slice(0, maxLength) + '\nn' + name.slice(maxLength);
        }
        return name;
      }
```

`\nn`的地方就可以加入一个换行符了。

# 安卓打包后不显示图表

## 描述：

安卓上打包安装到真机后，图表显示不出来，ios则没问题。

## 解决方案：

- 将`native-echarts/src/components/Echarts/index.js`中的代码：`source={require('./tpl.html')}`修改为：
  `source= {Platform.OS === 'ios' ? require('./tpl.html') : { uri: 'file:///android_asset/tpl.html' }}`


- 同时将tpl.html文件拷贝到安卓项目下面的app/src/main/assets文件夹中。

参考：https://segmentfault.com/a/1190000011856689

# 在某些安卓手机上闪烁的问题

## 描述：

在某些手机上发现，图表区域会一黑一白的闪烁。典型机型如：oppo R7c和oppo R8，安卓系统版本都是4.4

## 解决方案：

https://www.jianshu.com/p/6fa9482695bf

## 研究：

为了弄明白这个问题的原理，我仔细对照了修改的部分和原本的代码，感觉比较有关系的是`renderChart.js`的改写。因为它区分了第一次渲染和后续渲染，如果不是第一次的话，就不调用`echarts.init`方法。于是缩小了修改的范围，`renderChart.js`还按照文章里的改，`node_modules/native-echarts/src/components/Echarts/index.js`改成：

```javascript
import React, { Component } from 'react';
import { WebView, View, StyleSheet, Platform } from 'react-native';
import renderChart from './renderChart';
import echarts from './echarts.min';

export default class App extends Component {

  constructor(props) {
    super(props);
  }

  shouldComponentUpdate(nextProps, nextState) {
    const thisProps = this.props || {}
    nextProps = nextProps || {}
    if (Object.keys(thisProps).length !== Object.keys(nextProps).length) {
      return true
    }
    for (const key in nextProps) {
      if (JSON.stringify(thisProps[key]) != JSON.stringify(nextProps[key])) {
// console.log('props', key, thisProps[key], nextProps[key])
        return true
      }
    }
    return false
  }

  componentWillReceiveProps(nextProps) {
    if(nextProps.option !== this.props.option) {

// 解决数据改变时页面闪烁的问题
      this.refs.chart.injectJavaScript(renderChart(nextProps, false))
    }
  }


  render() {
    return (
      <View style={{flex: 1, height: this.props.height || 400,}}>
        <WebView
          ref="chart"
          scrollEnabled = {false}
          injectedJavaScript = {renderChart(this.props, true)}
          style={{
            height: this.props.height || 400,
            backgroundColor: this.props.backgroundColor || 'transparent'
          }}
          scalesPageToFit={false}
          source= {Platform.OS === 'ios' ? require('./tpl.html') : { uri: 'file:///android_asset/tpl.html' }}
        />
      </View>
    );
  }
}
```

装到oppo R7上之后问题依旧。

非常的奇怪啊。这就说明，其实问题不是出在renderChart.js里面，推翻了我之前的设想，设想被推翻是很气人的，我必须得弄明白哪里出问题了。运用控制变量法，打了几个对照实验的包，分别是：

1. 新的renderChart + 旧的index.js + 部分新的index.js
2. 新的renderChart + 新的index.js

1无效，2有效，说明问题不是renderChart，也不是index.js里面的`componentWillReceiveProps`和`shouldComponentUpdate`。那其他的修改就只剩`render`方法里面了。仔细观察了render方法里面异同，发现

```jsx
<View style={{flex: 1, height: this.props.height || 400,}}>
  <WebView
    ref="chart"
    scrollEnabled = {false}
    injectedJavaScript = {renderChart(this.props)}
    style={{
      height: this.props.height || 400,
      backgroundColor: this.props.backgroundColor || 'transparent'// 这是唯一的区别
    }}
    source= {Platform.OS === 'ios' ? require('./tpl.html') : { uri: 'file:///android_asset/tpl.html' }}
    onMessage={event => this.props.onPress ? this.props.onPress(JSON.parse(event.nativeEvent.data)) : null}
  />
</View>
```

唯一的区别是backgroundColor，于是用旧的renderChart + 旧的index.js + 去掉backgroundColor，发现问题解决了。

但最后，为了干掉重复echarts.init可能造成的问题，我还是按照下述进行了修改：

`renderChart.js`:

```jsx
// 修改后
import echarts from './echarts.min';
import toString from '../../util/toString';
var myChart = null;
export default function renderChart(props,isFirst) {
  // const height = props.height || 400;
  const height = `${props.height || 400}px`;
  const width = props.width ? `${props.width}px` : 'auto';
  if (isFirst){
    return `
    document.getElementById('main').style.height = "${height}";
    document.getElementById('main').style.width = "${width}";
    myChart = echarts.init(document.getElementById('main'));
    myChart.setOption(${toString(props.option)});
  `
  }else{
    return `
    document.getElementById('main').style.height = "${height}";
    document.getElementById('main').style.width = "${width}";
    myChart.setOption(${toString(props.option)});
  `
  }
}
```

`index.js`:

```Jsx
// 修改后
import React, { Component } from 'react';
import { WebView, View, StyleSheet, Platform } from 'react-native';
import renderChart from './renderChart';
import renderChartNoFirst from './renderChart'
import echarts from './echarts.min';


export default class App extends Component {
// 预防过渡渲染

  shouldComponentUpdate(nextProps, nextState) {
    const thisProps = this.props || {}
    nextProps = nextProps || {}
    if (Object.keys(thisProps).length !== Object.keys(nextProps).length) {
      return true
    }
    for (const key in nextProps) {
      if (JSON.stringify(thisProps[key]) != JSON.stringify(nextProps[key])) {
// console.log('props', key, thisProps[key], nextProps[key])
        return true
      }
    }
    return false
  }

  componentWillReceiveProps(nextProps) {
    if(nextProps.option !== this.props.option) {

// 解决数据改变时页面闪烁的问题
      this.refs.chart.injectJavaScript(renderChart(nextProps,false))
    }
  }

  render() {
    if (Platform.OS == 'android'){
      return (
        <View style={{flex: 1, height: this.props.height || 400,}}>
          <WebView
            ref="chart"
            scrollEnabled = {false}
            injectedJavaScript = {renderChart(this.props,true)}
            style={{
              height: this.props.height || 400,
            }}
            //source={require('./tpl.html')}
            source={{uri: 'file:///android_asset/tpl.html'}}
          />
        </View>
      );
    }else{
      return (
        <View style={{flex: 1, height: this.props.height || 400,}}>
          <WebView
            ref="chart"
            scrollEnabled = {false}
            scalesPageToFit={false}
            injectedJavaScript = {renderChart(this.props,true)}
            style={{
              height: this.props.height || 400,
            }}
            source= {Platform.OS === 'ios' ? require('./tpl.html') : { uri: 'file:///android_asset/tpl.html' }}
          />
        </View>
      );
    }

  }
}
```

