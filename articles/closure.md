#! https://zhuanlan.zhihu.com/p/363640347
# 温故而知新 - 重新认识JavaScript的Closure
闭包的重温。

## 定义：什么是Closure？

[MDN: Closures](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Closures) 的定义如下：
>A **closure** is the combination of a function bundled together (enclosed) with references to its surrounding state (the **lexical environment**). In other words, a closure gives you access to an outer function’s scope from an inner function. In JavaScript, closures are created every time a function is created, at function creation time.

再来看看，来自 [专业性](https://en.wanweibaike.com/wiki-Closure%20(programming)) 的解释：
>In programming languages, a **closure**, also **lexical closure** or **function closure**, is a technique for implementing lexically scoped name binding in a language with first-class functions. Operationally, a closure is a record storing a function together with an environment. The environment is a mapping associating each **free variable** of the function (variables that are used locally, but defined in an enclosing scope) with the value or reference to which the name was bound when the closure was created. Unlike a plain function, a closure allows the function to access those *captured variables* through the closure's copies of their values or references, even when the function is invoked outside their scope.

简而言之，`closure`是由一个函数以及声明该函数的词法环境组成。

需要注意几个点：

- a closure is a record

  闭包是一个结构体，存储了一个函数（通常是其入口地址）和一个声明该函数的环境（相当于一个符号查找表）。

- free variable

  自由变量（在函数外部定义但在函数内被引用的变量）。闭包跟函数最大的不同在于，当捕捉闭包的时候，它的自由变量会在捕捉时被确定，这样即便脱离了捕捉时的上下文，它也能照常运行。

来看下面的代码：

```js
var a = 1;
function test() {
  const b = 1;
  function inner1() {
    var c = a + 2;

    function inner2() {
      var d = a + b;

      return c;
    };

    inner2()
  };

  inner1();
}
test();
```

使用`Chrome`，分别在`inner1()`、`inner2()`、`return c`三处进行断点调试：

- Step1: 程序运行到`inner1()`

  可以在控制台看到`Scope`里面有`Local`和`Global`两项

- Step2: 程序运行到`inner2()`

  可以在控制台看到`Scope`里面有`Local`、`Closure（test）`、`Global`三项，`Closure（test）`里面有函数`test`中定义的变量`b`

- Step3: 程序运行到`return c`

  可以在控制台看到`Scope`里面有`Local`、`Closure（test）`、`Closure（inner1）`、`Global`四项，`Closure（test）`里面有函数`test`中定义的变量`b`，`Closure（inner1）`里面有函数`inner1`中定义的变量`c`

若内部函数不使用外部函数声明的变量，来修改实例调试看看：

```js
var a = 1;
function test() {
  const b = 1;
  function inner1() {
    var c = a + 2;

    function inner2() {
      var d = a + c;

      return c;
    };

    inner2()
  };

  inner1();
}
test();
```

继续按之前步骤调试，可以发现程序运行到`inner2()`时，`Scope`里面没有`Closure（test）`了。呈现的结果：内部函数不引用任何外部函数中的变量，不会产生闭包。

MDN里面提到
>In JavaScript, closures are created every time a function is created, at function creation time.

再回顾`closure`概念，其包括函数入口地址和关联的环境。这跟函数创建时类似的，从这角度看，**函数是闭包**。那么按理论来讲，上述例子应该是会产生闭包，而实际在`chrome`调试发现并没有。猜测是不是这里进行了优化，当没有自由变量的时候，就不进行闭包处理？找了找资料，发现这确实是编译技巧，叫[`Lambda lifting`](https://en.wanweibaike.com/wiki-Closure_conversion)。

至此，总结**闭包产生的必要条件**：

- 函数存在内部函数，即函数嵌套
- 内部函数存在引用任何外部函数中的变量，即存在自由变量
- 内部函数被执行

## 分析：为什么`closure`会常驻内存？

先来结论：闭包之所以会常驻在内存，在于闭包的`lexical environment`没有被释放。

```js
function test() {
  var a = 0;

  return {
    increase: function() { a++; },
    getValue: function() { return a; },
  };
}

var obj = test();

obj.getValue();
obj.increase();
obj.getValue();
```

尝试分析下执行过程：

1. 初始执行，创建全局`lexical environment`

    ```text
    GlobalExecutionContext = {
      LexicalEnvironment: {
        EnvironmentRecord: {
          Type: "Object",
          test: < func >
        }
        outer: <null>,
        ThisBinding: <Global Object>
      },
      VariableEnvironment: {
        EnvironmentRecord: {
          Type: "Object",
          // Identifier bindings go here
          obj: undefined,
        }
        outer: <null>, 
        ThisBinding: <Global Object>
      }
    }
    ```

2. 执行`test`函数，进入`test`函数的`lexical environment`

    ```text
    FunctionExecutionContextOfTest = {
      LexicalEnvironment: {
        EnvironmentRecord: {
          Type: "Object",
        }
        outer: <null>,
        ThisBinding: <Global Object>
      },
      VariableEnvironment: {
        EnvironmentRecord: {
          Type: "Object",
          a: 0,
        }
        outer: <null>, 
        ThisBinding: <Global Object>
      }
    }
    ```

3. 返回一个对象，里面包含两个函数，两个函数的`lexical environment`

    ```text
    FunctionExecutionContextOfIncrease、
    FunctionExecutionContextOfGetValue： {
      LexicalEnvironment: {
        EnvironmentRecord: {
          Type: "Object",
        }
        outer: <LexicalEnvironmentOfTest>,
        ThisBinding: <Global Object or undefined>
      },
      VariableEnvironment: {
        EnvironmentRecord: {
          Type: "Object",
        }
        outer: <LexicalEnvironmentOfTest>, 
        ThisBinding: <Global Object or undefined>
      }
    }
    ```

4. 调用`obj`的`getValue`函数，进入`getValue`函数的`lexical environment`，按规范，通过[`GetIdentifierReference(lex, name, strict)`](https://262.ecma-international.org/10.0/#sec-getidentifierreference)来解析`a`，在`increase`函数的`lexical environment`中并不能解析`a`，便转到`outer`对应的外部`lexical environment`中继续解析

5. 调用`obj`的`increase`函数，过程类似调用`obj`的`getValue`函数。

大体过程是如此。规范中关于[FunctionInitialize](https://262.ecma-international.org/10.0/#sec-functioninitialize)，可以看到`4.Set F.[[Environment]] to Scope.`，看下规范关于`[[Environment]]`的说明

> | Type | Description |
> | --- | --- |
> | Lexical Environment | The Lexical Environment that the function was closed over. Used as the outer environment when evaluating the code of the function. |

那么，调用`obj`的`getValue`函数，转到`outer`对应的外部`lexical environment`中继续解析，这里的`lexical environment`，应该是指函数的内部属性`[[Scopes]]`。

在之前`chrome`中调试的例子，可以看到`Scope`中出现`Closure（test）`，里面保留的是`test`函数`enviroment record`中被其他`lexical environment`引用的那部分。

此外，`obj.getValue()`是在`Global lexical environment`中被调用，被执行时，其`outer`却不是`Global lexical environment`，这是因为外部词法环境引用指向的是逻辑上包围内部词法环境的词法环境：
>The outer reference of a (inner) Lexical Environment is a reference to the Lexical Environment that logically surrounds the inner Lexical Environment.

## 总结

`closure`，搜寻相关资料重新认识，阅读规范，理解专业定义。原来很多东西并不是表面上那么简单！

## 参考资料
1. 万维百科[Closure](https://en.wanweibaike.com/wiki-Closure%20(programming))
2. MDN[Closures](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Closures)
3. [ECMAScript® 2019 Language Specification](https://262.ecma-international.org/10.0/)
3. [解读闭包，这次从ECMAScript词法环境，执行上下文说起](https://cloud.tencent.com/developer/article/1676297)
4. [所有的函式都是閉包：談 JS 中的作用域與 Closure](https://blog.huli.tw/2018/12/08/javascript-closure/)