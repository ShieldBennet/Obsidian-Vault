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
interface Parser<I, A, E> {
	parse: (i: I) => A | E
}
```

在这次修改中, 我们将异常像数据一样由函数返回, 类似于Golang 中的错误处理方式.

但直接通过`union type`进行抽象有一个弊端: 我们将难以分辨解析器返回的数据是属于成功分支的`A`呢, 还是失败分支的`E`呢?

尤其是在`A`和`E`使用同一种类型进行表示的时候, 会更加难以分辨和处理.

对此, 我们将通过`tagged unin type`进行抽象, 类型声明如下:

```TypeScript
interface Left<E> {  
  readonly _tag: 'Left'  
  readonly left: E  
}  
  
interface Right<A> {  
  readonly _tag: 'Right'  
  readonly right: A  
}  
  
type Either<E, A> = Left<E> | Right<A>
```

通过在union type 的基础上增加一个标识符`tag`, 我们便能够更加便捷地对其进行区分和处理.

### 组合子

组合子, 顾名思义, 就是对某种抽象的组合操作, 在本文中, 特指为对解析器的组合操作.

我们可以先捋一捋一些常规的组合操作:

- map: P <$> f 代表对解析器P1的结果进行f操作
- compose: P2 . P1 代表输入数据需要先经过P1 解析, 再经过P2 解析
- union: P1 <|> P2 代表输入的数据通过两个解析器中的一个.
- intersect: P1 &  P2 代表输入的数据**同时**满足P1和P2两个解析器

#### 串行运算

串行运算是一种常见的抽象, 比如JavaScript 中的`Promise.then`就是串行运算的经典例子:

```TypeScript
const inc = n => n + 1;
Promise.resolve(1).then(inc);
```

上面这段代码对`Promise<number>`进行了`inc`的串行运算.

既当`Promise`处于`resolved`状态时, 对其包含的`value: number`进行`inc`, 其返回结果同样为一个`Promise`.

若`Promise`处于`rejected`状态时, 不对其进行任何操作, 而是直接返回一个`rejected`状态的`Promise`.

我们可以脱离Promise, 进而得出`then`的更加泛用的抽象: 
> 对一个上下文中的结果进行进一步计算, 其返回值同样包含于这个上下文中, 且具有*短路*(short circuit)的特性.

在`Promise.then`中, 这个上下文既是"有可能成功的异步返回值".

得力于这种抽象, 我们可以摆脱`call back hell`和对状态的手动断言(GoLang 的`r, err := f()`).

让我们思考一下, 其实上文中提到的`Either`抽象同样符合这种运算:

1. 当`Either`处于成功的分支`Right`时, 对其进行进一步的运算.
2. 当Either处于失败的分支`Left`时, 直接返回当前的`Either`.

其实现如下:

```TypeScript
const map = <A, E, B>(f: (a: A) => B) =>  
  (fa: Either<E, A>): Either<E, B> => {  
    if (fa._tag === 'Left') {  
      return fa;  
    }  
    return {  
      _tag: 'Right',  
      right: f(fa.right),  
    };  
  };
```

值得注意的是, 这里我们将函数命名为`map`, 而非`then`, 这是为了符合函数式编程的[Functor](https://www.wikiwand.com/en/Functor)定义.

> Functor 是范畴论的一个术语, 在这里我们可以简单将其理解为"实现了map函数"的interface.

进一步地, Parser 同样符合"串行运算"的特质, 为了简洁, 我们这里只给出其类型定义:

```TypeScript
type map = <I, E, A, B>(f: (a: A) => B) => (fa: Parser<I, A, E>) => Parser<I, B, E>;
```














