> 当前文件处在活跃的开发状态。最新的描述请查看 `interop/reusability.md`。

# 概述
装饰器使我们在设计类和属性时注释和改变它们成为可能。

尽管ES5 对象字面量字段的值支持任意的表达式，但 ES6 的类只支持作为值得函数字面量。装饰器使 ES6 重新拥有了在设计时运行代码的能力，同时保持声明式语法。

# 实现细节

一个装饰器是：

* 一个表达式
* 对函数进行求值运算
* 接受目标、名称和装饰器描述作为参数
* 返回准备安装到目标对象上的装饰器的描述[可选]

考虑定义一个简单的类：

```js
class Person {
  name() { return `${this.first} ${this.last}` }
}
```

对这个类进行求值会把 `name` 函数安装到 `Person.prototype` 上，大致如下：

```js
Object.defineProperty(Person.prototype, 'name', {
  value: specifiedFunction,
  enumerable: false,
  configurable: true,
  writable: true
});
```

装饰器放在属性定义之前：

```js
class Person {
  @readonly
  name() { return `${this.first} ${this.last}` }
}
```

现在，在安装描述符到 `Person.prototype` 之前，会调用装饰器：

```js
let description = {
  type: 'method',
  initializer: () => specifiedFunction,
  enumerable: false,
  configurable: true,
  writable: true
};

description = readonly(Person.prototype, 'name', description) || description;
defineDecoratedProperty(Person.prototype, 'name', description);

function defineDecoratedProperty(target, { initializer, enumerable, configurable, writable }) {
  Object.defineProperty(target, { value: initializer(), enumerable, configurable, writable });
}
```

现在在相应的 `defineProperty` 真正调用之前，我们有机会改变类的属性。

装饰器放在 getters 和/或 setters 之前影响属性存取操作：

```js
class Person {
  @nonenumerable
  get kidCount() { return this.children.length; }
}

let description = {
  type: 'accessor',
  get: specifiedGetter,
  enumerable: true,
  configurable: true
}

function nonenumerable(target, name, description) {
  descriptor.enumerable = false;
  return descriptor;
}
```

一个更详细的例子演示一个记住存取器的简单装饰器：

```js
class Person {
  @memoize
  get name() { return `${this.first} ${this.last}` }
  set name(val) {
    let [first, last] = val.split(' ');
    this.first = first;
    this.last = last;
  }
}

let memoized = new WeakMap();
function memoize(target, name, descriptor) {
  let getter = descriptor.get, setter = descriptor.set;

  descriptor.get = function() {
    let table = memoizationFor(this);
    if (name in table) { return table[name]; }
    return table[name] = getter.call(this);
  }

  descriptor.set = function(val) {
    let table = memoizationFor(this);
    setter.call(this, val);
    table[name] = val;
  }
}

function memoizationFor(obj) {
  let table = memoized.get(obj);
  if (!table) { table = Object.create(null); memoized.set(obj, table); }
  return table;
}
```

装饰类也是可以的。这种情况下，装饰器接受目标构造器作为参数。

```js
// A simple decorator
@annotation
class MyClass { }

function annotation(target) {
   // Add a property on target
   target.annotated = true;
}
```

因为装饰器是表达式，所以装饰器可以像工厂函数一样使用接受额外的参数。

```js
@isTestable(true)
class MyClass { }

function isTestable(value) {
   return function decorator(target) {
      target.isTestable = value;
   }
}
```

属性装饰器也可以这样用：

```js
class C {
  @enumerable(false)
  method() { }
}

function enumerable(value) {
  return function (target, key, descriptor) {
     descriptor.enumerable = value;
     return descriptor;
  }
}
```

Because descriptor decorators operate on targets, they also naturally work on
static methods. The only difference is that the first argument to the decorator
will be the class itself (the constructor) rather than the prototype, because
that is the target of the original `Object.defineProperty`.
因为描述符装饰器作用于目标，它们自然也可以应用在静态方法上。唯一的区别就是传入装饰器的第一个参数将会是类本身（构造器）而不是原型，因为原型是原始的 `Object.defineProperty` 的目标。


基于同样的原因，描述符装饰器也可以用于对象字面量，把需要被创建的对象传递给装饰器。

# 去糖 

## 申明类 

### 语法

```js
@F("color")
@G
class Foo {
}
```

### 去糖 (ES6)

```js
var Foo = (function () {
  class Foo {
  }

  Foo = F("color")(Foo = G(Foo) || Foo) || Foo;
  return Foo;
})();
```

### 去糖 (ES5)

```js
var Foo = (function () {
  function Foo() {
  }

  Foo = F("color")(Foo = G(Foo) || Foo) || Foo;
  return Foo;
})();
```

## 定义类的方法 

### 语法 

```js
class Foo {
  @F("color")
  @G
  bar() { }
}
```

