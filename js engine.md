# 深入 js 执行上下文

### 前言

编写一段 js 代码，它是如何被执行的？js 引擎在执行代码的过程中都做了哪些操作？本文将会带大家来了解一下 js 的详细执行过程，包括对 ECS、GO、VO、VE、AO、作用域链等概念的理解。

> 注：本文只探讨 js 的执行流程，不对页面渲染（例如 html 如何生成 dom 树、CSSOM 等）做阐述

### 一、JS 执行流程

#### 1、初始化全局对象（GO）
首先，js 引擎会在执行代码之前，也就是解析代码时，会在我们的堆内存创建一个全局对象：Global Object（GO）

- 所有的作用域（scope）都可以访问该全局对象
- 对象里面会包含一些全局的方法和类，像 Math、Date、String、Array、setTimeout 等等
- 其中有一个 window 属性是指向该全局对象自己

例如以下代码：

```
var name = 'curry'
var message = 'I am a coder'
var num = 30
```

初始化时生成的 GO 伪代码大致是：

```
GlobalObject = {
  Math: '类',
  Date: '类',
  String: '类',
  setTimeout: '函数',
  setInterval: '函数',
  window: GlobalObject,
  ...
  name: undefined,
  message: undefined,
  num: undefined
}
```

#### 2、执行上下文调用栈（ECS）
js 引擎为了执行代码，引擎内部会有一个执行上下文调用栈（Execution Context Stack，简称 ECS），它被用来压入或弹出执行上下文（EC），从而执行代码

执行上下文（EC）有三个重要的属性，变量对象（Variable object，VO），作用域链（Scope chain）和 this，后面会介绍

在 js 中有三种执行上下文类型：

- 全局执行上下文：这是默认或者说是最基础的执行上下文，一个程序中只会存在一个全局上下文，它在整个 javascript
脚本的生命周期内都会存在于执行堆栈的最底部不会被栈弹出销毁。全局上下文会生成一个全局对象（以浏览器环境为例，这个全局对象是
window），并且将 this 值绑定到这个全局对象上
- 函数执行上下文：每当一个函数被调用时，都会创建一个新的函数执行上下文（不管这个函数是不是被重复调用）
- Eval 函数执行上下文：执行在 eval 函数内部的代码也会有它属于自己的执行上下文，但由于并不经常使用
eval，所以在这里不做分析

**ECS 开始时如何执行？**

ECS 会先执行我们的全局代码块，在执行前会构建一个全局执行上下文（Global Execution Context，简称 GEC），一开始 GEC 就会被放入到 ECS 中执行

全局执行上下文（GEC）包含两部分内容：

- 创建阶段：会将全局定义的变量、函数等加入到 GO 中（也就是上面的初始化对象过程），但是并不会真正赋值（初始为 undefined），这个过程也就是为大家所熟知的：变量的作用域提升（hoisting）
- 执行阶段：对变量进行赋值，或者执行其它的函数

下图为 GEC 被放入 ECS 后的表现形式：

