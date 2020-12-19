---
title: 一些提高代码健壮性的方法
date: 2020-12-18 16:54:08
tags:
- 健壮性
categories:
- 编程
---

>转载自 [橙红年代](https://juejin.cn/post/6896118234391511053#heading-2)

在过去的开发经历中处理了各种奇葩BUG，认识到代码健壮性（鲁棒性）是衡量工作效率、生活质量一个非常重要的标准，本文主要整理了提高代码健壮性的一些思考。
## 1. 更安全地访问对象
###1.1. 不要相信接口数据
不要相信前端传的参数，也不要信任后台返回的数据

比如某个api/xxx/list的接口，按照文档的约定
```javascript
{
    code: 0,
    msg: "",
    data: [
     // ... 具体数据
    ],
};
```
前端代码可能就会写成
```
const {code, msg, data} = await fetchList()
data.forEach(()=>{})

```
因为我们假设了后台返回的`data`是一个数组，所以直接使用了`data.forEach`，如果在联调的时候遗漏了一些异常情况

- 预期在没有数据时`data`会返回[]空数组，但后台的实现却是不返回`data`字段

- 后续接口更新，`data`从数组变成了一个字典，跟前端同步不及时

这些时候，使用`data.forEach`时就会报错，

```
Uncaught TypeError: data.forEach is not a function
```

所以在这些直接使用后台接口返回值的地方，最好添加类型检测
```
Array.isArray(data) && data.forEach(()=>{})
```
同理，后台在处理前端请求参数时，也应当进行相关的类型检测。

### 1.2. 空值合并运算符

由于JavaScript动态特性，我们在查询对象某个属性时如`x.y.z`，最好检测一下`x`和`y`是否存在
```
let z = x && x.y && x.y.z
```
经常这么写就显得十分麻烦，dart中安全访问对象属性就简单得多
```
var z = a?.y?.z;
```
在ES2020中提出了空值合并运算符的草案，包括`??`和`?.`运算符，可以实现与dart相同的安全访问对象属性的功能。目前打开最新版Chrome就可以进行测试了

![安全访问对象属性](http://img.shymean.com/oPic/1585982625374_463.png "安全访问对象属性")

在此之前，我们可以封装一个安全获取对象属性的方法
```
function getObjectValueByKeyStr(obj, key, defaultVal = undefined) {
    if (!key) return defaultVal;
    let namespace = key.toString().split(".");
    let value,
        i = 0,
        len = namespace.length;
    for (; i < len; i++) {
        value = obj[namespace[i]];
        if (value === undefined || value === null) return defaultVal;
        obj = value;
    }
    return value;
}
var x = { y: { z: 100,},};

var val = getObjectValueByKeyStr(x, "y.z");
// var val = getObjectValueByKeyStr(x, "zz");
console.log(val);
```
前端不可避免地要跟各种各种浏览器、各种设备打交道，一个非常重要的问题就是兼容性，尤其是目前我们已经习惯了使用ES2015的特性来开发代码，`polyfill`可以帮助解决我们大部分问题。

## 2. 记得异常处理

参考：

- [JS错误处理 MDN](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Guide/Control_flow_and_error_handling)
- [js构建ui的统一异常处理方案](https://www.cnblogs.com/laden666666/p/5281993.html)，这个系列的文章写得非常好

异常处理是代码健壮性的首要保障，关于异常处理有两个方面

- 合适的错误处理可以提要用户体验，在代码出错时优雅地提示用户
- 将错误处理进行封装，可以减少开发量，将错误处理与代码解耦

### 2.1. 错误对象

可以通过throw语句抛出一个自定义错误对象
```
// Create an object type UserException
function UserException (message){
  // 包含message和name两个属性
  this.message=message;
  this.name="UserException";
}
// 覆盖默认[object Object]的toString
UserException.prototype.toString = function (){
  return this.name + ': "' + this.message + '"';
}

// 抛出自定义错误
function f(){
    try {
        throw new UserException("Value too high");
    }catch(e){
        if(e instanceof UserException){
            console.log('catch UserException')
            console.log(e)
        }else{
            console.log('unknown error')
            throw e
        }
    }finally{
        // 可以做一些退出操作，如关闭文件、关闭loading等状态重置
        console.log('done')
        return 1000 // 如果finally中return了值，那么会覆盖前面try或catch中的返回值或异常
    }
}
f()
```
### 2.2. 同步代码

对于同步代码，可以使用通过责任链模式封装错误，即当前函数如果可以处理错误，则在catch中进行处理：如果不能处理对应错误，则重新将catch抛到上一层
```
function a(){
    throw 'error b'
}

// 当b能够处理异常时，则不再向上抛出
function b(){
    try{
        a()
    }catch(e){
        if(e === 'error b'){
            console.log('由b处理')
        }else {
            throw e
        }
    }
}

function main(){
    try {
        b()
    }catch(e){
        console.log('顶层catch')
    }
}
### 2.3. 异步代码
由于catch无法获取异步代码中抛出的异常，为了实现责任链，需要把异常处理通过回调函数的方式传递给异步任务

function a(errorHandler) {
    let error = new Error("error a");
    if (errorHandler) {
        errorHandler(error);
    } else {
        throw error;
    }
}

function b(errorHandler) {
    let handler = e => {
        if (e === "error b") {
            console.log("由b处理");
        } else {
            errorHandler(e);
        }
    };

    setTimeout(() => {
        a(handler);
    });
}

let globalHandler = e => {
    console.log(e);
};
b(globalHandler);
```

### 2.4. Prmise的异常处理
Promise只包含三种状态：`pending`、`rejected`和`fulfilled`
```
let promise2 = promise1.then(onFulfilled, onRejected)
```
下面是`promise`抛出异常的几条规则
```
function case1(){
    // 如果promise1是rejected态的，但是onRejected返回了一个值（包括undifined），那么promise2还是fulfilled态的，这个过程相当于catch到异常，并将它处理掉，所以不需要向上抛出。
    var p1 = new Promise((resolve, reject)=>{
        throw 'p1 error'
    })

    p1.then((res)=>{
        return 1
    }, (e)=>{
        console.log(e)
        return 2
    }).then((a)=>{
        // 如果注册了onReject，则不会影响后面Promise执行
        console.log(a) // 收到的是2
    })
}
function case2(){
    //  在promise1的onRejected中处理了p1的异常，但是又抛出了一个新异常，，那么promise2的onRejected会抛出这个异常
    var p1 = new Promise((resolve, reject)=>{
        throw 'p1 error'
    })
    p1.then((res)=>{
        return 1
    }, (e)=>{
        console.log(e)
        throw 'error in p1 onReject'
    }).then((a)=>{}, (e)=>{
        // 如果p1的 onReject 抛出了异常
        console.log(e)
    })
}

function case3(){
    // 如果promise1是rejected态的，并且没有定义onRejected，则promise2也会是rejected态的。
    var p1 = new Promise((resolve, reject)=>{
        throw 'p1 error'
    })

    p1.then((res)=>{
        return 1
    }).then((a)=>{
        console.log('not run:', a)
    }, (e)=>{
        // 如果p1的 onReject 抛出了异常
        console.log('handle p2:', e)
    })
}
function case4(){
    // // 如果promise1是fulfilled态但是onFulfilled和onRejected出现了异常，promise2也会是rejected态的，并且会获得promise1的被拒绝原因或异常。
    var p1 = new Promise((resolve, reject)=>{
        resolve(1)
    })
    p1.then((res)=>{
        console.log(res)
        throw 'p1 onFull error'
    }).then(()=>{}, (e)=>{
        console.log('handle p2:', e)
        return 123
    })
}
```
因此，我们可以在`onRejected`中处理当前`promise`的错误，如果不能，就把他抛给下一个`promise`

### 2.5. async
`async/await`本质上是promise的语法糖，因此也可以使用`promise.catch`类似的捕获机制
```
function sleep(cb, cb2 =()=>{},ms = 100) {
    cb2()
    return new Promise((resolve, reject) => {
        setTimeout(() => {
            try {
                cb();
                resolve();
            }catch(e){
                reject(e)
            }
        }, ms);
    });
}
// 通过promise.catch来捕获
async function case1() {
    await sleep(() => {
        throw "sleep reject error";
    }).catch(e => {
        console.log(e);
    });
}
// 通过try...catch捕获
async function case2() {
    try {
        await sleep(() => {
            throw "sleep reject error";
        })
    } catch (e) {
        console.log("catch:", e);
    }
}
// 如果是未被reject抛出的错误，则无法被捕获
async function case3() {
    try {
        await sleep(()=>{}, () => {
            // 抛出一个未被promise reject的错误
            throw 'no reject error'
        }).catch((e)=>{
            console.log('cannot catch:', e)
        })
    } catch (e) {
        console.log("catch:", e);
    }
}
```

## 3. 更稳定的第三方模块
在实现一些比较小功能的时候，比如日期格式化等，我们可能并不习惯从npm找一个成熟的库，而是自己顺手写一个功能包，由于开发时间或者测试用例不足，当遇见一些未考虑的边界条件，就容易出现BUG。

这也是npm上往往会出现一些很小的模块，比如这个判断是否为奇数的包：`isOdd`，周下载量居然是60来万。

![isOdd](http://img.shymean.com/oPic/1585919200512_881.png "isOdd")

使用一些比较成熟的库，一个很重要原因是，这些库往往经过了大量的测试用例和社区的考验，肯定比我们顺手些的工具代码更安全。

一个亲身经历的例子是：根据UA判断用户当前访问设备，正常思路是通过正则进行匹配，当时为了省事就自己写了一个
```
export function getOSType() {
  const ua = navigator.userAgent

  const isWindowsPhone = /(?:Windows Phone)/.test(ua)
  const isSymbian = /(?:SymbianOS)/.test(ua) || isWindowsPhone
  const isAndroid = /(?:Android)/.test(ua)

  // 判断是否是平板
  const isTablet =
    /(?:iPad|PlayBook)/.test(ua) ||
    (isAndroid && !/(?:Mobile)/.test(ua)) ||
    (/(?:Firefox)/.test(ua) && /(?:Tablet)/.test(ua))

  // 是否是iphone
  const isIPhone = /(?:iPhone)/.test(ua) && !isTablet

  // 是否是pc
  const isPc = !isIPhone && !isAndroid && !isSymbian && !isTablet
  return {
    isIPhone,
    isAndroid,
    isSymbian,
    isTablet,
    isPc
  }
}
```
上线后发现某些小米平板用户的逻辑判断出现异常，调日志看见UA为
```
Mozilla/5.0 (Linux; U; Android 8.1.0; zh-CN; MI PAD 4 Build/OPM1.171019.019) AppleWebKit/537.36 (KHTML, like Gecko) Version/4.0 Chrome/57.0.2987.108 Quark/3.8.5.129 Mobile Safari/537.36
```
即使把MI PAD添加到正则判断中临时修复一下，万一后面又出现其他设备的特殊UA呢？所以，凭借自己经验写的很难把所有问题都考虑到，后面替换成 [mobile-detect](https://www.npmjs.com/package/mobile-detect) 这个库。

使用模块的缺点在于

可能会增加文件依赖体积，增加打包时间等，这个问题可以通过打包配置解决，将不会经常变更的第三方模块打包到`vendor`文件中配置缓存
在某些项目可能会由于安全考虑需要减少第三方模块的使用，或者要求先进行源码`code review`
当然在进行模块选择的时候也要进行各种考虑，包括稳定性、旧版本兼容、未解决`issue`等问题。当选择了一个比较好的工具模块之后，我们就可以将更多的精力放在业务逻辑中。

## 4. 本地配置文件
在开发环境下，我们可能需要一些本地的开关配置文件，这些配置只在本地开发时存在，不进入代码库，也不会跟其他同事的配置起冲突。

我推崇将mock模板托管到git仓库中，这样可以方便其他同事开发和调试接口，带来的一个问题时本地可能需要一个引入mock文件的开关

下面是一个常见的做法：新建一个本地的配置文件config.local.js，然后导出相关配置信息
```
// config.local.js
module.exports = {
  needMock: true
}
```
记得在`.gitignore`中忽略该文件
```
config.local.js
```
然后通过`try...catch...`加载该模块，由于文件未进入代码库，在其他地方拉代码更新时会进入`catch`流程，本地开发则进入正常模块引入流程
```
// mock/entry.js
try {
  const { needMock } = require('./config.local')
  if (needMock) {
    require('./index') // 对应的mock入口
    console.log('====start mock api===')
  }
} catch (e) {
  console.log('未引入mock，如需要，请创建/mock/config.local并导出 {needMock: true}')
}
```
最后在整个应用的入口文件判断开发环境并引入
```
if (process.env.NODE_ENV === 'development') {
  require('../mock/entry')
}
```
通过这种方式，就可以在本地开发时愉快地进行各种配置，而不必担心忘记在提交代码前注释对应的配置修改~

## 5. Code Review
参考：

- [Code Review 是苦涩但有意思的修行](https://mp.weixin.qq.com/s/xoekgWhWDK6GioKq0OFI1A)

`Code Review`应该是是上线前一个必经的步骤，我认为CR主要的作用有

- 能够确认需求理解是否出现偏差，避免扯皮

- 优化代码质量，包括冗余代码、变量命名和过分封装等，起码除了写代码的人之外还得保证审核的人能看懂相关逻辑

对于一个需要长期维护迭代的项目而言，每一次`commit`和`merge`都是至关重要的，因此在合并代码之前，最好从头检查一遍改动的代码。即使是在比较小的团队或者找不到审核人员，也要把合并认真对待。

6. 小结
本文主要整理了提高`JavaScript`代码健壮性的一些方法，主要整理了

- 安全地访问对象属性，避免数据异常导致代码报错
- 捕获异常，通过责任链的方式进行异常处理或上报
- 使用更稳定更安全的第三方模块，
- 认真对待每一次合并，上线前先检查代码

此外，还需要要养成良好的编程习惯，尽可能考虑各种边界情况。
