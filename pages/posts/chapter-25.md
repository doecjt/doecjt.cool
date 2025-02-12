﻿---
title: 第25章—装饰器与反射元数据：了解装饰器基本原理与应用
date: 2021-05-19T16:25:00.000+00:00
lang: zh
duration: 10min
type: note
---

上一节我们了解了 TypeScript 与 ECMAScript 的关系，以及可选链与空值合并这两个 TypeScript 中的 ECMAScript 提案。其实，还有一个 ECMAScript 提案也已经成为 TypeScript 中相当重要的一部分，它就是装饰器。

装饰器语法在 Python、Java 等语言中都能见到，但在 JavaScript 中并没有被大量使用。一方面是因为，装饰器其实还不能被称为 JavaScript 的一部分，另一方面则是它对应用场景有着一定要求，比如只能使用在 Class 上，而 Class 并不是 JavaScript 中大量使用的语法。

至于为什么说装饰器还不是 JavaScript 的一部分，我们会在扩展阅读中介绍更多。这一节我们只关注 TypeScript 中的装饰器，从基础语法到不同种类的装饰器，从反射到反射元数据，再到基于这些概念实现依赖注入、IoC 容器等等。

> 本节代码见：[Decorators](https://github.com/linbudu599/TypeScript-Tiny-Book/tree/main/packages/21-decorators)

首先我们需要明确的是，**装饰器的本质其实就是一个函数**，只不过它的入参是提前确定好的。同时，TypeScript 中的装饰器目前**只能在类以及类成员上使用**。

装饰器通过 `@` 语法来使用：

```typescript
function Deco() { }

@Deco
class Foo {}
```

这样的装饰器只能起到固定的功能，我们实际上使用更多的是 Decorator Factory，即装饰器工厂的形式：

```typescript
function Deco() {
  return () => {}
}

@Deco()
class Foo {}
```

在这种情况下，程序执行时会先执行 `Deco()` ，再用内部返回的函数作为装饰器的实际逻辑。这样，我们就可以通过入参来灵活地调整装饰器的作用。接下来，我们就来学习一下 TypeScript 中的装饰器是如何使用的，它们分别有什么作用？

## 装饰器大起底

TypeScript 中的装饰器可以分为**类装饰器**、**方法装饰器**、**访问符装饰器**、**属性装饰器**以及**参数装饰器**五种，最常见的主要还是类装饰器、方法装饰器以及属性装饰器。接下来，我们会依次介绍这几种装饰器的具体使用。

### 类装饰器

类装饰器是直接作用在类上的装饰器，它在执行时的入参只有一个，那就是这个类本身（而不是类的原型对象）。因此，我们可以通过类装饰器来覆盖类的属性与方法，如果你在类装饰器中返回一个新的类，它甚至可以篡改掉整个类的实现。

```typescript
@AddProperty('linbudu')
@AddMethod()
class Foo {
  a = 1;
}

function AddMethod(): ClassDecorator {
  return (target: any) => {
    target.prototype.newInstanceMethod = () => {
      console.log("Let's add a new instance method!");
    };
    target.newStaticMethod = () => {
      console.log("Let's add a new static method!");
    };
  };
}

function AddProperty(value: string): ClassDecorator {
  return (target: any) => {
    target.prototype.newInstanceProperty = value;
    target.newStaticProperty = `static ${value}`;
  };
}
```

这里，我们通过 TypeScript 内置的 ClassDecorator 类型定义来进行类型标注，由于类装饰器只有一个参数，我们也不想使用过多的类型代码，这里我就直接 any 了。我们的函数返回了一个 ClassDecorator ，因此这个装饰器就是一个 Decorator Factory，在实际执行时需要以 `@Deco()` 的形式调用。

在 AddMethod 与 AddProperty 方法中，我们分别在 target、`target.prototype` 上添加了方法与属性，还记得 ES6 中 Class 的本质仍然是基于原型的吗？在这里 target 上的属性实际上是**静态成员**，也就是其实例上不会获得的方法，而 `target.prototype` 上的属性才是会随着继承与实例化过程被传递的**实例成员**。

我们来调用一下看看：

```typescript
const foo: any = new Foo();

foo.newInstanceMethod();
(<any>Foo).newStaticMethod();

console.log(foo.newInstanceProperty);
console.log((<any>Foo).newStaticProperty);
```

```text
Let's add a new instance method!
Let's add a new static method!
linbudu
static linbudu
```

我们在这里调用的方法并没有直接在 Foo 中定义，而是通过装饰器来强行添加！我们也可以在装饰中返回一个子类：

```typescript
const OverrideBar = (target: any) => {
  return class extends target {
    print() {}
    overridedPrint() {
      console.log('This is Overrided Bar!');
    }
  };
};

@OverrideBar
class Bar {
  print() {
    console.log('This is Bar!');
  }
}

// 被覆盖了，现在是一个空方法
new Bar().print();

// This is Overrided Bar!
(<any>new Bar()).overridedPrint();
```

在 React Class 组件时代，其实你会发现有许多功能也是通过装饰器实现的。如 Mobx 的 `@observer` 与 `@observable`，React-Redux 的 `@connect` 等。

### 方法装饰器

方法装饰器的入参包括**类的原型**、**方法名**以及**方法的属性描述符**（PropertyDescriptor），而通过属性描述符你可以控制这个方法的内部实现（即 value）、可变性（即 writable）等信息。

能拿到原本实现，也就意味着，我们可以在执行原本方法的同时，插入一段新的逻辑，比如计算这个方法的执行耗时：

```typescript
class Foo {
  @ComputeProfiler()
  async fetch() {
    return new Promise((resolve, reject) => {
      setTimeout(() => {
        resolve('RES');
      }, 3000);
    });
  }
}

function ComputeProfiler(): MethodDecorator {
  return (
    _target,
    methodIdentifier,
    descriptor: TypedPropertyDescriptor<any>
  ) => {
    const originalMethodImpl = descriptor.value!;
    descriptor.value = async function (...args: unknown[]) {
      const start = new Date();
      const res = await originalMethodImpl.apply(this, args); // 执行原本的逻辑
      const end = new Date();
      console.log(
        `${String(methodIdentifier)} Time: `,
        end.getTime() - start.getTime()
      );
      return res;
    };
  };
}

(async () => {
  console.log(await new Foo().fetch());
})();
```

```text
fetch Time:  3003
RES
```

需要注意的是，方法装饰器的 target 是**类的原型而非类本身**。

### 访问符装饰器

访问符装饰器并不常见，甚至访问符对于部分同学来说也是陌生的，但它其实就是 `get value(){}` 与 `set value(v)=>{}` 这样的方法，其中 getter 在你访问这个属性 `value` 时触发，而 setter 在你对 `value` 进行赋值时触发。访问符装饰器本质上仍然是方法装饰器，它们使用的类型定义也相同。

需要注意的是，访问符装饰器只能同时应用在一对 getter / setter 的其中一个，即要么装饰 getter 要么装饰 setter 。这是因为，不论你是装饰哪一个，装饰器入参中的属性描述符都会包括 getter 与setter 方法：

```typescript
class Foo {
  _value!: string;

  get value() {
    return this._value;
  }

  @HijackSetter('LIN_BU_DU')
  set value(input: string) {
    this._value = input;
  }
}

function HijackSetter(val: string): MethodDecorator {
  return (target, methodIdentifier, descriptor: any) => {
    const originalSetter = descriptor.set;
    descriptor.set = function (newValue: string) {
      const composed = `Raw: ${newValue}, Actual: ${val}-${newValue}`
      originalSetter.call(this, composed);
      console.log(`HijackSetter: ${composed}`);
    };
    // 篡改 getter，使得这个值无视 setter 的更新，返回一个固定的值
    // descriptor.get = function () {
    //   return val;
    // };
  };
}

const foo = new Foo();
foo.value = 'LINBUDU'; // HijackSetter: Raw: LINBUDU, Actual: LIN_BU_DU-LINBUDU
```

在这个例子中，我们通过装饰器劫持了 setter ，在执行原本的 setter 方法修改了其参数。同时，我们也可以在这里去劫持 getter（`descriptor.get`），这样一来在读取这个值时，会直接返回一个我们固定好的值，而非其实际的值（如被 setter 更新过的）。

### 属性装饰器

属性装饰器在独立使用时能力非常有限，它的入参只有**类的原型**与**属性名称**，返回值会被忽略，但你仍然可以通过**直接在类的原型上赋值**来修改属性：

```typescript
class Foo {
  @ModifyNickName()
  nickName!: string;
  constructor() {}
}

function ModifyNickName(): PropertyDecorator {
  return (target: any, propertyIdentifier) => {
    target[propertyIdentifier] = '林不渡!';
    target['otherName'] = '别名林不渡!';
  };
}

console.log(new Foo().nickName);
// @ts-expect-error
console.log(new Foo().otherName);
```

```text
林不渡!
别名林不渡!
```

我们在原型对象上强行写入了属性，但这种方法实际上过于 hack，在后面我们会了解如何通过委托的方式来为一个属性注入值。

### 参数装饰器

参数装饰器包括了构造函数的参数装饰器与方法的参数装饰器，它的入参包括**类的原型**、**参数所在的方法名**与**参数在函数参数中的索引值（即第几个参数）**，如果只是单独使用，它的作用同样非常有限。

```typescript
class Foo {
  handler(@CheckParam() input: string) {
    console.log(input);
  }
}

function CheckParam(): ParameterDecorator {
  return (target, methodIdentifier, index) => {
    console.log(target, methodIdentifier, index);
  };
}

// {} handler 0
new Foo().handler('linbudu');
```

后面我们会了解如何基于参数装饰器进行参数的默认值注入与校验，现在就先到这儿，思考另一个问题：一个类中可以同时拥有这几种装饰器，那么这些**不同装饰器的执行时机与顺序是如何的**？

### 装饰器的执行机制

装饰器的执行机制中主要包括**执行时机**、**执行原理**以及**执行顺序**这三个概念。

首先是执行时机，还记得我们在最开始说的吗？装饰器的本质就是一个函数，因此只要在类上定义了它，即使不去实例化这个类或者读取静态成员，它也会正常执行。很多时候，其实我们也并不会实例化具有装饰器的类，而是通过反射元数据的能力来消费，这一点我们后面会讲到。而装饰器的执行原理，我们可以通过编译后的代码来了解：

```typescript
@Cls()
class Foo {
  constructor(@Param() init?: string) { }

  @Prop()
  prop!: string

  @Method()
  handler(@Param() input: string) {

  }
}
```

这一段代码编译的产物会是这样的（经过简化）：

```javascript
"use strict";
var __decorate = (this && this.__decorate) || function (decorators, target, key, desc) {
   // ...
};
var __param = (this && this.__param) || function (paramIndex, decorator) {
    return function (target, key) { decorator(target, key, paramIndex); }
};

let Foo = class Foo {
    constructor(init) { }
    handler(input) {
    }
};
__decorate([
    Prop(),
], Foo.prototype, "prop", void 0);
__decorate([
    Method(),
    __param(0, Param()),
], Foo.prototype, "handler", null);
Foo = __decorate([
    Cls(),
    __param(0, Param()),
], Foo);
```

> 完整的代码见：[Playground](https://www.typescriptlang.org/play?#code/GYVwdgxgLglg9mABAYQDYGcAUBKAXC1AQ3XQBEBTCOAJ0KhsQG8AoRRa8qEapTKQ6gHNO2RAF4AfE0QBfZnOahIsBIgCynABZwAJjnwao2nRSq161Jq3aduvfkJHipjWfOaLw0eEgAK1OAAHfUR-IPJqKABPUxo6BhY2Di4eRD4BYShRSSs2OQUlb1VfAUIAWxCS2jLOCNjzBOtkuzSHTOyXNwUAATQsbGYIIhJEADE4OFzEKjB0KGoQaBpMbqrynEQYMBgoAH58OeotwVFXBTZVgOCBtkCrg-nj8UQAclQtgCMQHRAXjwuwtdrHM6DAIIgQbAIICHkcwIJni9IWDEO8wF8fn9rN1DMYcNZNIQwDpUBEVmsKqItoEQFBYcdTv83Njcbp8WxkeDOQAJIkksmrUqUzZgGl0iGPeGM6z5DyDBBzRDACbPMDkADuYwmOAA3EA)

这里的 `__decorate` 方法，其实就是通过实际入参来判断当前到底执行的是哪种装饰器，然后执行对应的装饰逻辑。而观察这个方法调用时的入参，我们会再次观察到这些装饰器的不同入参：**方法与属性装饰器是类的原型对象**，而**类装饰器才能获得这个类本身作为入参**。而属性装饰器应用时，这个属性还未被初始化（属性需要实例化才会有值），这也是为什么它无法像方法装饰器那样获取到值。

可以看到，上面的装饰器顺序依次是**实例上的属性、方法、方法参数**，然后是**静态的属性、方法、方法参数**，最后是**类以及类构造函数参数**。

而从这一编译结果中，我们还能观察到不同类型装饰器的**执行顺序**。首先是实例上的属性、方法、方法参数，然后是静态的属性、方法、方法参数，最后是类以及类构造函数参数。而装饰器的**应用顺序**则略有不同，**方法参数装饰器会先于方法装饰器应用**（`__param(0, Param())`）。

> 关于执行顺序与应用顺序，执行是**装饰器求值得到最终装饰器表达式**的过程，而应用则是**最终装饰器逻辑代码执行**的过程：
>
> ```typescript
> function deco() {
>   // 执行
>   return () => {
>     // 应用
>   }
> }
> ```

实际上，对于实例与静态的属性、方法装饰器而言，它们的执行与应用顺序其实**取决于它们定义的位置**，你可以在上面的例子里把方法定义在属性之前，就会发现执行顺序变成了**方法**-**方法参数**-**属性**，即先定义先执行。

在 TypeScript 官方文档中对应用顺序给出了详细的定义：

1. _参数装饰器_，然后依次是*方法装饰器*，_访问符装饰器_，或*属性装饰器*应用到每个实例成员。
2. _参数装饰器_，然后依次是*方法装饰器*，_访问符装饰器_，或*属性装饰器*应用到每个静态成员。
3. *参数装饰器*应用到构造函数。
4. *类装饰器*应用到类。

最后，我们再看一个例子，来更深刻地了解执行顺序与应用顺序：

```typescript
function Deco(identifier: string): any {
  console.log(`${identifier} 执行`);
  return function () {
    console.log(`${identifier} 应用`);
  };
}

@Deco('类装饰器')
class Foo {
  constructor(@Deco('构造函数参数装饰器') name: string) {}

  @Deco('实例属性装饰器')
  prop?: number;

  @Deco('实例方法装饰器')
  handler(@Deco('实例方法参数装饰器') args: any) {}
}
```

以上的代码输出是这样的：

```text
实例属性装饰器 执行
实例属性装饰器 应用
实例方法装饰器 执行
实例方法参数装饰器 执行
实例方法参数装饰器 应用
实例方法装饰器 应用
类装饰器 执行
构造函数参数装饰器 执行
构造函数参数装饰器 应用
类装饰器 应用
```

执行顺序就不再赘述，这里我们主要关注应用顺序。顺序大致是**实例属性-实例方法参数-构造函数参数-类**，好像不对，不是说参数装饰器先应用吗？这是因为在这个例子中，我们是先定义属性和属性装饰器的，因此属性装饰器会先应用。如果方法在前，可不就是方法参数装饰器先应用？

你会发现，类装饰器是最后应用的。也就是说，如果我们在方法装饰器中标记某些信息，最终的类装饰器是可以消费到，并且基于此信息对类或类的实例进行某些操作的。如标记为 `@Deprecated` 的方法，我们在最终的类装饰器中可以将这些方法实现替换为一个报错！而标记这些信息的方法则有很多，最简单的如，在全局声明一个 Map，类作为 Key，这些信息作为 Value 也是可以的。当然，后面我们会说到如何使用更好的方式实现。

#### 多个同类装饰器的执行顺序

另外，我们也可以使用多个同种装饰器，比如一个类上可以有好多个类装饰器：

```typescript
@Deprecated()
@User()
@Internal
@Provide()
class Foo {}
```

这种情况下，这些装饰器的执行顺序又是怎样的？其顺序分为两步。首先，**由上至下**依次对装饰器的表达式求值，得到装饰器的实现，`@Internal` 中实现即为 Internal 方法，而 `@Provide()` 中实现则需要进行一次求值。

然后，这些装饰器的具体实现才会**从下往上**调用，如这里是 Provide、Internal、User、Deprecated 的顺序。从这个角度来看，甚至有点像洋葱模型：

```typescript
function Foo(): MethodDecorator {
  console.log('foo in');
  return (target, propertyKey, descriptor) => {
    console.log('foo out');
  };
}

function Bar(): MethodDecorator {
  console.log('bar in');
  return (target, propertyKey, descriptor) => {
    console.log('bar out');
  };
}

const Baz: MethodDecorator = () => {
  console.log('baz apply');
};

class User {
  @Foo()
  @Bar()
  @Baz
  method() {}
}

// foo in
// bar in
// baz apply
// bar out
// foo out
```

类似的，如果一个方法中的多个参数均存在装饰器，那么同样是 `Parma1 in` - `Param2 in ` - `Param2 out` - `Param1 out` 的顺序，也就是**后面参数的装饰器逻辑**反而先执行。

**但我们通常不会在同种装饰器中进行存在依赖关系的操作。** 对于属性、参数装饰器来说，我们通常只进行信息注册，委托别人处理。对于方法装饰器来说，我们最多只进行方法执行前后的逻辑注入。而这些过程都应当是彼此独立的。

那么，这里的委托又如何实现呢？这时候我们就要介绍一位新朋友了：**反射（Reflect）**。你可能很早就认识，但没怎么接触过。

## 反射 Reflect

Reflect 在 ES6 中被首次引入，它主要是为了配合 Proxy 保留一份方法原始的实现逻辑，如以下来自阮一峰老师的 ES6 标准入门中 [Reflect](https://es6.ruanyifeng.com/#docs/reflect) 一节的代码：

```js
Proxy(target, {
  set(target, name, value, receiver) {
    const success = Reflect.set(target, name, value, receiver)
    if (success)
      console.log(`property ${name} on ${target} set to ${value}`)

    return success
  }
})
```

Proxy 将修改这个对象的 set 方法，但我们可以通过 `Reflect.set` 方法获得原本的默认实现（不会被修改），先执行完默认实现逻辑再添加自己的额外逻辑。

Proxy 上的这些方法会一一对应到 Reflect 中（或者说 Reflect 中只有 Proxy 上方法的对应实现），如 defineProperty、deleteProperty、apply、get、set、has 等等。这些方法其实也可以在别的对象上找到，如 `Object.defineProperty`、`Function.prototype.apply` 等等，因此 Reflect 其实也起到了方法收拢的作用。

如果你有 Java、Go 等语言的基础，一定会反驳说反射才不是用来干这个的呢。别急，我们才刚要开始介绍。

上面的 Proxy 对象的 set 方法是运行时才实际执行的，也就是说我们通过反射，在**运行时去修改了程序的行为**。这就是反射的核心要素：**在程序运行时去检查以及修改程序行为**，比如在代码运行时通过 `Reflect.construct` 实例化一个类，通过 `Reflect.setPrototypeOf` 修改对象原型指向，这些其实都属于反射 API 。

> 此前 JavaScript 中的反射 API 散落在各个顶级对象的命名空间下，因此我们需要 Reflect 来进行一次统一。

比如通过反射来实例化一个类：

```typescript
// 普通情况
const foo = new Foo()
foo.hello()

// 基于反射
const foo = Reflect.construct(Foo, [])
const hello = Reflect.get(foo, 'hello')
Reflect.apply(hello, foo, [])
```

我们的主要内容和反射并没有太大的关系，下面要介绍的反射元数据才是本节的重量级角色。但你仍然需要铭记反射的核心理念：**在程序运行时去检查以及修改程序行为**。

## 反射元数据 Reflect Metadata

不同于反射，**反射元数据（Reflect Metadata）** 这一提案虽然同样很早就被提出，但至今都未真正的成为 ECMAScript 的一部分，原因在于元数据和装饰器提案的联系非常紧密，随着装饰器提案迟迟不能推进，元数据当然也无法独自向前。因此，想要使用反射元数据，你还需要安装 [reflect-metadata](https://github.com/rbuckton/reflect-metadata) ，并在入口文件中的顶部 `import "reflect-metadata"` 。

反射元数据提案（即 `"reflect-metadata"` 包）为顶级对象 Reflect 新增了一批专用于元数据读写的 API，如 `Reflect.defineMetadata`、`Reflect.getMetadata` 等。那么元数据又是什么？你可以将元数据理解为**用于描述数据的数据**，如某个方法的参数信息、返回值信息就可称为该方法的元数据。

那么元数据又存储在哪里？提案中专门说明了这一点，为类或类属性添加了元数据后，构造函数（或是构造函数的原型，根据静态成员还是实例成员决定）会具有 `[[Metadata]]` 属性，该属性内部包含一个 Map 结构，键为属性键，值为元数据键值对。也就是说，**静态成员的元数据信息存储于构造函数**，而**实例成员的元数据信息存储于构造函数的原型上**。

我们来简单使用下元数据的注册与提取：

```typescript
import 'reflect-metadata';

class Foo {
  handler() {}
}

Reflect.defineMetadata('class:key', 'class metadata', Foo);
Reflect.defineMetadata('method:key', 'handler metadata', Foo, 'handler');
Reflect.defineMetadata(
  'proto:method:key',
  'proto handler metadata',
  Foo.prototype,
  'handler'
);
```

defineMetadata 的入参包括元数据 Key、元数据 Value、目标类 Target 以及一个可选的属性，在这里我们的三个调用分别是在 Foo、Foo.handler 以及 Foo.prototype 上注册元数据。而提取则可以通过 getMetadata 方法：

```typescript
// [ 'class:key' ]
console.log(Reflect.getMetadataKeys(Foo));
// ['method:key']
console.log(Reflect.getMetadataKeys(Foo, 'handler'));
// ['proto:method:key'];
console.log(Reflect.getMetadataKeys(Foo.prototype, 'handler'));

// class metadata
console.log(Reflect.getMetadata('class:key', Foo));
// handler metadata
console.log(Reflect.getMetadata('method:key', Foo, 'handler'));
// proto handler metadata
console.log(Reflect.getMetadata('proto:method:key', Foo.prototype, 'handler'));
```

实际上，反射元数据正是我们实现属性装饰器中提到的“委托”能力的基础。我们在属性装饰器中去注册一个元数据，然后在真正实例化这个类时，就可以拿到类原型上的元数据，以此对实例化完毕的类再进行额外操作。比如说，我先通过元数据说明，这个属性需要获得变量 a 的值，在实例化时，我们发现有这个元数据，就会对应进行赋值操作。

正是考虑到这一点，反射元数据中直接就内置了基于装饰器的调用方式：

```typescript
@Reflect.metadata('class:key', 'METADATA_IN_CLASS')
class Foo {
  @Reflect.metadata('prop:key', 'METADATA_IN_PROPERTY')
  public prop: string = 'linbudu';

  @Reflect.metadata('method:key', 'METADATA_IN_METHOD')
  public handler(): void {}
}
```

`@Reflect.metadata` 装饰器会基于应用的位置进行实际的逻辑调用，如在类上装饰时以类作为 target 进行注册，而在静态成员与实例成员中分别使用构造函数、构造函数原型。

```typescript
const foo = new Foo();

// METADATA_IN_CLASS
console.log(Reflect.getMetadata('class:key', Foo));
// undefined
console.log(Reflect.getMetadata('class:key', Foo.prototype));

// METADATA_IN_METHOD
console.log(Reflect.getMetadata('method:key', Foo.prototype, 'handler'));
// METADATA_IN_METHOD
console.log(Reflect.getMetadata('method:key', foo, 'handler'));

// METADATA_IN_PROPERTY
console.log(Reflect.getMetadata('prop:key', Foo.prototype, 'prop'));
// METADATA_IN_PROPERTY
console.log(Reflect.getMetadata('prop:key', foo, 'prop'));
```

看起来我们现在拥有了实现委托的基本能力，但实际上这还不够。所有的元数据都需要我们提前定义好，如果我们希望直接用一些已有的信息作为元数据呢？比如下面这个例子：

```typescript
class UserService {
  @InjectModel()
  userModel: UserModel;
}
```

我希望将 userModel 属性的类型 UserModel 作为一个元数据信息注入，同时我不会为 `@InjectModel()` 装饰器提供任何信息，那我们就束手无策了吗？

还记得我们在介绍反射概念时说的，**反射允许程序去检视自身**，而属性类型作为程序的一部分，也应当是能被反射收集的。为了实现这一目的，反射元数据提案中还内置了基于类型的元数据，你可以通过 `design:type`、`design:paramtypes` 以及 `design:returntype` 这三个内置的元数据 Key，获取到类与类成员的类型、参数类型、返回值类型：

```typescript
import 'reflect-metadata';

function DefineType(type: Object) {
  return Reflect.metadata('design:type', type);
}
function DefineParamTypes(...types: Object[]) {
  return Reflect.metadata('design:paramtypes', types);
}
function DefineReturnType(type: Object) {
  return Reflect.metadata('design:returntype', type);
}

@DefineParamTypes(String, Number)
class Foo {
  @DefineType(String)
  get name() {
    return 'linbudu';
  }

  @DefineType(Function)
  @DefineParamTypes(Number, Number)
  @DefineReturnType(Number)
  add(source: number, input: number): number {
    return source + input;
  }
}

const foo = new Foo();
// [ [Function: Number], [Function: Number] ]
const paramTypes = Reflect.getMetadata('design:paramtypes', foo, 'add');
// [Function: Number]
const returnTypes = Reflect.getMetadata('design:returntype', foo, 'add');
// [Function: String]
const type = Reflect.getMetadata('design:type', foo, 'name');
```

需要注意的是，这一提案实际上并不依赖 TypeScript ，这些类型信息来自于运行时，而非我们的类型标注。同时这些内置元数据取出的值是装箱类型对象，如 String、Number 等。

TypeScript 为其进行了额外的支持，然后我们才可以获取到类型标注所对应的元数据，如：

```typescript
class Bar {
  @DefineType(Foo)
  prop!: Foo;
}

const bar = new Bar();
// [class Foo]
const type2 = Reflect.getMetadata('design:type', bar, 'prop');
```

这也是为什么我们需要启用 `emitDecoratorMetadata` 配置的原因之一。上面的装饰器执行机制代码中我们看到了编译后的装饰器代码，而启用 `emitDecoratorMetadata` 后，产物中会多出这些代码：

```js
const __metadata = (this && this.__metadata) || function (k, v) {
  if (typeof Reflect === 'object' && typeof Reflect.metadata === 'function')
    return Reflect.metadata(k, v)
}

__decorate([
  Prop(),
  __metadata('design:type', String) // 新增
], Foo.prototype, 'prop', void 0)

__decorate([
  Method(),
  __param(0, Param()),
  __metadata('design:type', Function), // 新增
  __metadata('design:paramtypes', [String]), // 新增
  __metadata('design:returntype', void 0) // 新增
], Foo.prototype, 'handler', null)

Foo = __decorate([
  Cls(),
  __param(0, Param()),
  __metadata('design:paramtypes', [String]) // 新增
], Foo)
```

有了装饰器、反射元数据以及内置的基于类型的元数据信息，我们就可以实现“委托”的能力了。以看似平平无奇的属性装饰器为例，我们使用元数据来实现基于装饰器的属性校验。

在这个例子里，我们会实现两种校验逻辑，对必填属性（Required）与属性类型的校验（String / Number / Boolean），其基本使用方式如下：

```typescript
class User {
  @Required()
  name!: string;

  @ValueType(TypeValidation.Number)
  age!: number;
}

const user = new User();
// @ts-expect-error
user.age = '18';
```

我们会将 user 实例传递给校验方法，在这里应当给出两处错误：没有提供必填属性 name，以及 age 属性的类型不符。

如果理解了元数据的作用，那我们的思路就很明确了，装饰器将元数据附加到属性或类上，然后校验方法中遍历属性读取这些元数据，再对比类型是否匹配即可。

首先是 Required ，我们肯定下意识是这么写：

```typescript
function Required(): PropertyDecorator {
  return (target, prop) => {
    Reflect.defineMetadata("required", true, target, prop);
  };
}
```

也就是在这个属性上定义了一个名为 required 的元数据。但你是否想过，如果实例中根本就没有这个属性呢？就像上面的 user 一样，那这里的元数据不就丢失了？

要解决这一问题，其实只需要将元数据定义在类上即可。我们用一个专门描述必填属性的元数据，存储这个类内部所有的必填属性即可：

```typescript
const requiredMetadataKey = Symbol('requiredKeys');

function Required(): PropertyDecorator {
  return (target, prop) => {
    const existRequiredKeys: string[] =
      Reflect.getMetadata(requiredMetadataKey, target) ?? [];

    Reflect.defineMetadata(
      requiredMetadataKey,
      [...existRequiredKeys, prop],
      target
    );
  };
}
```

而对于属性的校验其实就简单了，由于对类型的校验逻辑可以归到一起，我们就使用**装饰器工厂 + 入参**的形式来注入对应的元数据信息，这次我们只需要在属性层面注入元数据即可：

```typescript
enum TypeValidation {
  String = 'string',
  Number = 'number',
  Boolean = 'boolean',
}

const validationMetadataKey = Symbol('expectedType');

function ValueType(type: TypeValidation): PropertyDecorator {
  return (target, prop) => {
    Reflect.defineMetadata(validationMetadataKey, type, target, prop);
  };
}
```

然后就是校验逻辑了，我们需要一个额外的 validator 方法：

```typescript
function validator(entity: any) {}

console.log(validator(user));
```

如果校验完全通过，那这一方法的返回值则是一个空数组，否则的话内部会存有报错信息。首先是对于必填属性的校验，我们需要取出注册在类上的，描述必填属性的元数据，再检查这些必填属性是否都存在了：

```typescript
function validator(entity: any) {
  const clsName = entity.constructor.name;
  const messages: string[] = [];
  // 先检查所有必填属性
  const requiredKeys: string[] = Reflect.getMetadata(
    requiredMetadataKey,
    entity
  );

  // 基于反射拿到所有存在的属性
  const existKeys = Reflect.ownKeys(entity);

  for (const key of requiredKeys) {
    if (!existKeys.includes(key)) {
      messages.push(`${clsName}.${key} should be required.`);
      // throw new Error(`${key} is required!`);
    }
  }

  return messages;
}
```

然后是对属性类型的校验，我们的 TypeValidation 枚举中，枚举值就是 `typeof` 的返回值，因此这里直接使用即可：

```typescript
function validator(entity: any) {
  // ...
  // 接着基于定义在属性上的元数据校验属性类型
  for (const key of existKeys) {
    const expectedType: string = Reflect.getMetadata(
      validationMetadataKey,
      entity,
      key
    );

    if (!expectedType) continue;

    // 枚举也是对象，因此 Object.values 同样可以生效（只不过也会包括键名）
    // @ts-expect-error
    if (Object.values(TypeValidation).includes(expectedType)) {
      const actualType = typeof entity[key];
      if (actualType !== expectedType) {
        messages.push(
          `expect ${entity.constructor.name}.${String(
            key
          )} to be ${expectedType}, but got ${actualType}.`
        );
        // throw new Error(`${String(key)} is not ${expectedType}!`);
      }
    }
  }
  return messages;
}
```

最终的输出会是这样的：

```
[
  'User.name should be required.',
  'expect User.age to be number, but got string.'
]
```

除了这两种校验，你也可以通过元数据的帮助来实现更复杂的校验逻辑。如 MinLength、MaxLength、Min、Max 甚至 Email、IP 这样，对属性值内容的校验。思路仍然还是那么简单明了：**注册元数据，消费元数据**。

## 总结与预告

这一节，我们了解了装饰器的基本概念，包括 TypeScript 中的五种装饰器，以及这些装饰器的入参、使用场景、执行顺序等等。另外我们还掌握了反射元数据的使用，目前看起来它好像并没有什么特别之处？那么在下一节，我们就会在反射元数据的基础上，去了解一个新的概念：控制反转。

## 扩展阅读

### 装饰器的坎坷进历程

正如我们在开头提到的，装饰器从被作为一个提案提出开始，很是经历了一番风雨，下面我们就来具体介绍一下它到底都经历了些什么。

首先需要明确的是，目前 JavaScript（ECMAScript）中的装饰器，和我们这节学习的 TypeScript 装饰器基本是两件完全不同的事物。[装饰器提案](https://github.com/tc39/proposal-decorators) 距离最开始提出已经过去了数年，在这期间提案内容，也就是语法、作用与运行时机制等，已经迭代了四个版本。

第四个版本在 2022 年 3 月份的 TC39 会议中终于如愿进入 Stage 3，也就意味着这一版本的实现基本上就是未来最终落地的版本。此前的版本都在 Stage 2 就胎死腹中，而 TypeScript 与 Babel 中的装饰器则是基于第一版的提案实现的，虽然语法都还是 `@` ，但这两个版本的装饰器实际上差异非常之大。

> 如果你有兴趣了解新版装饰器的具体语义，可以阅读我此前发表的 [2022 年 3 月 TC39 会议报告](https://mp.weixin.qq.com/s/6PTcjJQTED3WpJH8ToXInw) 来了解更多。另外，在 ECMAScript 装饰器进入 Stage 4，或已经有可用的编译支持（Babel / TypeScript ）后，我也会更新关于新版装饰器的使用说明。

通常来说， TypeScript 只会对已经到达 Stage 3 的提案进行提前的支持，如可选链、空值合并、逻辑赋值等。当 TypeScript 最初引入装饰器时大概是在 2015 年，此时装饰器提案位于 Stage 1 阶段。

促使 TS 提前引入的一个重要原因是，当时存在一门 TS 的超集语言（也就是 JS 的超集的超集？） [AtScript](https://en.wikipedia.org/wiki/AtScript)，它在 TS 的基础上去支持了装饰器语法，来供 Angular 框架使用。TS 团队与 Angular 团队在某种契机下达成一致，决定将装饰器以及相关的注解能力直接引入 TypeScript 中，而 Angular 团队不再维护 AtScript ，这实际上避免了未来可能出现的竞争与社区生态分裂问题。

虽然这两个版本的装饰器确实差异很大，但你其实无需担心出现未来需要面对断崖式的更新，目前新版装饰器的能力基本上能完全覆盖旧版所能提供的能力，因此升级成本对于用户或者框架开发者来说都不会太高。而如果还想继续使用旧版装饰器怎么办？我猜 TypeScript 会通过引入一个新的 Compiler Option 来控制实际表现与编译产物。

### Reflect.decorate

如果你去观察了装饰器的编译代码，会发现 `__decorate` 方法中有一段代码是检查 `Reflect.decorate` 方法是否存在。这一方法其实也来自于 Reflect Metadata，见 [L115](https://github.com/rbuckton/reflect-metadata/blob/master/Reflect.ts#L115)。这一方法的作用就是，通过反射的方式来进行装饰，如：

```typescript
class Foo {}

Reflect.decorate([/** ...一组装饰器 */], Foo)
```

这也就意味着，你甚至可以**在方法内部去装饰某一个类或其成**员，而不是仅仅只能依赖需要提前定义好的装饰器。