![image](https://article.biliimg.com/bfs/article/7112c0c871e2bbb9b3c66198c1d72b535d2d981f.png)

#### 3、变量对象（VO）

从上面看到，在 Execution Context 中，会保存变量对象（Variable object，VO），下面就看看变量对象是什么

**变量对象**是与执行上下文相关的数据作用域。它是一个与上下文相关的特殊对象，其中存储了在上下文中定义的变量和函数声明。一般 VO 中会包含以下信息：

- 变量（Variable Declaration，var）
- 函数声明（Function Declaration，FD）
- 函数的形参（arguments）

当 js 代码运行中，如果试图寻找一个变量的时候，就会首先查找 VO。对于前面例子中的代码，GEC 中的 VO 就指向 GO

注意，假如上面的例子代码中有下面两个语句，Global VO 仍将不变

```
// 函数表达式
(function bar(){})
// 全局声明
baz = "property of global object"
```

也就是说，对于VO，下面两种情况比较特殊：

- 函数表达式（与函数声明相对）不包含在 VO 之中
- 没有使用 var 声明的变量（这种变量是"全局"的声明方式，只是给 Global 添加了一个属性，并不在 VO 中）

#### 4、活动（激活）对象（AO）

只有全局上下文的变量对象允许通过 VO 的属性名称间接访问；在函数执行上下文中，VO 是不能直接访问的，此时由激活对象（Activation Object，缩写为AO）扮演 VO 的角色。激活对象是在进入函数上下文时刻被创建的，它通过函数的 arguments 属性进行初始化

Arguments 对象是函数上下文里的激活对象  AO 中的内部对象，它包括下列属性：

- callee：指向当前函数的引用
- length：传递的参数的个数
- properties-indexes：函数的参数值（标识写法，这个不是真实的属性，按参数列表从左到右排列，索引初始值为 0）

对于 VO 和 AO 的关系可以理解为，VO 在不同的 Execution Context 中会有不同的表现：当在 Global Execution Context 中，可以直接使用 VO；但是，在函数 Execution Context 中，AO 就会被创建，如下图：

![image](https://article.biliimg.com/bfs/article/6c364ca21cb9bee724089b52987fa8276e7f57fc.png)

#### 5、调用栈调用过程

看一个例子，如下代码：

```
var name = 'curry'

console.log(message)

var message = 'I am a coder'

function foo() {
  var name = 'foo'
  console.log(name)
}

var num1 = 30
var num2 = 20

var result = num1 + num2

foo()
```

##### 1. 初始化全局对象

- 与普通变量有所不同，函数存放的是地址，会指向函数对象
- 从上往下解析 js 代码，当解析到 foo 函数时，因为 foo 不是普通变量，并不会赋为 undefined，js 引擎会在堆内存中开辟一块空间存放 foo 函数，在全局对象中引用其地址
- 这个开辟的函数存储空间最主要存放了该函数的 **父级作用域** 和函数的 **执行体代码块**

![image](https://article.biliimg.com/bfs/article/e03016b4827f5f93a1b5c47af5aea408126115fd.png)

##### 2. GEC 创建阶段

- 构建一个全局执行上下文（GEC），放到执行上下文调用栈（ECS）中，并将其 VO 的内存地址指向 GlobalObject（GO）

![image](https://article.biliimg.com/bfs/article/43454465d893fbb560ff04d6857089cc67681dfb.png)

##### 3. GEC 执行阶段

- 当执行 var name = 'curry' 时，就从 VO（对应的就是 GO）中找到 name 属性赋值为 curry
- 接下来执行 console.log(message)，就从 VO 中找到 message，此时的 message 还为 undefined，因为 message 真正赋值在下一行代码，所以就直接打印 undefined
- 后面就依次进行赋值，执行到 var result = num1 + num2，也是从 VO 中找到 num1 和 num2 两个属性的值进行相加，然后赋值给 result，result 最终就为 50
- 最后执行到 foo()，也就是需要去执行 foo 函数了，涉及到函数执行上下文，下面来看，截至目前呈以下状态：

![image](https://article.biliimg.com/bfs/article/6cebe4a87582f900e6fa6949f04c9f77e87e099c.png)

- 函数执行上下文
    - 先找到 foo 函数的存储地址，然后解析 foo 函数，生成函数的活动对象 AO
    - 根据 AO 生成函数执行上下文（FEC），并将其放入执行上下文栈（ECS）中
    - 前两步都属于创建阶段，此时函数内的变量还未赋值。接下来开始执行 foo 函数内代码，依次找到 AO 中的属性并赋值，当执行 console.log(name) 时，就会去 foo 的 VO（对应的就是 foo 函数的 AO）中找到 name 属性值并打印出来

![image](https://article.biliimg.com/bfs/article/11b77acde7543152f7215e385687ebcebeef0216.png)

- 至此，函数内所有代码执行完，FEC 出栈，函数的 AO 失去了引用，进行销毁，全局代码执行完成

#### 6、函数嵌套

如下代码：

```
var message = 'global'

function foo(m) {
  var message = 'foo'
  console.log(m)

  function bar() {
    console.log(message)
  }

  bar()
}

foo(30)
```

1. 初始化全局对象（GO），执行全局代码前创建 GEC，并将 GO 关联到 VO，然后将 GEC 加入 ECS 中，foo 函数存储空间中指定的父级作用域为全局对象

![image](https://article.biliimg.com/bfs/article/332abb626a5b82b875ae01477e9e312496eda116.png)

2. 开始执行全局代码，从上往下依次给全局属性赋值，给 message 属性赋值为 global

![image](https://article.biliimg.com/bfs/article/9bc6be75bda0584689af14f10632a33da1f10008.png)

3. 执行到 foo 函数调用，准备执行 foo 函数前，创建 foo 函数的 AO，bar 函数存储空间中指定父级作用域为 foo 函数的 AO

![image](https://article.biliimg.com/bfs/article/0866d72cb8664575dbf0db8eaa5fc419399631c3.png)

4. 创建 foo 函数的 FEC，并加入到 ECS 中，然后开始执行 foo 函数体内的代码

![image](https://article.biliimg.com/bfs/article/701919452fe7d4a6a05e6d3087de99d434f261b9.png)

5. 执行到 bar 函数调用，准备执行 bar 函数前，创建 bar 函数的 AO（bar 函数中没有定义属性和声明函数，以空对象表示）

![image](https://article.biliimg.com/bfs/article/f9d0e56609360961544e6905b43f8e3ec8504007.png)

6. 创建 bar 函数的FEC，并加入到 ECS 中，然后开始执行 bar 函数体内的代码，

![image](https://article.biliimg.com/bfs/article/4320d92298d8c43eb673e8ff067749dad1edc058.png)

7. 最后，全局中所有代码执行完成，bar 函数执行上下文出栈，foo 函数 AO 对象失去了引用，进行销毁。接着 foo 函数执行上下文出栈，foo 函数 AO 对象失去了引用，进行销毁，同样，foo 函数 AO 对象销毁后，bar 函数的存储空间也失去引用，进行销毁

#### 7、细看 Execution Context（EC）

当一段 js 代码执行的时候，js 解释器会创建 Execution Context，会有两个阶段：

- 创建阶段（当函数被调用，但是开始执行函数内部代码之前）
    - 创建Scope chain
    - 创建VO/AO（variables, functions and arguments）
    - 设置this的值
- 执行阶段
    - 设置变量的值、函数的引用，然后解释/执行代码

这里想要详细介绍一下"创建VO/AO"中的一些细节，因为这些内容将直接影响代码运行的行为

对于"创建VO/AO"这一步，JavaScript解释器主要做了下面的事情：

- 根据函数的参数，创建并初始化 arguments object
- 扫描函数内部代码，查找函数声明（Function declaration）
    - 对于所有找到的函数声明，将函数名和函数引用存入 VO/AO 中
    - **如果 VO/AO 中已经有同名的函数，那么就进行覆盖**
- 扫描函数内部代码，查找变量声明（Variable declaration）
    - 对于所有找到的变量声明，将变量名存入 VO/AO 中，并初始化为 "undefined"
    - 如果变量名称跟已经声明的形参或函数相同，**则变量声明不会干扰已经存在的这类属性**（创建阶段）

#### 8、关于上下文的例子

```
(function(){
    console.log(foo);
    console.log(bar);
    console.log(baz);
    
    var foo = function(){};
    
    function bar(){
        console.log("bar");
    }
    
    var bar = 20;
    console.log(bar);
    
    function baz(){
        console.log("baz");
    }
})()
```

代码的运行结果为：

![image](https://article.biliimg.com/bfs/article/14a94823c33e991d5264fa7fa15d8a885e01b9a7.png)

### 二、作用域链

#### 1、分析一段代码

```
var scope = "global scope"
function checkscope(){
    var scope = "local scope"
    function f(){
        return scope
    }
    return f()
}
checkscope()
```

1. 执行全局代码，创建全局执行上下文，全局上下文被压入执行上下文栈

```
ECStack = [
  globalContext
]
```

2. 全局上下文初始化

```
globalContext = {
  VO: [global],
  Scope: [globalContext.VO],
  this: globalContext.VO
}
```

3. 初始化的同时，checkscope 函数被创建，保存作用域链到函数的内部属性 [[scope]]

```
checkscope.[[scope]] = [
  globalContext.VO
]
```

4. 执行 checkscope 函数，创建 checkscope 函数执行上下文，checkscope 函数执行上下文被压入执行上下文栈

```
ECStack = [
  checkscopeContext,
  globalContext
]
```

5. checkscope 函数执行上下文初始化
    - 复制函数 [[scope]] 属性创建作用域链
    - 初始化活动对象，即加入形参、函数声明、变量声明
    - 将活动对象压入 checkscope 作用域链顶端

同时 f 函数被创建，保存作用域链到 f 函数的内部属性 [[scope]]

```
checkscopeContext = {
  AO: {
    arguments: {
        length: 0
    },
    scope: undefined,
    f: reference to function f(){}
  },
  Scope: [AO, globalContext.VO],
  this: undefined
}
```

6. 执行 f 函数，创建 f 函数执行上下文，f 函数执行上下文被压入执行上下文栈

```
ECStack = [
  fContext,
  checkscopeContext,
  globalContext
]
```

7. f 函数执行上下文初始化, 跟第 5 步相同

```
fContext = {
  AO: {
    arguments: {
      length: 0
    }
  },
  Scope: [AO, checkscopeContext.AO, globalContext.VO],
  this: undefined
}
```

8. f 函数执行，沿着作用域链查找 scope 值，返回 scope 值

9. f 函数执行完毕，f 函数上下文从执行上下文栈中弹出

```
ECStack = [
  checkscopeContext,
  globalContext
]
```

10. checkscope 函数执行完毕，checkscope 执行上下文从执行上下文栈中弹出

```
ECStack = [
  globalContext
]
```

#### 2、作用域例子

看了上面的流程后，下面两段代码执行的结果分别是什么？

```
var scope = "global scope"
function checkscope(){
    var scope = "local scope"
    function f(){
        return scope
    }
    return f()
}
checkscope()
```

```
var scope = "global scope"
function checkscope(){
    var scope = "local scope"
    function f(){
        return scope
    }
    return f
}
checkscope()()
```

两段代码都会打印 "local scope"，不解释


#### 3、闭包

MDN 对闭包的定义为：

> 闭包是指那些能够访问自由变量的函数

什么是自由变量呢？

> 自由变量是指在函数中使用的，但既不是函数参数也不是函数的局部变量的变量

由此，我们可以看出闭包共有两部分组成：

> 闭包 = 函数 + 函数能够访问的自由变量

例如：

```
var a = 1

function foo() {
    console.log(a)
}

foo()
```

foo 函数可以访问变量 a，但是 a 既不是 foo 函数的局部变量，也不是 foo 函数的参数，所以 a 就是自由变量。

那么，函数 foo + foo 函数访问的自由变量 a 不就是构成了一个闭包嘛。。。

确实如此，从技术的角度讲，所有的 JavaScript 函数都是闭包

这是理论上的闭包，其实还有一个实践角度上的闭包，我们平时说的闭包一般是指实践角度的闭包

- 从理论角度：所有的函数。因为它们都在创建的时候就将上层上下文的数据保存起来了。哪怕是简单的全局变量也是如此，因为函数中访问全局变量就相当于是在访问自由变量，这个时候使用最外层的作用域
- 从实践角度：以下函数才算是闭包
    - 即使创建它的上下文已经销毁，它仍然存在（比如，内部函数从父函数中返回）
    - 在代码中引用了自由变量

##### 从作用域链的角度看闭包

还是以下例子：

```
var scope = "global scope"
function checkscope(){
    var scope = "local scope"
    function f(){
        return scope
    }
    return f
}

var foo = checkscope()
foo()
```

我们知道，当 f 函数执行的时候，checkscope 函数上下文已经被销毁了啊(即从执行上下文栈中被弹出)，怎么还会读取到 checkscope 作用域下的 scope 值呢？

当我们了解了具体的执行过程后，我们知道 f 执行上下文维护了一个作用域链：

```
fContext = {
  Scope: [AO, checkscopeContext.AO, globalContext.VO],
}
```

就是因为这个作用域链，f 函数依然可以读取到 checkscopeContext.AO 的值，说明当 f 函数引用了 checkscopeContext.AO 中的值的时候，即使 checkscopeContext 被销毁了，但是 JavaScript 依然会让 checkscopeContext.AO 活在内存中，f 函数依然可以通过 f 函数的作用域链找到它，JavaScript 做到了让作用域链上的 AO 不被销毁，从而实现了闭包

##### 闭包案例

又是经典面试题：

```
var data = []

for (var i = 0; i < 3; i++) {
  data[i] = function () {
    console.log(i)
  };
}

data[0]()
data[1]()
data[2]()
```

如何让以上执行打出 1 2 3，只需做以下改造即可：

```
var data = []

for (var i = 0; i < 3; i++) {
  data[i] = (function (i) {
        return function(){
            console.log(i)
        }
  })(i)
}

data[0]()
data[1]()
data[2]()
```

通过分析上下文执行过程，就能清楚的看到以上两组代码最终形成作用域链的不同：

```
// 修改前
Scope: [AO, globalContext.VO]

// 修改后
Scope: [AO, 匿名函数Context.AO, globalContext.VO]
```

### 三、this

需要从 ECMAScript 规范类型（描述语言底层行为逻辑，并不存在于实际的 js 代码中） Reference 开始讲起，有点复杂，不是重点，有机会再说

### 四、es2015+

上面讲了半天执行上下文，其实是 ES3 的内容。。。，
虽然现在 ES 规范和以前不同，有很多概念上的改变，以前的概念现在可能已经没有了，但是 JavaScript 的执行过程还是不变的，使用 ES3 的概念，我个人感觉更容易理解，接下来说说 es6+ 后的改变

#### 执行上下文

执行上下文在创建阶段会创建以下两个环境:

- Lexical Environment（词法环境）
- Variable Environment（变量环境）

#### 1、词法环境

简单来说，词法环境就是一个保存标识符-变量映射的结构，这里标识符指的是变量/函数的名称，变量是对实际对象[包括函数对象和数组对象]或原始数据的引用。词法环境也就对应着 ES3 中 执行上下文（EC）的结构

每个词法环境有三个组成部分：

- Environment Record（环境记录器，对应 VO）
- Reference to the outer environment（指向外部环境的引用，对应 scope chain）
- This binding（this 绑定）

##### 环境记录器

环境记录器是变量和函数声明存储在词法环境中的位置，环境记录器有两类:

- 声明性环境记录（Declarative environment record）：存储变量和函数声明。函数代码的词法环境包含一个声明性环境记录（对应 AO）
- 对象环境记录（Object environment record）：全局代码（global code）的词法环境包含一个对象环境记录。除了变量和函数声明，对象环境记录还存储了一个全局绑定 window 对象（对应 GO）

##### 外部环境引用

是它能够接触到的外部的词法环境，如果在当前词法环境中没有找到想要查找的变量，则可以在外部环境中查找它们，也就是作用域链的概念

##### this

在全局执行上下文中，this 的值指向全局对象，在浏览器中，它指的就是 Window 对象

在函数执行上下文中，this 的值取决于函数的调用方式。如果它是通过对象引用调用的，那么 this 的值被设置为该对象，否则，this 的值被设置为全局对象

#### 2、变量环境

也是一个词法环境，它的环境记录器保存由变量声明在执行上下文中创建的绑定

由于变量环境也是一个词法环境，因此它具有上述定义的词法环境的所有属性

在 ES6 中，词法环境（LexicalEnvironment）组件和变量环境（VariableEnvironment）组件之间其中一个区别是，前者用于存储函数声明和变量（let 和 const）绑定，而后者仅用于存储变量（var）绑定

#### 3、例子

```
let a = 20
const b = 30
var c
function mul(e, f) {
  var g = 20
  return e * f * g
}
c = mul(20, 30)
```

当执行上述代码时，js 引擎创建一个全局执行上下文来执行全局代码。在创建阶段，全局执行上下文伪代码：

```
GlobalExectionContext = {
  LexicalEnvironment: {
    EnvironmentRecord: {
      Type: "Object",
      a: < uninitialized > ,
      b: < uninitialized > ,
      mul: < ref. func >
    }
    outer: < null > ,
    ThisBinding: < Global Object >
  },
  VariableEnvironment: {
    EnvironmentRecord: {
      Type: "Object",
      c: undefined,
    }
    outer: < null > ,
    ThisBinding: < Global Object >
  }
}
```

在执行阶段，完成变量赋值，还未执行函数，伪代码：

```
GlobalExectionContext = {
  LexicalEnvironment: {
    EnvironmentRecord: {
      Type: "Object",
      a: 20,
      b: 30,
      mul: < ref. func >
    }
    outer: <null>,
    ThisBinding: <Global Object>
  },
  VariableEnvironment: {
    EnvironmentRecord: {
      Type: "Object",
      c: undefined,
    }
    outer: <null>,
    ThisBinding: <Global Object>
  }
}
```

当遇到对 function mul(20, 30) 的调用时，将创建一个新的函数执行上下文来执行函数代码，在创建阶段，函数执行上下文伪代码:

```
FunctionExectionContext = {
  LexicalEnvironment: {
    EnvironmentRecord: {
      Type: "Declarative",
      Arguments: { 0: 20, 1: 30, length: 2 },
    },
    outer: <Global Lexical Environment>,
    ThisBinding: <Global Object>,
  },
  VariableEnvironment: {
    EnvironmentRecord: {
      Type: "Declarative",
      g: undefined
    },
    outer: <Global Lexical Environment>,
    ThisBinding: <Global Object>
  }
}
```

函数执行阶段，完成对函数内变量的赋值，在执行阶段，函数的执行上下文伪代码：

```
FunctionExectionContext = {
  LexicalEnvironment: {
    EnvironmentRecord: {
      Type: "Declarative",
      Arguments: { 0: 20, 1: 30, length: 2 },
    },
    outer: <Global Lexical Environment>,
    ThisBinding: <Global Object>,
  },
  VariableEnvironment: {
    EnvironmentRecord: {
      Type: "Declarative",
      g: 20
    },
    outer: <Global Lexical Environment>,
    ThisBinding: <Global Object>
  }
}
```

函数执行完成后，返回值赋值给了全局词法环境中的变量 c，函数执行上下文被弹出，全局代码完成，程序结束

在创建阶段，代码被扫描以查找变量和函数声明，函数声明被完整地存储在环境中，对于 var，被设置为 undefined，对于 let，则为未初始化

当 let 在执行阶段，如果无法在其代码中声明的位置得到它的值，那么它将被赋值为 undefined，而对于 const 如果不赋值，则会直接报错

### 总结

虽然知道了以上过程，并不能直接在工作中使用，但理解以上概念还是有助于我们更容易、更深入地理解其他常说的概念，就比如 变量提升、作用域链、闭包、上下文 等等。另外就是会对 js 的执行过程心里大概有个数，使执行的过程更加具象化，遇到相关的问题，也能让分析更加透明

**-- 以上 --**

<br><br><br>


