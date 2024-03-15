---
title: Javascript all(),bind(),apply()
tags:
  - nodejs
date: '2023-11-04'
summary: You need to know Javascript all(),bind(),apply()
---

### Javascript all(),bind(),apply() 方法介绍

1. 下面简化程序不work：

``` JavaScript
var abc = {
    sendmail: function () {
        console.log("sendmail")
    },
    get: function () {
        console.log("get")
        this.sendmail();
    },
}
setInterval(abc.get, 1000);

```
输出结果：

``` javascript
        this.sendmail();
             ^

TypeError: this.sendmail is not a function

```
2. 正确修改方法是：setInterval(abc.get.bind(abc), 1000); 为什么需要bind呢？
今天重温Javascript中call(), apply() 和 bind()函数是什么意思？
首先查看Javascript 函数属性有哪些？
``` javascript
interface Function {
    /**
     * Calls the function, substituting the specified object for the this value of the function, and the specified array for the arguments of the function.
     * @param thisArg The object to be used as the this object.
     * @param argArray A set of arguments to be passed to the function.
     */
    apply(this: Function, thisArg: any, argArray?: any): any;

    /**
     * Calls a method of an object, substituting another object for the current object.
     * @param thisArg The object to be used as the current object.
     * @param argArray A list of arguments to be passed to the method.
     */
    call(this: Function, thisArg: any, ...argArray: any[]): any;

    /**
     * For a given function, creates a bound function that has the same body as the original function.
     * The this object of the bound function is associated with the specified object, and has the specified initial parameters.
     * @param thisArg An object to which the this keyword can refer inside the new function.
     * @param argArray A list of arguments to be passed to the new function.
     */
    bind(this: Function, thisArg: any, ...argArray: any[]): any;

    /** Returns a string representation of a function. */
    toString(): string;

    prototype: any;
    readonly length: number;

    // Non-standard extensions
    arguments: any;
    caller: Function;
}
```
3. 具体细节我就不再讲述了，代码有注释。举个例子说明吧:
先看下bind函数，官方文档说：该bind()方法创建了一个新函数，在调用时，其this关键字设置为提供的值。这是非常强大的。它让我们明确定义this调用函数时的值。让我们分解一下。当我们使用该bind()方法时：JS 引擎正在创建一个新pokemonName实例并将pokemon其绑定为它的this变量。理解它复制 pokemonName 函数很重要。创建pokemonName函数的副本后，它可以调用logPokemon()，尽管它最初不在 pokemon 对象上。它现在将识别其属性（Pika and Chu）及其方法。很酷的事情是，在我们 bind() 一个值之后，我们可以像使用任何其他普通函数一样使用该函数。我们甚至可以更新函数以接受参数，传递它们.
```javascript
var pokemon = {
    firstname: 'Pika',
    lastname: 'Chu ',
    getPokeName: function() {
        var fullname = this.firstname + ' ' + this.lastname;
        return fullname;
    }
};

var pokemonName = function() {
    console.log(this.getPokeName() + 'I choose you!');
};

var logPokemon = pokemonName.bind(pokemon); // creates new object and binds pokemon. 'this' of pokemon === pokemon now

logPokemon(); // 'Pika Chu I choose you!'
```

call(), apply()方法，call() 的官方文档说：该call()方法调用具有给定this值和单独提供的参数的函数。这意味着我们可以调用任何函数，并明确指定在调用函数中this应该引用什么。与bind()方法真的很像！bind()和call()主要区别在于call()方法接受额外的参数，立即执行它被调用的函数，call() 方法不会复制正在调用它的函数。call()并apply()服务于完全相同目的。它们工作方式的唯一区别是call() 期望所有参数单独传入，而 apply() 期望我们所有参数的数组。请注意，apply 接受数组，并且 call期望转递每个参数。

``` javascript
var pokemon = {
    firstname: 'Pika',
    lastname: 'Chu ',
    getPokeName: function() {
        var fullname = this.firstname + ' ' + this.lastname;
        return fullname;
    }
};

var pokemonName = function(snack, hobby) {
    console.log(this.getPokeName() + ' loves ' + snack + ' and ' + hobby);
};

pokemonName.call(pokemon,'sushi', 'algorithms'); // Pika Chu  loves sushi and algorithms
pokemonName.apply(pokemon,['sushi', 'algorithms']); // Pika Chu  loves sushi and algorithms

```

这些存在于每个 JS 函数中的内置方法非常有用。即使您最终没有在日常编程中使用它们，您在阅读其他人的代码时仍然会经常遇到它们。

### 参考文档
[1] [mozilla.org](https://developer.mozilla.org/en-US/docs/Web/JavaScript/)  
[2] [medium.com](https://medium.com/@omergoldberg/javascript-call-apply-and-bind-e5c27301f7bb)