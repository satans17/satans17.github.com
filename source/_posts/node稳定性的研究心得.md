title: Node稳定性的研究心得
date: 2014-05-04 11:06:14
tags: Node.js
---

目前大部分Web服务器，如Apache，都使用多线程的方式响应多用户请求，即一个线程服务一个用户请求。这种模式其中一个好处是，当某个请求的线程上抛出的异常没被捕获，只会影响当前这个线程，不会影响其他请求。

由于Node执行在单线程上，一旦线程上抛的异常没有被捕获，就会引起整个进程的崩溃。所以对Node服务的异常处理、保证进程的稳定性非常重要。

再好的程序员也不敢保证自己的代码不出现异常，除了尽可能捕获可能出现的异常，我们还需要通过一些规范减少异常发生，通过单元测试辅助我们验证代码，通过一些工具保证服务的稳定性。下面我从这几个方面探讨如何保证Node的稳定性。



## 一 异常捕获

提升稳定性最直接的方式就是尽可能的捕捉异常，Node提供3种方式。



### 1.1 try/catch

在大多数语言中，try/catch是捕获异常的好手，能确保我们的代码不进入不可控流程。但是由于Node回调/异步的特性，我们无法通过try/catch来捕捉所有的异常，看下面的示例：

```js
try {
    process.nextTick(function () {
        throw new Error("error");
    });
} catch (err) {
    //can not catch it
    console.log(err);
}
```

```js
try {
    setTimeout(function(){
        throw new Error("error");
    },1)
} catch (err) {
    //can not catch it
    console.log(err);
}
```

上面的代码没有像预期的那样帮我们捕获异常，后果就是这个未被捕获的异常导致整个Node进程crash，所以Node中try/catch的方式不是那么管用。
如果有一种方法能帮我们全局捕获异常，Node服务就不会轻易挂掉了。Node确实提供了这种方式，不过却不能完全满足我们的需求。


### 1.2 uncaughtException

当一个异常未被捕获，冒泡回归到事件循环中就会触发uncaughtException事件。

```js
process.on('uncaughtException', function(err) {
    console.error('Error caught in uncaughtException event:', err);
});


try {
    setTimeout(function(){
        throw new Error("error");
    },1)
} catch (err) {
    //can not catch it
    console.log(err);
}
```

只要给uncaughtException配置了回调，Node进程不会异常退出，但异常发生的上下文已经丢失，我们无法给出友好的返回，比如告诉用户哪里出问题了。而且由于uncaughtException事件发生后，会丢失当前环境的堆栈，可能导致Node不能正常进行内存回收，从而导致内存泄露。所以，uncaughtException的正确使用姿势是，当uncaughtException触发，记录error日志，然后结束Node进程，我们通过日志监控，报警及时解决异常。

```js
process.on('uncaughtException', function(err) {
    // 记录日志
    logger(err);
    // 结束进程
    process.exit(1);
});
```


### 1.3 domain

为了弥补try/catch和uncaughtException的不足，在node v0.8+版本的时候，发布了一个模块`domain`，这个模块能捕捉异步回调中出现的异常。
看下面的示例：


```js
var d = domain.create();

process.on('uncaughtException', function(err) {
    console.error(err);
});

d.on('error', function(err) {
    console.error('Error caught by domain:', err);
});

d.run(function() {
    process.nextTick(function() {
        throw new Error("test domain");
    });
});
```

运行代码我们会发现，异常会被domain捕获到，uncaughtException不会被触发。虽然我们对domain模块寄予厚望，不过目前domain模块的评级为“Unstable”，因为存在不少性能以及稳定性问题。
关于domain的详细介绍，可以看看以下几篇文章：

