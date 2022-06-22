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

该JSON 可以对应到TypeScript 类型:
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

> 由于`JSON.parser`返回类型为`any`, 故在我们需要显示地声明`user`变量为`User`类型.

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
- 后端说接口开发完了, 前端进行联调, 结果很多字段都与接口定义里不符合, QA 的同事打开页面时, 页面直接崩溃了, 前端开发人员在群里被批评教育...

所以在有些场景下, 我们需要为IO(Input/Output, 比如网络请求, 文件读取)数据进行类型检验.

## io-ts

社区上有很多库提供了"对数据进行校验"这个功能, 但我们今天重点讲讲[io-ts](https://github.com/gcanti/io-ts).

io-ts 的特殊点在于:
- io-ts 的校验是与TypeScript 的类型一一对应的, 完备程度甚至可以称为TypeScript 的运行时类型检查.
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

只有赋予解析器更灵活地处理异常的能力, 我们才能实现更加灵活的组合方式和错误日志的收集.

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

此外, TypeScript 还实现了类型理论中的union type, intersect type:

```TypeScript
type Union = A | B;
type Intersect = A & B;
```

在余下篇幅中, 我们会一一实现这些类型对应的Parser. 

### 组合子

在实现这些类型的Parser 之前, 让我们先来了解一个概念 -- **组合子**.

组合子, 顾名思义, 就是对某种抽象的组合操作, 在本文中, 特指为对解析器的组合操作.

如上是示例所示, 在TypeScript 中, 我们也是经常使用"组合" 的方式组合类型:

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

#### map 组合子

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

#### compose 组合子

在`Ramda` 中, 有一个常用的函数 -- [pipe](https://ramdajs.com/docs/#pipe), `compose`函数与其类似, 不同之处在于函数的组合顺序:

```TypeScript
pipe(f1, f2, f3);
```

等价于:

```TypeScript
compose(f3, f2, f1);
```

即, pipe 是从左到右结合, 而compose 是从右到左结合.

我们的Parser 也有类似的抽象, 为了简洁, 我们这里只给出其类型定义:

```TypeScript
type compose = <A, E, B>(ab: Parser<A, E, B>) => <I>(ia: Parser<I, E, A>) => Parser<I, E, B>;
```

#### fromArray 组合子

对应TypeScript 的`Array` 类型构造器, 我们的Parser 也同样需要类似的映射, 其类型声明如下:
```TypeScript
type FromArray = <I, E, A>(item: Parser<I, E, A>) => Parser<I[], E, A[]>;
```

从类型推断实现是函数式编程的经典做法, 我们不妨根据上述类型推断下`fromArray`的实现.

`fromArray`的返回值是`Parser<I[], E, A[]>`, 与此同时我们有参数`item: Parser<I, E, A>`, 那么我们可以对`I[]`的元素进行`item`进行parser 后得到`Either<E, A>[]`, 之后将`Either<E, A>[]`转换成`Either<E, A[]>`作为最终`Parser`的返回值.

这个类型转换具有通用性, 是函数式编程中的一个重要抽象, 在本节中会化一些篇幅对其推导, 最终将改抽象对应到Haskell 的`sequenceA`函数.

为了`Either<E, A>[] => Either<E, A[]>`的转换逻辑更加清晰, 我们不妨声明一个`type alias`并对其进行简化:

```TypeScript
type F<A> = Either<string, A>;
```

然后我们便可以将`Either<E, A>[] => Either<E, A[]>`简化为`Array<F<A>> => F<Array<A>>`, 为了使其更加泛用, 我们可以将`Array`替换为类型变量`T`, 得到`T<F<A>> => F<T<A>>`.

我们将伪代码`T<F<A>> => F<T<A>>`转换成Haskell 的类型签名, 即可得到:

```Haskell
t (f a) -> f (t a)
```

将此类型输入到[Hoogle](https://hoogle.haskell.org/?hoogle=t%20(f%20a)%20-%3E%20f%20(t%20a)), 我们看到这样一条类型签名:

> sequenceA :: [Applicative](https://hackage.haskell.org/package/base-4.16.1.0/docs/Prelude.html#t:Applicative "Prelude") f => t (f a) -> f (t a)

> 这段类型签名中的`Applicative f =>`是Haskell 中的类型约束, 在余下篇幅中会对其重点讲解, 可以暂时对其忽略.

即, Haskell 已经有我们所需要的类型转行的抽象, 函数名为`sequenceA`.

我们先记下有`sequnceA`这么个东西, 还有它是干什么的, 在余下篇幅中会进一步阐述.

#### fromStruct 组合子

`fromStruct`对应的是TypeScript 中的`interface`类型, 其类型定义如下:

```TypeScript
type FromStruct = <P extends Record<string, Parser<any, string, any>>>(properties: P) =>  
    Parser<{ [K in keyof P]: InputOf<P[K]> }, string, { [K in keyof P]: OutputOf<P[K]> }>;
```

> 为了简化类型声明, 上例中将`Parser<I, E, A>`中的`E`固定为`string`类型.

让我们检验下类型推断:

```TypeScript
declare const fromStruct: FromStruct;  
declare const p2: Parser<number, string, string>;  
const v = fromStruct({a: p2})
```

其中`v`被推断为: `Parser<{a: number}, string, {a: string}>`.

在实现层面上, 我们可以将其类型简化为`RecordLike<ParserLike<A>> => ParserLike<RecordLike<A>>`, 即:

```Haskell
t (f a) -> f (t a)
```

`fromStruct`和`fromArray`一样, 其实现最终导向了这个"奇怪"的类型转换, 接下来我们就深入这个类型签名, 讲讲其背后蕴含的理论.

#### Traversable和Applicative

我们再来看这个类型签名:

```Haskell
t (f a) -> f (t a)
```

这个类型的特征是转换后, `t`和`f`的位置发生了变化, 即, "里外翻转".

其实这种转换在JavaScript我们早已使用到了, 例如`Promise.all`方法:

```
all<T>(values: Array<Promise<T>>): Promise<Array<T>>;
```

让我们从`Promise.all`这个特例推导出这个函数的普遍性抽象.

`Promise.all`的执行逻辑(示例所用, 并非node底层实现)如下:
1. 创建一个空的`Promise r`, 并将其值设定为空数组: `Promise.resolve([])`
2. 尝试将`values`数组中的`Promise`的值一个个通过`Promise.then`串联`concat`进`Promise r`.
3. 返回`Promise r`

代码实现如下:
```TypeScript
const all = <A>(values: Array<Promise<A>>): Promise<A[]> => values.reduce(  
    (r, v) => r.then(as => v.then(a => as.concat(a))),  
    Promise.resolve([] as A[]),  
);
```

这个实现中使用了`Promise`的一些操作, 罗列如下:

- 依赖到的`Promise`类型的操作:
	- `Promise.resolve`
	- `Promise.then`

其中的`Promise.then`其实是兼具了`Fuctor.map`和`Monad.chain`实现.

`Functor`上文提到过, 让我们简单看看`Monad`.

``` TypeScript
interface Monad<F> extends Applicative<F>{
	chain: <A, B>(fa: F<A>, f: (a: A) => F<B>) => F<B>;
}
```

> 此为伪代码, TypeScript 不支持*higher kinded types*, 故这段代码在实际的TypeScript 中会报错.

`Promise.then`的两种用法分别对应`Functor.map`和`Monad.chain`:
- `then<A, B>(f: (a:A) => B): Promise<B>` 对应`Functor.map`
- `then<A, B>(f: (a:A) => Promise<B>): Promise<B>` 对应`Monad.chain`

`Monad`相比于`Functor`, 拥有更加"强大"的能力:

> 对两个嵌套上下文进行合并, 即`Promise<Promise<A>> => Promise<A>`的转换

在`Monad`的类型声明中, `Monad`还实现了`Applicative`:

```TypeScript
interface Applicative<F> extends Functor<F> {  
  of: <A>(a: A) => F<A>;  
  ap: <A, B>(fab: F<(a: A) => B>, fa: F<A>) => F<B>;
}
```

其中的`of`很好理解, 就是将一个值包裹进上下文中, 比如`Promise.resolve`.

而`ap`, 对于`Promise`可以将其实现为:

```TypeScript
const ap = <A, B>(ffab: Promise<(a: A) => B>, fa: Promise<A>): Promise<B> => fa.then(a => ffab.then(fab => fab(a)));
```

> 在函数式编程中, `Functor`, `Monad`, `Applicative`这样的类型构造器的类型约束称为`type class`, 而`Promise`这样的实现了某种`type class`的类型称为`instance of type class`.

如代码示例所示, `ap`可以通过`Monad.chain`实现, 那么其意义是什么?

答案是`Monad`是比`Applicative`更加"强大", 但也更加**严格**的约束.

一个函数, 对其依赖的类型拥有更加宽松的类型约束, 其使用场景也会更加广泛, 例如:

```TypeScript
type Move = (o: Animal) => void;
```

就比

```TypeScript
type Move = (o: Dog) => void;
```

使用场景更加广泛, 也更加合适, 即[最小依赖原则](https://www.wikiwand.com/en/Law_of_Demeter).

`Monad`比`Applicative`更加"强大"的点在于:

> `Applicative`能够对一系列上下文进行串联并且收集其中的值.
> `Monad`在`Applicative`的基础上, 能够基于一个上下文中的值, 灵活地创建另外一个包裹在上下文中的值. -- [stackoverflow上的回答](https://stackoverflow.com/a/14581288/6592925)

在`Promise.all`中, 我们其实只需要将`Promise`限定为`Applicative`:

```TypeScript
const all_ = <A,>(values: Array<Promise<A>>): Promise<A[]> =>  
  values.reduce(  
    (r, v) =>  
      ap(  
        map((as: A[]) => (a: A) => as.concat(a), r),  
        v,  
      ),  
    Promise.resolve([] as A[]),  
  );
```

这里的`Promise.all`便是`Promise`版的`sequenceA`实现, 同样的, 我们也可以使用同样的抽象实现`Parser`版的`sequenceA`, 此处留给读者自己去探索发现.

## 总结

本文简单讲解了`io-ts`实现背后的函数式编程原理.

但实际上, `io-ts`真实的实现运用了更多的设计, 比如`tag less final`, 报错类型也使用了其他的代数数据类型(`ADT`)等, 覆盖面之广, 是仅仅一篇博客无法讲完的.

有兴趣的读者推荐[这篇教程](https://github.com/enricopolanski/functional-programming).




















