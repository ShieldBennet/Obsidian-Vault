在与后端开发同事对接API时, 同事问我:
> 你们前端是如何对JSON 数据进行encode/decode 的?

这个问题对一个纯前端工程师来说是有些"奇怪"的.

因为前端并不需要对JSON 进行encode/decode , 只需要对JSON string 进行parse.

parse 之后的数据便是JavaScript 中的数据结构, 这也是JSON 名字的由来: `JavaScript Object Notation`.

但由于JavaScript 的数据结构与其他编程语言并不一致, 比如JavaScript 中主要用`number` 类型代表数字, 但在Golang 中, 根据存储空间的不同, 将数字分为:

> `uint8`, `uint16`, `uint32`, `uint64`, `int8`, `int16`, `int32` , `int64` 等

所以在将JSON 转换为对应的编程语言的数据结构时, 需要声明JSON 与编程语言数据结构的对应关系, 然后再进行转换, 这个过程称为`encode`.

## TypeScript 中的类型

TypeScript 在设计之初便以兼容JavaScript 为原则, 所以JSON 也可以直接转换为TypeScript 中的类型.

比如有以下JSON 数据:
```JSON
{
  "gender": 0
}
```

该JSON可以对应到TypeScript 类型:
```TypeScript
enum Gender {  
  Female = 0,  
  Male = 1,  
}  
  
interface User {  
  gender: Gender;  
}
```

对应的parse 代码为:
```TypeScript
const user: User = JSON.parse(`{ "gender": 0 }`);
```

由于`JSON.parser`返回类型为`any`, 故在我们需要显示地声明`user`变量为`User`类型.

但是如果JSON 数据为:
```JSON
{
  "gender": 2
}
```

这个时候我们的parse 代码还是会成功运行, 但这个时候如果程序中我们还是按照类型声明那样将`gender`字段当做`0 | 1`的枚举, 那么便有可能导致严重的业务逻辑缺陷.

根本原因在于, TypeScript 不会对数据的类型进行运行时的检验, TypeScript 的类型基本上只存在于编译时.

这是众多BUG 的源头, 想以下以下场景:

- 后端的接口定义里将一个字段声明数组, 但实际上有的时候返回null, 前端没有对这个case 进行处理, 导致前端页面崩溃.
- 后端接口定义里, 将一个字段声明为required, 但实际上有的时候返回undefined, 前端没有对中case 进行处理, 页面上直接显示`username: undefined`.
- 后端说接口开发完了, 前端进行联调, 结果很多字段都与接口定义里不符合, QA的同事打开页面时, 页面直接崩溃了, 前端开发人员在群里被批评教育...

所以在有些场景下, 我们需要为IO(Input/Output, 比如网络请求, 文件读取)数据进行类型检验.

## io-ts

