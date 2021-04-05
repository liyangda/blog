更新下相关知识，立足过往，拥抱未来。

## 概念
直接看规范关于`ExecutionContext`的定义：

> An execution context is a specification device that is used to track the runtime evaluation of code by an ECMAScript implementation.

`ExecutionContext`为抽象概念，用来描述可执行代码的执行环境。可执行代码的运行，都是在`ExecutionContext`中。

#### 管理方式`ExecutionContextStack`

`Execution context Stack`为后进先出（LIFO）的栈结构。栈顶永远是`running execution context`。当控制从当前`execution context`对应可执行代码转移到另一段可执行代码时，相应的`execution context`将被创建，并压入栈顶，执行结束，对应的`execution context`从栈顶弹出。

##### 思索：什么`ECMAScript`特性会使`Execution context stack`不遵循`LIFO`规则？
规范里面提到：
>Transition of the running execution context status among execution contexts usually occurs in stack-like last-in/first-out manner. However, some ECMAScript features require non-LIFO transitions of the running execution context.

然后在规范里面，并没有找到`some ECMAScript features`到底是什么特性。不过，第一反应，`Generator`算不算？在stackoverflow上，有这么一个讨论[Execution Context Stack. Violation of the LIFO order using Generator Function](https://stackoverflow.com/questions/48365754/execution-context-stack-violation-of-the-lifo-order-using-generator-function?r=SearchResults#)

```js
function *gen() {
  yield 1;
  return 2;
}

let g = gen();

console.log(g.next().value);
console.log(g.next().value);
```
调用一个函数时，当前`execution context`暂停执行，被调用函数的`execution context`创建并压入栈顶，当函数返回时，函数`execution context`被销毁，暂停的`execution context`得以恢复执行。

现在，使用的是`Generator`，`Generator`函数的`execution context`在返回`yield`表达式的值之后仍然存在，并未销毁，只是暂停并移交出了控制权，可能在某些时候恢复执行。

究竟是不是呢？有待求证。

### 词法环境`Lexical Environments`
看规范的定义：
>A Lexical Environment is a specification type used to define the association of Identifiers to specific variables and functions based upon the lexical nesting structure of ECMAScript code.

按规范来说，`Lexical Environment`定义了标识`identifiers`与`Variables`或`Functions`的映射。

#### 组成 `Lexical Environment`包含两部分：
- `Environment Record`
记录被创建的标识`identifiers`与`Variables`或`Functions`的映射

| 类型 | 简述 |
| --- | --- |
| [Declarative Environment Records](https://262.ecma-international.org/10.0/#sec-declarative-environment-records) | 记录 `var`、`const`、`let`、`class`、`import`、`function`等声明 |
| [Object Environment Records](https://262.ecma-international.org/10.0/#sec-object-environment-records) | 与某对象绑定，记录该对象中`string identifier`的属性，非`string identifier`的属性不会被记录。`Object environment records`为`with`语句所创建 |
| [Function Environment Records](https://262.ecma-international.org/10.0/#sec-function-environment-records) | `Declarative Environment Records`的一种，用于函数的顶层，如果为非箭头函数的情况，提供`this`的绑定，若还引用了 `super`则提供`super`方法的绑定 |
| [Global Environment Records](https://262.ecma-international.org/10.0/#sec-global-environment-records) | 包含所有顶层声明及`global object`的属性，`Declarative Environment Records`与`Object Environment Records`的组合 |
| [Module Environment Records](https://262.ecma-international.org/10.0/#sec-module-environment-records) | `Declarative Environment Records`的一种，用于`ES module`的顶层，除去常量和变量的声明，还包含不可变的`import`的绑定，该绑定提供了到另一`environment records`的间接访问 |

- 外部`Lexical Environment`的引用
通过引用构成了嵌套结构，引用可能为`null`

#### 分类 `Lexical Environment`分三类：
- `Global Environment`
没有外部`Lexical Environment`的`Lexical Environment`
- `Module Environment` 
包含了模块顶层的声明以及导入的声明，外部`Lexical Environment`为`Global Environment`
- `Function Environment`
对应于`JavaScript`中的函数，其会建立`this`的绑定以及必要的`super`方法的绑定

### 变量环境`Variable Environments`
在ES6前，声明变量都是通过`var`声明的，在ES6后有了`let`和`const`进行声明变量，为了兼容`var`，便用`Variable Environments`来存储`var`声明的变量。

`Variable Environments`实质上仍为`Lexical Environments`

## 机制
具体可以参考规范[ECMAScript 2019 Language Specification](https://262.ecma-international.org/10.0/)。相关的是在[`8.3 Execution Contexts`](https://262.ecma-international.org/10.0/#sec-execution-contexts)。

一篇很不错的文章参考[`Understanding Execution Context and Execution Stack in Javascript`](https://blog.bitsrc.io/understanding-execution-context-and-execution-stack-in-javascript-1c9ea8642dd0)， 该文章的中文翻译版[中文版](https://juejin.cn/post/6844903682283143181#heading-8)

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
GlobalExecutionContext = {
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
在运行阶段，变量赋值已经完成。因此`GlobalExecutionContext`在执行阶段看起来就像是这样的：
```js
GlobalExecutionContext = {
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
当遇到函数`foo(a, b)`的调用时，新的`FunctionExecutionContext`被创建并执行函数中的代码。在创建阶段像这样：
```js
FunctionExecutionContext = {
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
FunctionExecutionContext = {
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
在函数执行完成以后，返回值会被存储在`c`里。因此`GlobalExecutionContext`更新。在这之后，代码执行完成，程序运行终止。

## 总结
`ECMAScript`规范是年年都在更新，得与时俱进的加强学习，立足过往及当下，拥抱未来！

## 参考资料
1. [所有的函式都是閉包：談 JS 中的作用域與 Closure](https://blog.huli.tw/2018/12/08/javascript-closure/)
2. [JS夯实之执行上下文与词法环境](https://my.oschina.net/u/4347428/blog/4263452)
3. [结合 JavaScript 规范来谈谈 Execution Contexts 与 Lexical Environments](https://blog.csdn.net/qq_35368183/article/details/103888311)
3. [You-Dont-Know-JS 2nd-ed](https://github.com/getify/You-Dont-Know-JS/blob/2nd-ed/scope-closures/ch1.md)
4. [ECMAScript 2019 Language Specification](https://262.ecma-international.org/10.0/)
5. [Understanding Execution Context and Execution Stack in Javascript](https://blog.bitsrc.io/understanding-execution-context-and-execution-stack-in-javascript-1c9ea8642dd0)
