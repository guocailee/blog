---
{"dg-publish":true,"permalink":"/Program/FrontEnd/TypeScript 之 More on Functions/","noteIcon":""}
---


## 前言

TypeScript 的官方文档早已更新，但我能找到的中文文档都还停留在比较老的版本。所以对其中新增以及修订较多的一些章节进行了翻译整理。

本篇整理自 TypeScript Handbook 中 「[More on Functions](https://link.zhihu.com/?target=https%3A//www.typescriptlang.org/docs/handbook/2/functions.html)」 章节。

本文并不严格按照原文翻译，对部分内容也做了解释补充。

## 正文

函数是任何应用的基础组成部分，无论它是局部函数 (local functions)，还是从其他模块导入的函数，亦或是类中的方法。当然，函数也是值 (values)，而且像其他值一样，TypeScript 有很多种方式用来描述，函数可以以怎样的方式被调用。让我们来学习一下如何书写描述函数的类型（types）。

## 函数类型表达式（Function Type Expressions）

最简单描述一个函数的方式是使用 函数类型表达式（function type expression）。它的写法有点类似于箭头函数：

```ts
function greeter(fn: (a: string) => void) {
  fn('Hello, World')
}

function printToConsole(s: string) {
  console.log(s)
}

greeter(printToConsole)
```

语法 `(a: string) => void` 表示一个函数有一个名为 `a` ，类型是字符串的参数，这个函数并没有返回任何值。

如果一个函数参数的类型并没有明确给出，它会被隐式设置为 `any`。

> 注意函数参数的名字是必须的，这种函数类型描述 `(string) => void`，表示的其实是一个函数有一个类型是 `any`，名为 `string` 的参数。

当然了，我们也可以使用类型别名（type alias）定义一个函数类型：

```ts
type GreetFunction = (a: string) => void
function greeter(fn: GreetFunction) {
  // ...
}
```

## 调用签名（Call Signatures）

在 JavaScript 中，函数除了可以被调用，自己也是可以有属性值的。然而上一节讲到的函数类型表达式并不能支持声明属性，如果我们想描述一个带有属性的函数，我们可以在一个对象类型中写一个调用签名（call signature）。

```ts
type DescribableFunction = {
  description: string
  (someArg: number): boolean
}
function doSomething(fn: DescribableFunction) {
  console.log(fn.description + ' returned ' + fn(6))
}
```

注意这个语法跟函数类型表达式稍有不同，在参数列表和返回的类型之间用的是 `:` 而不是 =>。

## 构造签名 （Construct Signatures）

JavaScript 函数也可以使用 `new` 操作符调用，当被调用的时候，TypeScript 会认为这是一个构造函数 (constructors)，因为他们会产生一个新对象。你可以写一个构造签名，方法是在调用签名前面加一个 `new` 关键词：

```ts
type SomeConstructor = {
  new (s: string): SomeObject
}
function fn(ctor: SomeConstructor) {
  return new ctor('hello')
}
```

一些对象，比如 `Date` 对象，可以直接调用，也可以使用 `new` 操作符调用，而你可以将调用签名和构造签名合并在一起：

```ts
interface CallOrConstruct {
  new (s: string): Date
  (n?: number): number
}
```

## 泛型函数 （Generic Functions）

我们经常需要写这种函数，即函数的输出类型依赖函数的输入类型，或者两个输入的类型以某种形式相互关联。让我们考虑这样一个函数，它返回数组的第一个元素：

```ts
function firstElement(arr: any[]) {
  return arr[0]
}
```

注意此时函数返回值的类型是 `any`，如果能返回第一个元素的具体类型就更好了。

在 TypeScript 中，泛型就是被用来描述两个值之间的对应关系。我们需要在函数签名里声明一个类型参数 (type parameter)：

```ts
function firstElement<Type>(arr: Type[]): Type | undefined {
  return arr[0]
}
```

通过给函数添加一个类型参数 `Type`，并且在两个地方使用它，我们就在函数的输入 (即数组) 和函数的输出 (即返回值) 之间创建了一个关联。现在当我们调用它，一个更具体的类型就会被判断出来：

```ts
// s is of type 'string'
const s = firstElement(['a', 'b', 'c'])
// n is of type 'number'
const n = firstElement([1, 2, 3])
// u is of type undefined
const u = firstElement([])
```

### 推断（Inference）

注意在上面的例子中，我们没有明确指定 `Type` 的类型，类型是被 TypeScript 自动推断出来的。

我们也可以使用多个类型参数，举个例子：

```ts
function map<Input, Output>(arr: Input[], func: (arg: Input) => Output): Output[] {
  return arr.map(func)
}

// Parameter 'n' is of type 'string'
// 'parsed' is of type 'number[]'
const parsed = map(['1', '2', '3'], (n) => parseInt(n))
```

注意在这个例子中，TypeScript 既可以推断出 Input 的类型 （从传入的 `string` 数组），又可以根据函数表达式的返回值推断出 `Output` 的类型。

### 约束（Constraints）

有的时候，我们想关联两个值，但只能操作值的一些固定字段，这种情况，我们可以使用 \*\*约束（constraint）\*\*对类型参数进行限制。

让我们写一个函数，函数返回两个值中更长的那个。为此，我们需要保证传入的值有一个 `number` 类型的 `length` 属性。我们使用 `extends` 语法来约束函数参数：

```ts
function longest<Type extends { length: number }>(a: Type, b: Type) {
  if (a.length >= b.length) {
    return a
  } else {
    return b
  }
}

// longerArray is of type 'number[]'
const longerArray = longest([1, 2], [1, 2, 3])
// longerString is of type 'alice' | 'bob'
const longerString = longest('alice', 'bob')
// Error! Numbers don't have a 'length' property
const notOK = longest(10, 100)
// Argument of type 'number' is not assignable to parameter of type '{ length: number; }'.
```

TypeScript 会推断 `longest` 的返回类型，所以返回值的类型推断在泛型函数里也是适用的。

正是因为我们对 `Type` 做了 `{length: number}` 限制，我们才可以被允许获取 `a` `b`参数的 `.length` 属性。没有这个类型约束，我们甚至不能获取这些属性，因为这些值也许是其他类型，并没有 length 属性。

基于传入的参数，`longerArray`和 `longerString` 中的类型都被推断出来了。记住，所谓泛型就是用一个相同类型来关联两个或者更多的值。

### 泛型约束实战（Working with Constrained Values）

这是一个使用泛型约束常出现的错误：

```ts
function minimumLength<Type extends { length: number }>(obj: Type, minimum: number): Type {
  if (obj.length >= minimum) {
    return obj
  } else {
    return { length: minimum }
    // Type '{ length: number; }' is not assignable to type 'Type'.
    // '{ length: number; }' is assignable to the constraint of type 'Type', but 'Type' could be instantiated with a different subtype of constraint '{ length: number; }'.
  }
}
```

这个函数看起来像是没有问题，`Type` 被 `{length: number}` 约束，函数返回 `Type` 或者一个符合约束的值。

而这其中的问题就在于函数理应返回与传入参数相同类型的对象，而不仅仅是符合约束的对象。我们可以写出这样一段反例：

```ts
// 'arr' gets value { length: 6 }
const arr = minimumLength([1, 2, 3], 6)
// and crashes here because arrays have
// a 'slice' method, but not the returned object!
console.log(arr.slice(0))
```

### 声明类型参数 （Specifying Type Arguments）

TypeScript 通常能自动推断泛型调用中传入的类型参数，但也并不能总是推断出。举个例子，有这样一个合并两个数组的函数：

```ts
function combine<Type>(arr1: Type[], arr2: Type[]): Type[] {
  return arr1.concat(arr2)
}
```

如果你像下面这样调用函数就会出现错误：

```ts
const arr = combine([1, 2, 3], ['hello'])
// Type 'string' is not assignable to type 'number'.
```

而如果你执意要这样做，你可以手动指定 `Type`：

```ts
const arr = combine<string | number>([1, 2, 3], ['hello'])
```

### 写一个好的泛型函数的一些建议

尽管写泛型函数很有意思，但也容易翻车。如果你使用了太多的类型参数，或者使用了一些并不需要的约束，都可能会导致不正确的类型推断。

### 类型参数下移（Push Type Parameters Down）

下面这两个函数的写法很相似：

```ts
function firstElement1<Type>(arr: Type[]) {
  return arr[0]
}

function firstElement2<Type extends any[]>(arr: Type) {
  return arr[0]
}

// a: number (good)
const a = firstElement1([1, 2, 3])
// b: any (bad)
const b = firstElement2([1, 2, 3])
```

第一眼看上去，两个函数可太相似了，但是第一个函数的写法可比第二个函数好太多了。第一个函数可以推断出返回的类型是 `number`，但第二个函数推断的返回类型却是 `any`，因为 TypeScript 不得不用约束的类型来推断 `arr[0]` 表达式，而不是等到函数调用的时候再去推断这个元素。

关于本节原文中的 `push down` 含义，在《重构》里，就有一个函数下移（Push Down Method）的优化方法，指如果超类中的某个函数只与一个或者少数几个子类有关，那么最好将其从超类中挪走，放到真正关心它的子类中去。即只在超类保留共用的行为。这种将超类中的函数本体复制到具体需要的子类的方法就可以称之为 "push down"，与本节中的去除 `extend any[]`，将其具体的推断交给 `Type` 自身就类似于 `push down`。

> Rule: 如果可能的话，直接使用类型参数而不是约束它

### 使用更少的类型参数 (Use Fewer Type Parameters)

这是另一对看起来很相似的函数：

```ts
function filter1<Type>(arr: Type[], func: (arg: Type) => boolean): Type[] {
  return arr.filter(func)
}

function filter2<Type, Func extends (arg: Type) => boolean>(arr: Type[], func: Func): Type[] {
  return arr.filter(func)
}
```

我们创建了一个并没有关联两个值的类型参数 `Func`，这是一个危险信号，因为它意味着调用者不得不毫无理由的手动指定一个额外的类型参数。`Func` 什么也没做，却导致函数更难阅读和推断。

> Rule: 尽可能用更少的类型参数

### 类型参数应该出现两次 （Type Parameters Should Appear Twice）

有的时候我们会忘记一个函数其实并不需要泛型

```ts
function greet<Str extends string>(s: Str) {
  console.log('Hello, ' + s)
}

greet('world')
```

其实我们可以如此简单的写这个函数：

```ts
function greet(s: string) {
  console.log('Hello, ' + s)
}
```

记住：类型参数是用来关联多个值之间的类型。如果一个类型参数只在函数签名里出现了一次，那它就没有跟任何东西产生关联。

> Rule: 如果一个类型参数仅仅出现在一个地方，强烈建议你重新考虑是否真的需要它

## 可选参数（Optional Parameters）

JavaScript 中的函数经常会被传入非固定数量的参数，举个例子：`number` 的 `toFixed` 方法就支持传入一个可选的参数：

```ts
function f(n: number) {
  console.log(n.toFixed()) // 0 arguments
  console.log(n.toFixed(3)) // 1 argument
}
```

我们可以使用 `?` 表示这个参数是可选的：

```ts
function f(x?: number) {
  // ...
}
f() // OK
f(10) // OK
```

尽管这个参数被声明为 `number`类型，`x` 实际上的类型为 `number | undefiend`，这是因为在 JavaScript 中未指定的函数参数就会被赋值 `undefined`。

你当然也可以提供有一个参数默认值：

```ts
function f(x = 10) {
  // ...
}
```

现在在 `f` 函数体内，`x` 的类型为 `number`，因为任何 `undefined` 参数都会被替换为 `10`。注意当一个参数是可选的，调用的时候还是可以传入 `undefined`：

```ts
declare function f(x?: number): void
// cut
// All OK
f()
f(10)
f(undefined)
```

### 回调中的可选参数（Optional Parameters in Callbacks）

在你学习过可选参数和函数类型表达式后，你很容易在包含了回调函数的函数中，犯下面这种错误：

```text
function myForEach(arr: any[], callback: (arg: any, index?: number) => void) {
  for (let i = 0; i < arr.length; i++) {
    callback(arr[i], i);
  }
}

```

将 `index?`作为一个可选参数，本意是希望下面这些调用是合法的：

```text
myForEach([1, 2, 3], (a) => console.log(a));
myForEach([1, 2, 3], (a, i) => console.log(a, i));

```

但 TypeScript 并不会这样认为，TypeScript 认为想表达的是回调函数可能只会被传入一个参数，换句话说，`myForEach` 函数也可能是这样的：

```ts
function myForEach(arr: any[], callback: (arg: any, index?: number) => void) {
  for (let i = 0; i < arr.length; i++) {
    // I don't feel like providing the index today
    callback(arr[i])
  }
}
```

TypeScript 会按照这个意思理解并报错，尽管实际上这个错误并无可能：

```ts
// 冴羽注：最新的 TypeScript 版本中并不会报错
myForEach([1, 2, 3], (a, i) => {
  console.log(i.toFixed())
  // Object is possibly 'undefined'.
})
```

那如何修改呢？不设置为可选参数其实就可以：

```ts
function myForEach(arr: any[], callback: (arg: any, index: number) => void) {
  for (let i = 0; i < arr.length; i++) {
    callback(arr[i], i)
  }
}

myForEach([1, 2, 3], (a, i) => {
  console.log(a)
})
```

在 JavaScript 中，如果你调用一个函数的时候，传入了比需要更多的参数，额外的参数就会被忽略。TypeScript 也是同样的做法。

> 当你写一个回调函数的类型时, 不要写一个可选参数, 除非你真的打算调用函数的时候不传入实参

## 函数重载（Function Overloads）

一些 JavaScript 函数在调用的时候可以传入不同数量和类型的参数。举个例子。你可以写一个函数，返回一个日期类型 `Date`，这个函数接收一个时间戳（一个参数）或者一个 月 / 日 / 年 的格式 (三个参数)。

在 TypeScript 中，我们可以通过写重载签名 (overlaod signatures) 说明一个函数的不同调用方法。 我们需要写一些函数签名 (通常两个或者更多)，然后再写函数体的内容：

```ts
function makeDate(timestamp: number): Date
function makeDate(m: number, d: number, y: number): Date
function makeDate(mOrTimestamp: number, d?: number, y?: number): Date {
  if (d !== undefined && y !== undefined) {
    return new Date(y, mOrTimestamp, d)
  } else {
    return new Date(mOrTimestamp)
  }
}
const d1 = makeDate(12345678)
const d2 = makeDate(5, 5, 5)
const d3 = makeDate(1, 3)

// No overload expects 2 arguments, but overloads do exist that expect either 1 or 3 arguments.
```

在这个例子中，我们写了两个函数重载，一个接受一个参数，另外一个接受三个参数。前面两个函数签名被称为重载签名 (overload signatures)。

然后，我们写了一个兼容签名的函数实现，我们称之为实现签名 (implementation signature) ，但这个签名不能被直接调用。尽管我们在函数声明中，在一个必须参数后，声明了两个可选参数，它依然不能被传入两个参数进行调用。

### 重载签名和实现签名（Overload Signatures and the Implementation Signature）

这是一个常见的困惑。大家常会这样写代码，但是又不理解为什么会报错：

```ts
function fn(x: string): void;
function fn() {
  // ...
}
// Expected to be able to call with zero arguments
fn();
Expected 1 arguments, but got 0.

```

再次强调一下，写进函数体的签名是对外部来说是 “不可见” 的，这也就意味着外界 “看不到” 它的签名，自然不能按照实现签名的方式来调用。

> 实现签名对外界来说是不可见的。当写一个重载函数的时候，你应该总是需要来两个或者更多的签名在实现签名之上。

而且实现签名必须和重载签名必须兼容（compatible），举个例子，这些函数之所以报错就是因为它们的实现签名并没有正确的和重载签名匹配。

```ts
function fn(x: boolean): void;
// Argument type isn't right
function fn(x: string): void;
// This overload signature is not compatible with its implementation signature.
function fn(x: boolean) {}

function fn(x: string): string;
// Return type isn't right
function fn(x: number): boolean;
This overload signature is not compatible with its implementation signature.
function fn(x: string | number) {
  return "oops";
}

```

### 写一个好的函数重载的一些建议

就像泛型一样，也有一些建议提供给你。遵循这些原则，可以让你的函数更方便调用、理解。

让我们设想这样一个函数，该函数返回数组或者字符串的长度：

```ts
function len(s: string): number
function len(arr: any[]): number
function len(x: any) {
  return x.length
}
```

这个函数代码功能实现了，也没有什么报错，但我们不能传入一个可能是字符串或者是数组的值，因为 TypeScript 只能一次用一个函数重载处理一次函数调用。

```ts
len(""); // OK
len([0]); // OK
len(Math.random() > 0.5 ? "hello" : [0]);
No overload matches this call.
  Overload 1 of 2, '(s: string): number', gave the following error.
    Argument of type 'number[] | "hello"' is not assignable to parameter of type 'string'.
      Type 'number[]' is not assignable to type 'string'.
  Overload 2 of 2, '(arr: any[]): number', gave the following error.
    Argument of type 'number[] | "hello"' is not assignable to parameter of type 'any[]'.
      Type 'string' is not assignable to type 'any[]'.

```

因为两个函数重载都有相同的参数数量和相同的返回类型，我们可以写一个无重载版本的函数替代：

```ts
function len(x: any[] | string) {
  return x.length
}
```

这样函数就可以传入两个类型中的任意一个了。

> 尽可能的使用联合类型替代重载

### 在函数中声明 this (Declaring `this` in a Function)

TypeScript 会通过代码流分析函数中的 `this` 会是什么类型，举个例子：

```ts
const user = {
  id: 123,

  admin: false,
  becomeAdmin: function () {
    this.admin = true
  },
}
```

TypeScript 能够理解函数 `user.becomeAdmin` 中的 `this` 指向的是外层的对象 `user`，这已经可以应付很多情况了，但还是有一些情况需要你明确的告诉 TypeScript `this` 到底代表的是什么。

在 JavaScript 中，`this` 是保留字，所以不能当做参数使用。但 TypeScript 可以允许你在函数体内声明 `this` 的类型。

```ts
interface DB {
  filterUsers(filter: (this: User) => boolean): User[]
}

const db = getDB()
const admins = db.filterUsers(function (this: User) {
  return this.admin
})
```

这个写法有点类似于回调风格的 API。注意你需要使用 `function` 的形式而不能使用箭头函数：

```ts
interface DB {
  filterUsers(filter: (this: User) => boolean): User[]
}

const db = getDB()
const admins = db.filterUsers(() => this.admin)
// The containing arrow function captures the global value of 'this'.
// Element implicitly has an 'any' type because type 'typeof globalThis' has no index signature.
```

## 其他需要知道的类型（Other Types to Know About）

这里介绍一些也会经常出现的类型。像其他的类型一样，你也可以在任何地方使用它们，但它们经常与函数搭配使用。

### `void`

`void` 表示一个函数并不会返回任何值，当函数并没有任何返回值，或者返回不了明确的值的时候，就应该用这种类型。

```ts
// The inferred return type is void
function noop() {
  return
}
```

在 JavaScript 中，一个函数并不会返回任何值，会隐式返回 `undefined`，但是 `void` 和 `undefined` 在 TypeScript 中并不一样。在本文的最后会有更详细的介绍。

> void 跟 undefined 不一样

### `object`

这个特殊的类型 `object` 可以表示任何不是原始类型（primitive）的值 (`string`、`number`、`bigint`、`boolean`、`symbol`、`null`、`undefined`)。`object` 不同于空对象类型 `{ }`，也不同于全局类型 `Object`。很有可能你也用不到 `Object`。

> object 不同于 `Object` ，总是用 `object`!

注意在 JavaScript 中，函数就是对象，他们可以有属性，在他们的原型链上有 `Object.prototype`，并且 `instanceof Object`。你可以对函数使用 `Object.keys` 等等。由于这些原因，在 TypeScript 中，函数也被认为是 `object`。

### `unknown`

`unknown` 类型可以表示任何值。有点类似于 `any`，但是更安全，因为对 `unknown` 类型的值做任何事情都是不合法的：

```ts
function f1(a: any) {
  a.b() // OK
}
function f2(a: unknown) {
  a.b()
  // Object is of type 'unknown'.
}
```

有的时候用来描述函数类型，还是蛮有用的。你可以描述一个函数可以接受传入任何值，但是在函数体内又不用到 `any` 类型的值。

你可以描述一个函数返回一个不知道什么类型的值，比如：

```ts
function safeParse(s: string): unknown {
  return JSON.parse(s)
}

// Need to be careful with 'obj'!
const obj = safeParse(someRandomString)
```

### `never`

一些函数从来不返回值：

```ts
function fail(msg: string): never {
  throw new Error(msg)
}
```

`never` 类型表示一个值不会再被观察到 (observed)。

作为一个返回类型时，它表示这个函数会丢一个异常，或者会结束程序的执行。

当 TypeScript 确定在联合类型中已经没有可能是其中的类型的时候，`never` 类型也会出现：

```ts
function fn(x: string | number) {
  if (typeof x === 'string') {
    // do something
  } else if (typeof x === 'number') {
    // do something else
  } else {
    x // has type 'never'!
  }
}
```

### `Function`

在 JavaScript，全局类型 `Function` 描述了 `bind`、`call`、`apply` 等属性，以及其他所有的函数值。

它也有一个特殊的性质，就是 `Function` 类型的值总是可以被调用，结果会返回 `any` 类型：

```ts
function doSomething(f: Function) {
  f(1, 2, 3)
}
```

这是一个无类型函数调用 (untyped function call)，这种调用最好被避免，因为它返回的是一个不安全的 `any`类型。

如果你准备接受一个黑盒的函数，但是又不打算调用它，`() => void` 会更安全一些。

## 剩余参数（Rest Parameters and Arguments）

### `parameters` 与 `arguments`

`arguments` 和 `parameters` 都可以表示函数的参数，由于本节内容做了具体的区分，所以我们定义 `parameters` 表示我们定义函数时设置的名字即形参，`arguments` 表示我们实际传入函数的参数即实参。

### 剩余参数（Rest Parameters）

除了用可选参数、重载能让函数接收不同数量的函数参数，我们也可以通过使用剩余参数语法（rest parameters），定义一个可以传入数量不受限制的函数参数的函数：

剩余参数必须放在所有参数的最后面，并使用 `...` 语法：

```ts
function multiply(n: number, ...m: number[]) {
  return m.map((x) => n * x)
}
// 'a' gets value [10, 20, 30, 40]
const a = multiply(10, 1, 2, 3, 4)
```

在 TypeScript 中，剩余参数的类型会被隐式设置为 `any[]` 而不是 `any`，如果你要设置具体的类型，必须是 `Array<T>` 或者 `T[]`的形式，再或者就是元祖类型（tuple type）。

### 剩余参数（Rest Arguments）

我们可以借助一个使用 `...` 语法的数组，为函数提供不定数量的实参。举个例子，数组的 `push` 方法就可以接受任何数量的实参：

```ts
const arr1 = [1, 2, 3]
const arr2 = [4, 5, 6]
arr1.push(...arr2)
```

注意一般情况下，TypeScript 并不会假定数组是不变的 (immutable)，这会导致一些意外的行为：

```ts
// 类型被推断为 number[] -- "an array with zero or more numbers",
// not specifically two numbers
const args = [8, 5]
const angle = Math.atan2(...args)
// A spread argument must either have a tuple type or be passed to a rest parameter.
```

修复这个问题需要你写一点代码，通常来说, 使用 `as const` 是最直接有效的解决方法：

```ts
// Inferred as 2-length tuple
const args = [8, 5] as const
// OK
const angle = Math.atan2(...args)
```

通过 `as const` 语法将其变为只读元组便可以解决这个问题。

注意当你要运行在比较老的环境时，使用剩余参数语法也许需要你开启 `[downlevelIteration](https://www.typescriptlang.org/tsconfig#downlevelIteration)` ，将代码转换为旧版本的 JavaScript。

## 参数解构（Parameter Destructuring）

你可以使用参数解构方便的将作为参数提供的对象解构为函数体内一个或者多个局部变量，在 JavaScript 中，是这样的：

```ts
function sum({ a, b, c }) {
  console.log(a + b + c)
}
sum({ a: 10, b: 3, c: 9 })
```

在解构语法后，要写上对象的类型注解：

```ts
function sum({ a, b, c }: { a: number; b: number; c: number }) {
  console.log(a + b + c)
}
```

这个看起来有些繁琐，你也可以这样写：

```ts
// 跟上面是有一样的
type ABC = { a: number; b: number; c: number }
function sum({ a, b, c }: ABC) {
  console.log(a + b + c)
}
```

## 函数的可赋值性 （Assignability of Functions）

### 返回 `void`

函数有一个 `void` 返回类型，会产生一些意料之外，情理之中的行为。

当基于上下文的类型推导（Contextual Typing）推导出返回类型为 `void` 的时候，并不会强制函数一定不能返回内容。换句话说，如果这样一个返回 `void` 类型的函数类型 `(type vf = () => void)`， 当被应用的时候，也是可以返回任何值的，但返回的值会被忽略掉。

因此，下面这些`() => void` 类型的实现都是有效的：

```ts
type voidFunc = () => void

const f1: voidFunc = () => {
  return true
}

const f2: voidFunc = () => true

const f3: voidFunc = function () {
  return true
}
```

而且即便这些函数的返回值赋值给其他变量，也会维持 `void` 类型：

```ts
const v1 = f1()

const v2 = f2()

const v3 = f3()
```

正是因为这个特性的存在，所以接下来的代码才会是有效的：

```ts
const src = [1, 2, 3]
const dst = [0]

src.forEach((el) => dst.push(el))
```

尽管 `Array.prototype.push` 返回一个数字，并且 `Array.prototype.forEach` 方法期待一个返回 `void` 类型的函数，但这段代码依然没有报错。就是因为基于上下文推导，推导出 forEach 函数返回类型为 void，正是因为不强制函数一定不能返回内容，所以上面这种 `return dst.push(el)` 的写法才不会报错。

另外还有一个特殊的例子需要注意，当一个函数字面量定义返回一个 `void` 类型，函数是一定不能返回任何东西的：

```ts
function f2(): void {
  // @ts-expect-error
  return true
}

const f3 = function (): void {
  // @ts-expect-error
  return true
}
```

## TypeScript 系列

冴羽的全系列文章地址：[https://github.com/mqyqingfeng/Blog](https://link.zhihu.com/?target=https%3A//github.com/mqyqingfeng/Blog)

TypeScript 系列是一个我都不知道要写什么的系列文章，如果你对于 TypeScript 有什么困惑或者想要了解的内容，欢迎与我交流，微信：「mqyqingfeng」，公众号：「冴羽的 JavaScript 博客」或者「yayujs」

如果有错误或者不严谨的地方，请务必给予指正，十分感谢。如果喜欢或者有所启发，欢迎 star，对作者也是一种鼓励。
[https://zhuanlan.zhihu.com/p/434016060](https://zhuanlan.zhihu.com/p/434016060)