社区上有很多库提供了"对数据进行校验"这个功能, 但我们今天重点讲讲[io-ts](https://github.com/gcanti/io-ts).

io-ts 的特殊点在于:
- io-ts 的校验是与TypeScript 的类型一一对应的, 完备程度甚至可以称为TypeScript 的运行时校验.
- io-ts 使用的是`组合子`(combinator)作为抽象模型, 这与大部分`validator generator`有本质上的区别.

本文会着重带领读者实现io-ts 的核心模块, 是对"如何使用组合子进行抽象"的实战讲解.

### 基础抽象

作为一个解析器(或者称为校验器), 我们可以将其类型表示为:
```TypeScript
interface Parser<I, A> {
	parse: (i: I) => A
}
```

这个类型用`I`表示解析器的输入, `A`表示解析器的输出.

但这么设计有一个问题: 对于解析过程中的报错, 我们只能通过`副作用`(side effect)进行收集.

最直接的方式是抛出一个异常(Error), 但该方式会导致**整个**解析被终止.

我们希望能够将一个个"小"解析器组合成"大"解析器, 所以不希望"大"解析器中的某一个"小解析器"的失败, 导致整个"大"解析器被终止.

只有赋予解析器更灵活地处理异常的能力, 我们才能实现更加灵活的组合方式和收集错误日志.

> 此处可能有些抽象, 如果有所疑惑是正常现象, 结合下文理解会更加容易些.

因此, 我们希望"能够像处理数据那样处理异常", 这使得我们需要将类型修改为以下形式:

```TypeScript
interface Parser<I, E, A> {
	parse: (i: I) => A | E;
}
```

在这次修改中, 我们将异常像数据一样由函数返回, 类似于Golang 中的错误处理方式.

但直接通过`union type`进行抽象有一个弊端: 我们将难以分辨解析器返回的数据是属于成功分支的`A`呢, 还是失败分支的`E`呢?

尤其是在`A`和`E`使用同一种类型进行表示的时候, 会更加难以分辨和处理.

对此, 我们将通过`tagged unin type`进行抽象, 类型声明如下:

```TypeScript
interface Left<E> {  
  readonly _tag: 'Left';  
  readonly left: E;  
}  
  
interface Right<A> {  
  readonly _tag: 'Right';  
  readonly right: A;  
}  
  
type Either<E, A> = Left<E> | Right<A>;
```

通过在union type 的基础上增加一个标识符`tag`, 我们便能够更加便捷地对其进行区分和处理.

基于Either, 我们可以将Parser 的类型优化为:

```TypeScript
interface Parser<I, E, A> {
	parse: (i: I) => Either<E, A>;
}
```

## TypeScript 的类型系统

由于我们的最终目标是实现于TypeScript 类型系统一一对应的类型检查, 所以我们先理一理TypeScript 类型系统的(部分)基本机制.

首先是TypeScript 的primitive 类型:

```TypeScript
type Primitive = number | string | boolean;
```

然后是类型构造器:

```TypeScript
type Numbers = number[];
```

当然, 还有最重要的`object type`:

```TypeScript
interface Point{
  x: number;
  y: number;
}
```

此外, TypeScript 还实现了类型理论中的union type, intersect type 和  literal type:

```TypeScript
type Union = A | B;
type Intersect = A & B;
type Hello = "hello";
```

在余下篇幅中, 我们会一一实现这些类型对应的Parser. 

### 组合子

在实现这些类型的Parser 之前, 让我们先来了解一个概念 -- **组合子**.

组合子, 顾名思义, 就是对某种抽象的组合操作, 在本文中, 特指为对解析器的组合操作.

在TypeScript 中, 我们也是经常使用"组合" 的方式组合类型:

```TypeScript
type Union = A | B;
type Intersect = A & B;
```

在这个例子中, 我们使用 `|` 和 `&` 作为组合子, 将类型`A`和`B`组合成新的类型. 

同样的, Parser 也有其对应的组合子:
- union: P1 | P2 代表输入的数据通过两个解析器中的一个.
- intersect: P1 &  P2 代表输入的数据**同时**满足P1和P2两个解析器

#### union 组合子

该组合子类似于`or`运算:

```TypeScript
type Union = <MS extends Parser<any, any, any>[]>(ms: MS) =>  
    Parser<InputOf<MS[number]>, ErrorOf<MS[number]>, OutputOf<MS[number]>>;

type InputOf<P> = P extends Parser<infer I, any, any> ? I : never;  
  
type OutputOf<P> = P extends Parser<any, any, infer A> ? A : never;  
  
type ErrorOf<P> = P extends Parser<any, infer E, any> ? E : never;  
```

类型看起来有些复杂, 让我们自己看看这个类型的效果:

```TypeScript
declare const union: Union;  
declare const p1: Parser<string, string, number>;  
declare const p2: Parser<number, string, string>;  
const p3 = union([p1, p2]);
```

`p3`的类型被TypeScript为:

```TypeScript
Parser<string | number, string, string | number>
```

#### intersert 组合子

该组合子类似于`and`运算:

```TypeScript
type Intersect = <LI, RI, E, LA, RA>(left: Parser<LI, E, LA>, right: Parser<RI, E, RA>) => Parser<LI & RI, E, LA & RA>;
```





















