- [Node.js 异步异常的处理与domain模块解析](http://cnodejs.org/topic/516b64596d38277306407936)
- [Node.js异常捕获额一些实践](http://www.alloyteam.com/2013/12/node-js-series-exception-caught/)
- [Node.js之异常处理](http://www.atatech.org/articles/16061)



## 二 阻止异常发生

我们当然无法阻止异常的发生，这里说的阻止异常发生，是希望大家能养成良好的编码习惯，严谨的思维逻辑，来尽可能减少代码抛出未捕获异常。


### 2.1 良好的异常处理习惯


-  异步API编写规范：由于异步调用中回调函数里的异常无法被外部捕获，所以我们将API内部发生的异常作为第一个参数传递给回调函数，包括NodeJS官方的API是遵循这个规范的。

```js
fs.readFile('/t.txt', function (err, data) {
  if (err) throw err;
  console.log(data);
});
```

```js
// 不推荐的做法
function fun(options,callback){
    if(!options{
        throw new Error("..")
    }
}

// 推荐的做法
function fun(options,callback){
    if(!options){
        callback(err,null)
    }
}

```



-  严格校验用户的输入

```js
function doSth(cb){
    if(typeof cb === 'function'){
        cb();
    }
}
```


-  使用try/catch处理可能出现异常的代码

```js
var obj;
try{
    obj = JSON.parse('')
}catch(e){
    obj = {};
}
```


-  不要直接在controller中抛异常，应该用500等状态更友好的返回错误


```js
// 不推荐的做法
app.get('/item.html', function (req, res, next) {
    if(!query["id"]){
        throw new Error('no item id');
    }
});

```


```js
// 推荐的做法
app.get('/item.html', function (req, res, next) {
    if(!query["id"]){
        next();
    }
});

```




### 2.2 单元测试

单元测试的重要性想必所有人都清楚，JS代码跑在浏览器端的时候我们未必会做，因为异常通常只会影响部分人使用的部分功能，不足以引起很多人的重视。JS代码跑在服务端情况完全不一样了，一旦出现异常，影响的是所有的用户，所以单元测试就显得非常重要。

至于如何在NodeJS中写单元测试，可以看看两位大神的分享：

- 苏千:[UnitTest in Nodejs 实战Nodejs单元测试](http://fengmk2.cnpmjs.org/ppt/unittest-and-bdd-in-nodejs-with-mocha.html#slide-0)
- 朴灵:[NODE.JS UNIT TESTING](http://html5ify.com/unittesting/slides/index.html#/)


### 2.3 记录日志

还是那句说，谁也不敢保证自己的代码不出现异常，因为有运行环境，网络环境等各种不稳定因素，所以建立健全的排查和跟踪机制就显得很重要，而日志就是实现这种机制的关键。
阿里线上环境已经有完善的日志监控体现，我们要做的就是去学会如何使用他。

ali-logger:[http://search.npm.taobao.net/package/ali-logger](http://search.npm.taobao.net/package/ali-logger)







## 三 多进程架构

### 3.1 cluster

cluster模块用于创建共享端口的多进程模式,这种模式使多个进程间共享一个监听状态的socket，并由系统将accept的connection分配给不同的子进程。
文档上有一个简单的示例：

```js
var cluster = require('cluster');
var http = require('http');
var numCPUs = require('os').cpus().length;

if (cluster.isMaster) {
  // Fork workers.
  for (var i = 0; i < numCPUs; i++) {
    cluster.fork();
  }

  cluster.on('exit', function(worker, code, signal) {
    console.log('worker ' + worker.process.pid + ' died');
  });
} else {
  // Workers can share any TCP connection
  // In this case its a HTTP server
  http.createServer(function(req, res) {
    res.writeHead(200);
    res.end("hello world\n");
  }).listen(8000);
}
```

利用`cluster`，我们可以根据CPU的数量创建多个worker进程，用户的请求会被分配到不同的进程上，如果某个进程出现异常，可以直接将这个进程crash掉，而不会影响其他进程。
关于`cluster`实现原理的介绍文章非常多，想深入了解的可以自行搜索一下。


#### 3.1.1 graceful + recluster

数据产品中使用graceful + recluster两个模块实现多进程和服务器稳定性的工作，分享一下使用方法：

-  [graceful](https://www.npmjs.org/package/graceful) : 当uncaughtException触发后，延迟一段时间退出进程
-  [recluster](https://www.npmjs.org/package/recluster) : 实现多进程，并且当有进程退出实现自动重启。


**app.js**

```js
var app = require('../app');
var graceful = require('graceful');

var server = app.listen(app.get('port'), function() {
    debug('Express server listening on port ' + server.address().port);
});

graceful({
    server: server,
    killTimeout: 30000,
    error:function(e){
        logger.error(e);
    }
});
```

**server.js**

```js
var recluster = require('recluster'),
    path = require('path');

var cluster = recluster(path.join(__dirname, 'app.js'));
cluster.run();

process.on('SIGUSR2', function() {
    console.log('Got SIGUSR2, reloading cluster...');
    cluster.reload();
});

console.log("spawned cluster, kill -s SIGUSR2", process.pid, "to reload");
```


开发环境,直接通过app启动:
```sh
$ node app.js
```


生产环境, 启动多个 worker:

```sh
$ node server.js
```








#### 3.1.2 pm2

如果你觉得graceful + recluster的方式太复杂，那么[pm2](https://github.com/Unitech/pm2)肯定是你最理想的选择。
pm2非常强大，生产环境使用pm2启动你的Node服务是个不错的选择，他能自动利用你的多核cup，完善的监控，日志记录等等..

pm2使用非常简单：

```sh
$ npm install pm2@latest -g
```

```sh
$ pm2 start app.js
```

```sh
$ pm2 list
```


![pm2](https://github.com/unitech/pm2/raw/master/pres/pm2-list.png)



如果你想更多的了解pm2，可以直接看pm2的文档:https://github.com/Unitech/pm2

不过正是由于pm2功能太过强大，我们没有选择pm2，因为他太复杂了，感觉还没有能力驾驭他，特别是万一出现问题，我们还不知道如何解决。也许我们的顾虑是多余的，不过需要一点时间去了解他。





## 小结

刚开始学习Node的使用，对Node确实有点小担心，因为使用别的语言做Web服务，根本不用担心因为一个错误导致整个服务crash的问题。
随着对Node的了解，我掌握了一些技巧，也打消了一些顾虑。
不过毕竟Node应用经验有限，所以欢迎大家一起探讨，积累更多宝贵的经验。







