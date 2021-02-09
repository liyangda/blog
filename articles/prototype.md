Javascript中对象有一个特殊的`[[Prototype]]`内置属性，其实就是对于其他对象的引用。几乎所有的对象在创建时`[[Prototype]]`属性都会被赋予一个非空的值。

所有普通的`[[Prototype]]`链最终都会指向内置的`Object.prototype`。

**JavaScrip和面向类的语言不同，它并没有类来作为对象的抽象模式。JavaScript中只有对象**

### 属性设置与屏蔽
```javascript
myObject.foo = "bar";
```
如果`myObject`对象中包含名为foo的普通数据访问属性，这条赋值语句只会修改已有的属性值。
如果`foo`不是直接存在于`myObject`中，`[[Prototype]]`链就会被遍历，类似`[[Get]]`操作。如果原型链上找不到`foo`，`foo`就会被直接添加到`myObject`上。

然而，如果`foo`存在于原型链上层，赋值语句`myObject.foo = "bar"`的行为就会有些不同：

> 1. 如果在`[[Prototype]]`链上层存在名为`foo`的普通数据访问属性，并且没有被标记为只读(`writable: false`)，那就会直接在`myObject`中添加一个名为`foo`的新属性，它是**屏蔽属性**
> 2. 如果在`[[Prototype]]`链上层存在`foo`，但是它被标记为只读(`writable: false`)，那么无法修改已有属性或者在myObject上创建屏蔽属性。如果运行在严格模式下，代码会抛出一个错误。否则，这条赋值语句会被忽略。总之，不会发生屏蔽。**更为奇怪的是，这个限制只存在于`=`赋值中，使用`Object.defineProperty(...)`并不会受影响**
> 3. 如果在`[[Prototype]]`链上层存在`foo`并且它是一个setter，那就一定会调用这个`setter`。`foo`不会被添加到(或者说屏蔽于)`myObject`，也不会重新定义`foo`这个`setter`

来看代码，测试执行下：
```javascript
class Parent {};
class Son extends Parent {}

// 测试第二种情况
Object.defineProperty(Parent, 'foo', {writable: false, value: 'bar'});

Son.foo; // 输出"bar"
Son.foo = "foo";
Son.foo; // 依旧输出"bar", 上一条赋值语句被忽略了
// 但是可以使用Object.defineProperty(...)
Object.defineProperty(Son, 'foo', {value: 'son_bar'});
Son.foo; // 输出"son_bar"
Parent.foo; // 输出"bar"

// 测试第三种情况
Object.defineProperty(
    Parent,
    'goo', 
    {
        set: function() {
            console.log('set被调用了');
        }
    }
);
/**
 * 正如第三点说的：[[Prototype]]链上层存在foo并且它是一个setter，
 * 那就一定会调用这个setter。
 */
Son.goo = 'son_goo'; // 输出 'set被调用了'
Son.goo; // undefined
Parent.hasOwnProperty('goo'); // true
Son.hasOwnProperty('goo'); // 输出false，foo并没有添加到Son中

// 用Object.defineProperty(...);
Object.defineProperty(
    Son,
    'goo', 
    {
        set: function() {
            console.log('son中的set被调用了');
        }
    }
);
Son.hasOwnProperty('goo'); // 输出true
Son.goo; // undefined
/**
 * 下一语句执行后，控制台没有输出。setter并没有被调用
 */ 
Son.goo = 'goo'; // 没有输出 'son中的set被调用了'
Son.goo; // undefined
```

### 类函数
```javascript
function Foo() {}

var a = new Foo();

Object.getPrototypeOf(a) === Foo.prototype; // true
```
调用`new Foo()`时会创建`a`，其中一步就是将`a`内部的`[[Prototype]]`链接到`Foo.prototype`所指向的对象。

在JavaScript中，不能创建一个类的多个实例，只能创建多个对象，它们`[[Prototype]]`关联的是同一个对象。`new Foo()`生成一个新对象`a`，`a`内部链接`[[Prototype]]`关联的是`Foo.prototype`。最后我们得到了两个对象，它们之间互相关联。并没有初始化一个类，实际上并没有从'类'中复制任何行为到一个对象中，只是让两个对象互相关联。

### 对象关联
`Object.create(null)`会创建一个拥有空(或者说`null`)`[[Prototype]]`链接的对象，这个对象无法进行委托。由于这个对象没有原型链，所以`instanceof`操作符无法进行判断，因此总是会返回false。这些特殊的空`[[Prototype]]`对象通常被称作"字典"，他们完全不会受到原型链的干扰，因此非常适合用来**存储数据**

```javascript
// Object.create()的polyfill代码
if (!Object.create) {
    Object.create = function(target) {
        function F() {}
        F.prototype = target;
        return new F();
    }
}
```