###  (ES6)

```js
var Foo = (function () {
  class Foo {
    bar() { }
  }

  var _temp;
  _temp = F("color")(Foo.prototype, "bar",
    _temp = G(Foo.prototype, "bar",
      _temp = Object.getOwnPropertyDescriptor(Foo.prototype, "bar")) || _temp) || _temp;
  if (_temp) Object.defineProperty(Foo.prototype, "bar", _temp);
  return Foo;
})();
```

### 去糖 (ES5)

```js
var Foo = (function () {
  function Foo() {
  }
  Foo.prototype.bar = function () { }

  var _temp;
  _temp = F("color")(Foo.prototype, "bar",
    _temp = G(Foo.prototype, "bar",
      _temp = Object.getOwnPropertyDescriptor(Foo.prototype, "bar")) || _temp) || _temp;
  if (_temp) Object.defineProperty(Foo.prototype, "bar", _temp);
  return Foo;
})();
```

## 定义类的存储器 

### 语法 

```js
class Foo {
  @F("color")
  @G
  get bar() { }
  set bar(value) { }
}
```

### 去糖 (ES6)

```js
var Foo = (function () {
  class Foo {
    get bar() { }
    set bar(value) { }
  }

  var _temp;
  _temp = F("color")(Foo.prototype, "bar",
    _temp = G(Foo.prototype, "bar",
      _temp = Object.getOwnPropertyDescriptor(Foo.prototype, "bar")) || _temp) || _temp;
  if (_temp) Object.defineProperty(Foo.prototype, "bar", _temp);
  return Foo;
})();
```

### 去糖 (ES5)

```js
var Foo = (function () {
  function Foo() {
  }
  Object.defineProperty(Foo.prototype, "bar", {
    get: function () { },
    set: function (value) { },
    enumerable: true, configurable: true
  });

  var _temp;
  _temp = F("color")(Foo.prototype, "bar",
    _temp = G(Foo.prototype, "bar",
      _temp = Object.getOwnPropertyDescriptor(Foo.prototype, "bar")) || _temp) || _temp;
  if (_temp) Object.defineProperty(Foo.prototype, "bar", _temp);
  return Foo;
})();
```

### 定义对象字面量的方法

### 语法 

```js
var o = {
  @F("color")
  @G
  bar() { }
}
```

### 去糖 (ES6)

```js
var o = (function () {
  var _obj = {
    bar() { }
  }

  var _temp;
  _temp = F("color")(_obj, "bar",
    _temp = G(_obj, "bar",
      _temp = void 0) || _temp) || _temp;
  if (_temp) Object.defineProperty(_obj, "bar", _temp);
  return _obj;
})();
```

###  (ES5)

```js
var o = (function () {
    var _obj = {
        bar: function () { }
    }

    var _temp;
    _temp = F("color")(_obj, "bar",
        _temp = G(_obj, "bar",
            _temp = void 0) || _temp) || _temp;
    if (_temp) Object.defineProperty(_obj, "bar", _temp);
    return _obj;
})();
```

## Object Literal Accessor Declaration

### Syntax

```js
var o = {
  @F("color")
  @G
  get bar() { }
  set bar(value) { }
}
```

### Desugaring (ES6)

```js
var o = (function () {
  var _obj = {
    get bar() { }
    set bar(value) { }
  }

  var _temp;
  _temp = F("color")(_obj, "bar",
    _temp = G(_obj, "bar",
      _temp = void 0) || _temp) || _temp;
  if (_temp) Object.defineProperty(_obj, "bar", _temp);
  return _obj;
})();
```

### Desugaring (ES5)

```js
var o = (function () {
  var _obj = {
  }
  Object.defineProperty(_obj, "bar", {
    get: function () { },
    set: function (value) { },
    enumerable: true, configurable: true
  });

  var _temp;
  _temp = F("color")(_obj, "bar",
    _temp = G(_obj, "bar",
      _temp = void 0) || _temp) || _temp;
  if (_temp) Object.defineProperty(_obj, "bar", _temp);
  return _obj;
})();
```

# Grammar

&emsp;&emsp;*DecoratorList*<sub> [Yield]</sub>&emsp;:  
&emsp;&emsp;&emsp;*DecoratorList*<sub> [?Yield]opt</sub>&emsp; *Decorator*<sub> [?Yield]</sub>

&emsp;&emsp;*Decorator*<sub> [Yield]</sub>&emsp;:  
&emsp;&emsp;&emsp;`@`&emsp;*LeftHandSideExpression*<sub> [?Yield]</sub>

