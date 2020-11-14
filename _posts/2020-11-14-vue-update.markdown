---
layout: post
title: 'ant design vue pro记录'
subtitle:   "vue component record"
date:       2020-11-14 21:30:00
author:     "Caesar"
header-mask: 0.3
catalog:    true
tags:
    - Vue
---
项目中使用ant-design-vue-pro做前端，经常会遇到一些问题。这篇文章，是关于其中遇到问题及解决方法。

### 组件更新
在s-table的查询表格(TableList)，操作列中加点击事件，当点击的时候，加载model框，显示一个子table详情信息。
但是当点击详情时，第一次加载后，后面的数据与第一次的完全相同。
为了使自定义组件DetailTable可以每次刷新，在组件中定义key
```vue
<detail-table
    :loading="loading"
    :xdata="xdata"
    :visible="visible"
    :key="time"
    :@cancel="handleCancel">
</detail-table>
```
在methods中定义一个修改key的方法
```javascript
 handleLoad: function () { 
    this.time = new Date().getTime()
  }
```
将**handleLoad** 加入到点击事件中，每次点击时传递值不同，进行加载，从而更新数据。

### s-table loadData
s-table中 :data 传入返回Promise对象的方法。
构造返回promise的方法
```
 new Promise( function(resolve) {
    resolve(some_data)
 }).then( (res) => {
    return some_data
 })

```
