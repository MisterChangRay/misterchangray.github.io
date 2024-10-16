---
layout: post
title:  "element Table表格列属性绑定关系分析"
date:   2024-10-16 13:29:20 +0800
categories:
      - 逆向
tags:
      - vue
      - elementui
---

最近有个需求，老板叫采集一个管理后台的表单内容到excel,原系统没有设计导出功能。于是只能爬接口了。

爬接口很简单,F12打开浏览器调试面板，点击查询，就可以看到请求表格的内容，但是响应值那么多数据，如何知道UI界面上的列和属性的映射关系呢？

这里说个简单可用的方法，针对于VUE2可用。

第一步，找到对应table渲染根元素。一般根元素大概特征如下：
1. div标签
2. 有一个data-v属性
3. class 包含
```html
<div data-v-4c7dc3fa="" class="el-table el-table--fit el-table--striped el-table--border el-table--enable-row-hover el-table--enable-row-transition el-table--medium" style="height: 100%;">
```

大概布局就像下图了：
![image](https://github.com/user-attachments/assets/c768683c-229c-4337-b69f-5b21a39bfa7b)


接下来，通过xpath访问这个table实例(修改class为实际的,然后再console执行): 
```js
a = $x("//div[@class='el-table el-table--fit el-table--striped el-table--border el-table--enable-row-hover el-table--enable-row-transition el-table--medium']")[0].__vue__
```

这里把table实例抓出来赋予了a, 可以自己看a对象的结构去拿数据。

在实例中遍历所有colum列，并打印结果：
```js
a.$children.forEach(item => {console.log(item._props.label, item._props.prop)})
```
![image](https://github.com/user-attachments/assets/577ec837-61e5-43b6-9320-91d15fe71c14)

这样就可以拿到所有列和属性的绑定关系了。
