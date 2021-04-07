#! https://zhuanlan.zhihu.com/p/363023930
# 温故而知新 - 重新认识JavaScript的Hoisting

为JavaScript里面的概念，温故而知新。

### 概念：Hositing是什么
在JavaScript中，如果试图使用尚未声明的变量，会出现`ReferenceError`错误。毕竟变量都没有声明，JavaScript也就找不到这变量。加上变量的声明，可正常运行：
```js
console.log(a);
// Uncaught ReferenceError: a is not defined
```
```js
var a;
console.log(a); // undefined
```

考虑下如果是这样书写：
```js
console.log(a); // undefined
var a;
```
直觉上，程序是自上向下逐行执行的。使用尚未声明的变量**a**，按理应该出现`ReferenceError`错误，而实际上却输出了`undefined`。这种现象，就是[**Hoisting**](https://developer.mozilla.org/en-US/docs/Glossary/Hoisting)。`var a`由于某种原因被"*移动*"到最上面了。可以理解为如下形式：
```js
var a;
console.log(a); // undefined
```

需要**注意**：
1. 实际上声明在代码里的位置是不会变的。
2. hoisting只是针对声明，赋值并不会。

    ```js
    console.log(a); // undefined
    var a = 2015;

    // 理解为如下形式
    var a;
    console.log(a); // undefined
    a = 2015;
    ```
    这里`var a = 2015`理解上可分成两个步骤：`var a`和`a = 2015`。
3. 函数表达式不会`hoisting`。

    ```js
    fn(); // TypeError: fn is not a function
    var fn = function () {}
    
    // 理解为如下形式
    var fn;
    fn();
    fn = function () {};
    ```
    这里`fn()`对`undefined`值进行函数调用导致非法操作，因此抛出`TypeError`错误。

4. 函数声明和变量声明，都会`hoisting`，需要注意的是，函数会优先`hoisting`：
    ```js
    console.log(fn);
    var fn;
    function fn() {}

    // 理解为如下形式
    function fn() {}
    var fn; // 重复声明，会被忽略
    console.log(fn);
    ```

对于有参数的函数：
```js
fn(2016);

function fn(a) {
    console.log(a); // 2016
    var a = 2015;
}

// 理解为如下形式
function fn(a) {
    var a = 2016; // 这里对应传参，值为函数调用时候传进来的值
    var a; // 重复声明，会被忽略
    console.log(a);
    a = 2015;
}
fn(2016);
```

总结一下，可以理解`Hoisting`是处理所有声明的过程。需要注意**赋值及函数表达式不会hoisting**。

### 意义：为什么需要Hoisting
可以处理函数互相调用的场景：
```js
function fn1(n) {
    if (n > 0) fn2(n);
}

function fn2(n) {
    console.log(n);
    fn1(n - 1);
}

fn1(6);
```
按逐行执行的观念来看，必然存在先后顺序，像`fn1`与`fn2`之间的相互调用，如果没有`hoisting`的话，是无法正常运行的。

### 规范：Hoisting的运行规则
具体可以参考规范[ECMAScript 2019 Language Specification](https://262.ecma-international.org/10.0/)。与Hoisting相关的，是在[`8.3 Execution Contexts`](https://262.ecma-international.org/10.0/#sec-execution-contexts)。

一篇很不错的文章参考[`Understanding Execution Context and Execution Stack in Javascript`](https://blog.bitsrc.io/understanding-execution-context-and-execution-stack-in-javascript-1c9ea8642dd0)

参考里面的例子：
```js
var a = 20;
var b = 40;
let c = 60;

function foo(d, e) {
    var f = 80;
    
    return d + e + f;
}

c = foo(a, b);
```
创建的`Execution Context`像这样：
```js
GlobalExectionContext = {
  LexicalEnvironment: {
    EnvironmentRecord: {
      Type: "Object",
      c: < uninitialized >,
      foo: < func >
    }
    outer: <null>,
    ThisBinding: <Global Object>
  },
  VariableEnvironment: {
    EnvironmentRecord: {
      Type: "Object",
      // Identifier bindings go here
      a: undefined,
      b: undefined,
    }
    outer: <null>, 
    ThisBinding: <Global Object>
  }
}
```
在运行阶段，变量赋值已经完成。因此`GlobalExectionContext`在执行阶段看起来就像是这样的：
```js
GlobalExectionContext = {
  LexicalEnvironment: {
    EnvironmentRecord: {
      Type: "Object",
      c: 60,
      foo: < func >,
    }
    outer: <null>,
    ThisBinding: <Global Object>
  },
  VariableEnvironment: {
    EnvironmentRecord: {
      Type: "Object",
      // Identifier bindings go here
      a: 20,
      b: 40,
    }
    outer: <null>, 
    ThisBinding: <Global Object>
  }
}
```
当遇到函数`foo(a, b)`的调用时，新的`FunctionExectionContext`被创建并执行函数中的代码。在创建阶段像这样：
```js
FunctionExectionContext = {
  LexicalEnvironment: {
    EnvironmentRecord: {
      Type: "Declarative",
      Arguments: {0: 20, 1: 40, length: 2},
    },
    outer: <GlobalLexicalEnvironment>,
    ThisBinding: <Global Object or undefined>,
  },
  VariableEnvironment: {
    EnvironmentRecord: {
      Type: "Declarative",
      f: undefined
    },
    outer: <GlobalLexicalEnvironment>,
    ThisBinding: <Global Object or undefined>,
  }
}
```
执行完后，看起来像这样：
```js
FunctionExectionContext = {
  LexicalEnvironment: {
    EnvironmentRecord: {
      Type: "Declarative",
      Arguments: {0: 20, 1: 40, length: 2},
    },
    outer: <GlobalLexicalEnvironment>,
    ThisBinding: <Global Object or undefined>,
  },
  VariableEnvironment: {
    EnvironmentRecord: {
      Type: "Declarative",
      f: 80
    },
    outer: <GlobalLexicalEnvironment>,
    ThisBinding: <Global Object or undefined>,
  }
}
```
在函数执行完成以后，返回值会被存储在`c`里。因此`GlobalExectionContext`更新。在这之后，代码执行完成，程序运行终止。

### 细节：var、let、const在hoisting上的差异
回顾`规范：Hoisting的运行规则`，可以注意到在创建阶段，不管是用`let`、`const`或`var`，都会进行`hoisting`。而差别在于：使用`let`和`const`进行声明的时候，设置为`uninitialized`(未初始化状态)，而`var`会设置为`undefined`。所以在`let`或`const`声明的变量之前访问时，会抛出`ReferenceError: Cannot access 'c' before initialization`错误。对应的名词为`Temporal Dead Zone`(暂时性死区)。
```js
function demo1() {
    console.log(c); // c 的 TDZ 开始
    let c = 10; // c 的 TDZ 结束
}
demo1();

function demo2() {
    console.log('begin'); // c 的 TDZ 开始
    let c; // c 的 TDZ 结束
    console.log(c);
    c = 10;
    console.log(c);
}
demo2();
```

### 总结
果真是温故而知新，发现自己懂得其实好少。鞭策自己，后续对`this`、`prototype`、`closures`及`scope`等，进行温故。

#### 参考资料
1. [我知道你懂 hoisting，可是你了解到多深？](https://blog.huli.tw/2018/11/10/javascript-hoisting-and-tdz/)
2. [MDN: Hoisting](https://developer.mozilla.org/en-US/docs/Glossary/Hoisting)
3. [You-Dont-Know-JS 2nd-ed](https://github.com/getify/You-Dont-Know-JS/blob/2nd-ed/scope-closures/ch1.md)
4. [ECMAScript 2019 Language Specification](https://262.ecma-international.org/10.0/)
5. [Understanding Execution Context and Execution Stack in Javascript](https://blog.bitsrc.io/understanding-execution-context-and-execution-stack-in-javascript-1c9ea8642dd0)
6. JavaScript高级程序设计