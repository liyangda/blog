# 温故而知新：重新认识JavaScript的this

`this`是`JavaScript`中的一个关键字，关于讲解`this`的文章，网上资源很多。这里记录关于`this`的思考：什么是`this`、为什么有`this`。

## 什么是`this`？

**`this`是描述函数执行的记录中的一个属性，只能在函数内部被访问到**，会在函数执行过程中被用到。

看规范中定义的抽象操作[`Call(F, V [, argumentsList])`](https://262.ecma-international.org/10.0/#sec-call)：
>The abstract operation Call is used to call the `[[Call]]` internal method of a **function object**. The operation is called with arguments F, V, and optionally argumentsList where **F is the function object**, **V is an ECMAScript language value that is the this value of the `[[Call]]`**, and **argumentsList is the value passed to the corresponding argument of the internal method**. If argumentsList is not present, a new empty List is used as its value.

执行`Call`操作的时候已经传进去了`this`值，那再看下相关表达式[Function Calls](https://262.ecma-international.org/10.0/#sec-function-calls)的说明：

```text
...
4.Let ref be the result of evaluating memberExpr.
...
9.Return ? EvaluateCall(func, ref, arguments, tailCall).
```

转到[EvaluateCall](https://262.ecma-international.org/10.0/#sec-evaluatecall)：

```text
1.If Type(ref) is Reference, then
    a. If IsPropertyReference(ref) is true, then
        i. Let thisValue be GetThisValue(ref).
    b. Else the base of ref is an Environment Record,
        i. Let refEnv be GetBase(ref).
       ii. Let thisValue be refEnv.WithBaseObject().
2.Else Type(ref) is not Reference,
    a. Let thisValue be undefined.
```

看这是在描述`this`的值如何确定，而`EvaluateCall`的入参是没有`this`。这么来看的话，可以理解，`this`是函数执行时，内部过程中的某个属性。即开头的结论：**`this`是描述函数执行的记录中的一个属性，只能在函数内部被访问到**

## 为什么有`this`？

为什么会有，可以思考`this`的使用，解决了什么问题？

[You Don't Know JS: this & Object Prototypes](https://github.com/getify/You-Dont-Know-JS/blob/1st-ed/this%20%26%20object%20prototypes/ch1.md)中这么解释的：

>`this`提供了一种更优雅的方式来隐式“传递”一个对象引用，因此可以将`API`设计得更加简洁并且易于复用。
>```js
>function identify() {
>   return this.name.toUpperCase();
>}
>
>function speak() {
>  var greeting = "Hello, I'm " + identify.call(this);
>}
>
>var me = { name: "Kyle" };
>var you = { name: "Reader" };
>
>identify.call(me); // KYLE
>identify.call(you); // READER
>
>speak.call(me); // Hello, I'm KYLE
>speak.call(you); // Hello, I'm READER
>```

(⊙o⊙)…尝试从这个单词本身思考下，`this`应该是作为一个代词，那就应该是起代替或指示作用，`this`用于近指。那指代的是什么呢？那就又得回到`ECMAScript`规范：
>The base value component is either undefined, an Object, a Boolean, a String, a Symbol, a Number, or an Environment Record. A base value component of undefined indicates that the Reference could not be resolved to a binding. The referenced name component is a String or Symbol value.

这么来看，`this`指代的事物的值可能性还蛮多的。考虑下，如果`undefined`, `an Object`, `a Boolean`, `a String`, `a Symbol`, `a Number`这些值，大可使用个标识符表示即可，没必要用到`this`。

所以，大胆猜测，`this`是不是指代`environment record`，而且还是就近的`environment record`？？？

## 参考资料

1. [You Don't Know JS: this & Object Prototypes](https://github.com/getify/You-Dont-Know-JS/blob/1st-ed/this%20%26%20object%20prototypes/ch1.md)
2. [What is “this” keyword in JavaScript?](www.educba.com/this-keyword-in-javascript/)
3. [How does the “this” keyword work?](https://stackoverflow.com/questions/3127429/how-does-the-this-keyword-work)
4. [JavaScript深入之从ECMAScript规范解读this](https://github.com/mqyqingfeng/Blog/issues/7)
5. [JS this](https://github.com/nightn/front-end-plan/blob/master/js/js-this.md)
6. [this 的值到底是什么？一次说清楚](https://zhuanlan.zhihu.com/p/23804247)