&emsp;&emsp;*PropertyDefinition*<sub> [Yield]</sub>&emsp;:  
&emsp;&emsp;&emsp;*IdentifierReference*<sub> [?Yield]</sub>  
&emsp;&emsp;&emsp;*CoverInitializedName*<sub> [?Yield]</sub>  
&emsp;&emsp;&emsp;*PropertyName*<sub> [?Yield]</sub>&emsp; `:`&emsp;*AssignmentExpression*<sub> [In, ?Yield]</sub>  
&emsp;&emsp;&emsp;*DecoratorList*<sub> [?Yield]opt</sub>&emsp;*MethodDefinition*<sub> [?Yield]</sub>

&emsp;&emsp;*CoverMemberExpressionSquareBracketsAndComputedPropertyName*<sub> [Yield]</sub>&emsp;:  
&emsp;&emsp;&emsp;`[`&emsp;*Expression*<sub> [In, ?Yield]</sub>&emsp;`]`

NOTE	The production *CoverMemberExpressionSquareBracketsAndComputedPropertyName* is used to cover parsing a *MemberExpression* that is part of a *Decorator* inside of an *ObjectLiteral* or *ClassBody*, to avoid lookahead when parsing a decorator against a *ComputedPropertyName*. 

&emsp;&emsp;*PropertyName*<sub> [Yield, GeneratorParameter]</sub>&emsp;:  
&emsp;&emsp;&emsp;*LiteralPropertyName*  
&emsp;&emsp;&emsp;[+GeneratorParameter] *CoverMemberExpressionSquareBracketsAndComputedPropertyName*  
&emsp;&emsp;&emsp;[~GeneratorParameter] *CoverMemberExpressionSquareBracketsAndComputedPropertyName*<sub> [?Yield]</sub>

&emsp;&emsp;*MemberExpression*<sub> [Yield]</sub>&emsp; :  
&emsp;&emsp;&emsp;[Lexical goal *InputElementRegExp*] *PrimaryExpression*<sub> [?Yield]</sub>  
&emsp;&emsp;&emsp;*MemberExpression*<sub> [?Yield]</sub>&emsp;*CoverMemberExpressionSquareBracketsAndComputedPropertyName*<sub> [?Yield]</sub>  
&emsp;&emsp;&emsp;*MemberExpression*<sub> [?Yield]</sub>&emsp;`.`&emsp;*IdentifierName*  
&emsp;&emsp;&emsp;*MemberExpression*<sub> [?Yield]</sub>&emsp;*TemplateLiteral*<sub> [?Yield]</sub>  
&emsp;&emsp;&emsp;*SuperProperty*<sub> [?Yield]</sub>  
&emsp;&emsp;&emsp;*NewSuper*&emsp;*Arguments*<sub> [?Yield]</sub>  
&emsp;&emsp;&emsp;`new`&emsp;*MemberExpression*<sub> [?Yield]</sub>&emsp;*Arguments*<sub> [?Yield]</sub>

&emsp;&emsp;*SuperProperty*<sub> [Yield]</sub>&emsp;:  
&emsp;&emsp;&emsp;`super`&emsp;*CoverMemberExpressionSquareBracketsAndComputedPropertyName*<sub> [?Yield]</sub>  
&emsp;&emsp;&emsp;`super`&emsp;`.`&emsp;*IdentifierName*

&emsp;&emsp;*ClassDeclaration*<sub> [Yield, Default]</sub>&emsp;:  
&emsp;&emsp;&emsp;*DecoratorList*<sub> [?Yield]opt</sub>&emsp;`class`&emsp;*BindingIdentifier*<sub> [?Yield]</sub>&emsp;*ClassTail*<sub> [?Yield]</sub>  
&emsp;&emsp;&emsp;[+Default] *DecoratorList*<sub> [?Yield]opt</sub>&emsp;`class`&emsp;*ClassTail*<sub> [?Yield]</sub>

&emsp;&emsp;*ClassExpression*<sub> [Yield, GeneratorParameter]</sub>&emsp;:  
&emsp;&emsp;&emsp;*DecoratorList*<sub> [?Yield]opt</sub>&emsp;`class`&emsp;*BindingIdentifier*<sub> [?Yield]opt</sub>&emsp;*ClassTail*<sub> [?Yield, ?GeneratorParameter]</sub>

&emsp;&emsp;*ClassElement*<sub> [Yield]</sub>&emsp;:  
&emsp;&emsp;&emsp;*DecoratorList*<sub> [?Yield]opt</sub>&emsp;*MethodDefinition*<sub> [?Yield]</sub>  
&emsp;&emsp;&emsp;*DecoratorList*<sub> [?Yield]opt</sub>&emsp;`static`&emsp;*MethodDefinition*<sub> [?Yield]</sub>

# Notes

In order to more directly support metadata-only decorators, a desired feature
for static analysis, the TypeScript project has made it possible for its users
to define [ambient decorators](https://github.com/jonathandturner/brainstorming/blob/master/README.md#c6-ambient-decorators)
that support a restricted syntax that can be properly analyzed without evaluation.
