
# JS异步编程

JS异步编程日趋火热，到底该怎么写异步，变成了一个有趣且不可回避的问题，通常做法是将异步事件放入串并行队列（数组或类数组）中，或者使用可以更好的控制异常捕获的Promise , 或着使用当下最新最前沿的E6 generator,在javascript的生态圈内，很容易找到对应实现的开源类库. **但其实我们在封装函数的时候，稍加注意，以上这些方案也可以不用，比如递归遍历是一个伴随串并行的应用场景，让我们通过下面遍历目录的例子，看看异步编程也可以这么写**：

```javascript
var fs = require('fs');
var path = require('path');

readPath('./Documents/Books');

function readPath(p) {
    fs.stat(p, function(err, stats) {
        if (stats.isFile()) {
            readFile(p, stats);
        } else if (stats.isDirectory()) {
            readDirectory(p, stats);
        }
    });
}

function readFile(p, stats) {
    fs.readFile(p, function(err, buf) {
        console.log(p);
    });
}

function readDirectory(p) {
    fs.readdir(p, function(err, files) {
        if (err) throw err;
        for (var i = 0, len = files.length; i < len; i++) {
            var childpath = path.join(p, files[i]);
            readPath(childpath);
        }
    });
}
```

在上面的执行过程中，

	串行了： 目录->文件
	并行了： 子节点

一个fs同步方法都没用，没有回调地狱，没有Promise，也没将异步执行逻辑塞进“串，并行队列“中, 跟写常规的同步逻辑几乎没差，看起来简单易读, 估计没写过js的人都能看懂。

众所周知，在javascript中，只有函数才有作用域（catch 是个特殊情况），并且在函数定义时会快照当前环境的作用域，在圈内这个东西叫 `闭包` ，有的人利用这一特性，每次执行异步调用时，先声明一个匿名函数，以便 `快照` 或 `抓住` 稍纵即逝的同步变量 ，其实完全没必要这么做，你用 `行参` 一样可以存你想要的东西，让我们再来看一个例子，这是一道考必包的经典题:

需求：每隔2秒 从1 console 到 5。

你可能会这么写
```javascript
for(var i=1;i<6;i++) {
    (function(){
	    var j = i;
		setTimeout(function(){
		   console.log(j);
		},2000*j)
	})()
}
```
或这么写
```javascript
for(var i=1;i<6;i++) {
    (function(i){
		setTimeout(function(){
		   console.log(i);
		},2000*i)
	})(i)
}
```

但其实根本没必要每次循环的时候都声明一个函数，`闭包` 是用来帮你保护私有变量的，在循环内部执行异步操作时，使用闭包 `快照` 局部变量的 相比 `行参` 代价较大（并且jshint会给你一条可爱的警告），所以你应该这么写

```javascript
function log(i){
	setTimeout(function(){
	   console.log(i);
	},2000*i)
}
for(var i=1;i<6;i++) {
	log(i);
}
```

javascript 是单线程异步编程语言，逻辑是一次一撸到底，然后等待事件发生再去响应，就好像蜘蛛织网，等待猎物上钩。所以我们可以将异步函数拆分封装成单个函数，看作是`事件`   , 每个事件就好像 `蜘蛛网` 上的一个节点。 在递归遍历目录的那个例子中，三个函数全是异步的，用可视化的方式在头脑中想一下这个画面，程序执行时，一次性创建了3个 `事件节点` ，然后当你触发 readPath 的时候，3个事件节点开始按照事先的约定相互响应。并且在每件事情的内部，可以根据需求的变动灵活控制的流程，还是以上边遍历目录为例改下需求：

需求：读取文件的最大并发数量不能超过 5

```javascript
var fs = require('fs');
var path = require('path');

var max = 5;
var pool = [];

readPath('./Documents');

function readPath(p) {
    fs.stat(path.normalize(p), function(err, stats) {
        if (err) throw err;
        if (pool.length < max) {
            if (stats.isFile()) {
                readFile(p, stats);
                pool.push(p)
            } else if (stats.isDirectory()) {
                readDirectory(p, stats);
                pool.push(p)
            }
        } else {
            readPath(p);
        }
    });
}

function readFile(p, stats) {
    fs.readFile(p, function(err, buf) {
        pool.splice(pool.indexOf(p),1);
        console.log(p);
    });
}

function readDirectory(p, stats) {
    fs.readdir(p, function(err, files) {
        if (err) throw err;
        pool.splice(pool.indexOf(p),1);
        for (var i = 0, len = files.length; i < len; i++) {
            var childpath = path.join(p, files[i]);
            readPath(childpath);
        }
    });
}
```

或者 抽出一个 `threadControl` 方法, 力求源代码最小化修改（仔细观察，其实下面的代码有个小小的bug，不过这里只是示意，没有考虑异常的情况）

```javascript
var fs = require('fs');
var path = require('path');

var max = 5;
var pool = [];

readPath('./Documents');

function threadControl(p) {
    if (pool.length < max) {
        pool.push(p)
        readPath(p);
    } else {
        setTimeout(function() {
            threadControl(p);
        }.bind(this), 10);
    }
}

function readPath(p) {
    fs.stat(path.normalize(p), function(err, stats) {
        if (err) throw err;
        if (stats.isFile()) {
            readFile(p, stats);
        } else if (stats.isDirectory()) {
            readDirectory(p, stats);
        }
    });
}

function readFile(p, stats) {
    fs.readFile(p, function(err, buf) {
        pool.splice(pool.indexOf(p), 1);
        console.log(p);
    });
}

function readDirectory(p, stats) {
    fs.readdir(p, function(err, files) {
        if (err) throw err;
        pool.splice(pool.indexOf(p), 1);
        for (var i = 0, len = files.length; i < len; i++) {
            var childpath = path.join(p, files[i]);
            threadControl(childpath);
        }
    });
}

```


Promise 的本质其实也是回调，只是内部走了一个hardcode 的 Pub/Sub , 而串并行队列是变形的回调计数器，说了半天还是跳不出回调的魔爪, 所以就算你使用了Promise,Serial/Parallel 这些工具, 你还是会有码出金字塔的概率。不管我们是否使用这些工具或es6 generator，在代码中减少一些不必要的 `闭包` 嵌套函数是非常必要的，这可以让你的世界变的清爽些，也可以让你或别人维护你的js代码时没那么痛苦.  程序是让机器执行，但却是给人读的，写的一手易读的代码挺好的，以上是我对JS异步编程的看法，希望有帮助。


张宁
2014－10－22