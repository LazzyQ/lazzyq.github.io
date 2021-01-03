---
title: "exports、module.exports和export、export default到底是咋回事"
date: 2021-01-03T14:17:04+08:00
draft: false
original: false
categories: 
  - fronted
tags: 
  - fronted
---

学习学习前端的知识，转载学习一下[原文](https://segmentfault.com/a/1190000010426778)

### 前言

难得有空，今天开始重新规范的学习一下node编程。
但是引入模块我看到用 require的方式，再联想到咱们的ES6各种export 、export default。

阿西吧，头都大了....

头大完了，那我们坐下先理理他们的使用范围。

> require: node 和 es6 都支持的引入
> 
> export / import : 只有es6 支持的导出引入
> 
> module.exports / exports: 只有 node 支持的导出

这一刻起，我觉得是时候要把它们之间的关系都给捋清楚了，不然我得混乱死。话不多少，咱们开干！！

<!--more-->

### node模块

Node里面的模块系统遵循的是CommonJS规范。
那问题又来了，什么是CommonJS规范呢？
由于js以前比较混乱，各写各的代码，没有一个模块的概念，而这个规范出来其实就是对模块的一个定义。

> CommonJS定义的模块分为: 模块标识(module)、模块定义(exports) 、模块引用(require)

先解释 exports 和 module.exports
在一个node执行一个文件时，会给这个文件内生成一个 exports和module对象，
而module又有一个exports属性。他们之间的关系如下图，都指向一块{}内存区域。

![exports、module.exports和export、export default到底是咋回事](/exports等/1.png)

那下面我们来看看代码的吧。

```js
//utils.js
let a = 100;

console.log(module.exports); //能打印出结果为：{}
console.log(exports); //能打印出结果为：{}

exports.a = 200; //这里辛苦劳作帮 module.exports 的内容给改成 {a : 200}

exports = '指向其他内存区'; //这里把exports的指向指走

//test.js

var a = require('/utils');
console.log(a) // 打印为 {a : 200} 
```

> 从上面可以看出，其实require导出的内容是module.exports的指向的内存块内容，并不是exports的。
>
> 简而言之，区分他们之间的区别就是 exports 只是 module.exports的引用，辅助后者添加内容用的。

用白话讲就是，exports只辅助module.exports操作内存中的数据，辛辛苦苦各种操作数据完，累得要死，结果到最后真正被require出去的内容还是module.exports的，真是好苦逼啊。

其实大家用内存块的概念去理解，就会很清楚了。

然后呢，为了避免糊涂，尽量都用 module.exports 导出，然后用require导入。

### ES中的模块导出导入

说实话，在es中的模块，就非常清晰了。不过也有一些细节的东西需要搞清楚。
比如 `export` 和 `export default`，还有 导入的时候，`import a from ..`，`import {a} from ..`，总之也有点乱，那么下面我们就开始把它们捋清楚吧。

**export 和 export default**

首先我们讲这两个导出，下面我们讲讲它们的区别

1. export与export default均可用于导出常量、函数、文件、模块等
2. 在一个文件或模块中，export、import可以有多个，export default仅有一个
3. 通过export方式导出，在导入时要加{ }，export default则不需要
4. export能直接导出变量表达式，export default不行。

下面咱们看看代码去验证一下

testEs6Export.js

```js
'use strict'
//导出变量
export const a = '100';  

 //导出方法
export const dogSay = function(){ 
    console.log('wang wang');
}

 //导出方法第二种
function catSay(){
   console.log('miao miao'); 
}
export { catSay };

//export default导出
const m = 100;
export default m; 
//export defult const m = 100;// 这里不能写这种格式。
```

index.js

```js
//index.js
'use strict'
var express = require('express');
var router = express.Router();

import { dogSay, catSay } from './testEs6Export'; //导出了 export 方法 
import m from './testEs6Export';  //导出了 export default 

import * as testModule from './testEs6Export'; //as 集合成对象导出

/* GET home page. */
router.get('/', function(req, res, next) {
  dogSay();
  catSay();
  console.log(m);
  testModule.dogSay();
  console.log(testModule.m); // undefined , 因为  as 导出是 把 零散的 export 聚集在一起作为一个对象，而export default 是导出为 default属性。
  console.log(testModule.default); // 100
  res.send('恭喜你，成功验证');
});

module.exports = router;
```

从上面可以看出，确实感觉ES6的模块系统非常灵活的。