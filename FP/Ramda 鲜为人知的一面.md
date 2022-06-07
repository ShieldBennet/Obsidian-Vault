在我们查阅[Ramda的文档](https://ramdajs.com/docs/)时, 常会见到一些"奇怪"的类型签名和用法:

"奇怪"的类型签名:  
```haskell
(Applicative f, Traversable t) => (a → f a) → t (f a) → f (t a)
```

某一些函数"奇怪"的用法:
```javascript
// R.ap can also be used as S combinator // when only two functions are passed 
R.ap(R.concat, R.toUpper)('Ramda') //=> 'RamdaRAMDA'
```

这些"奇怪"的点背后隐藏着Ramda 背后"更深"一层的设计, 本文将会对此作出讲解, 并阐述背后通用的函数式编程理论知识.

## Ramda 为人熟知的一面

Ramda 经常被当做Lodash 的另外一个"更加FP"的替代库.

相对于Lodash, Ramda 的优势(之一)在于柯里化和data last的设计带来的便捷的管道式编程(pipe).

举一个简单的代码对比示例:

Ramda: 
```javascript
const myFn = R.pipe (
  R.fn1,
  R.fn2 ('arg1', 'arg2'),
  R.fn3 ('arg3'),
  R.fn4
)
```

Lodash:
```javascript
const myFn = (x, y) => {
  const var1 = _.fn1 (x, y)
  const var2 = _.fn2 (var1, 'arg1', 'arg2')
  const var3 = _.fn3 (var2, 'arg3')
  return _.fn4 (var3)
}
```

*该示例节选之[Stackoverflow上的回答](https://stackoverflow.com/a/71403954/6592925)*

## Ramda 类型签名下鲜为人知的一面

在Ramda 的API文档中, 类型签名的语法有些"奇怪":

> add
```haskell
Number → Number → Number
```

我们结合Ramda 的柯里化规则, 稍加推测, 可以将这个函数转换为TypeScript 的定义:

```TypeScript
export function add(a: number, b: number): number;

export function add(a: number): (b: number) => number;
```

OK, 那为什么Ramda 的文档不直接使用TypeScript 表达函数的类型呢?

其实上面的示例已经部分回答了这个问题 -- 因为更加简洁.

其实Ramda 文档中的类型签名使用的是Haskell 的语法, Haskell 作为一门函数式编程语言, 其语法可以很简洁地表达柯里化的语义, 相较之下, TypeScript 的重载的表达方式就显得比较臃肿.

当然, 使用Haskell 的类型签名的意义不仅于此, 让我们再看看其他"奇怪"的函数类型:

> ap
```haskell
[a → b] → [a] → [b]
Apply f => f (a → b) → f a → f b
(r → a → b) → (r → a) → (r → b)
```

结合文档中的demo:

```javascript
R.ap([R.multiply(2), R.add(3)], [1,2,3]); //=> [2, 4, 6, 4, 5, 6]

R.ap([R.concat('tasty '), R.toUpper], ['pizza', 'salad']); //=> ["tasty pizza", "tasty salad", "PIZZA", "SALAD"] 

// R.ap can also be used as S combinator 
// when only two functions are passed 
R.ap(R.concat, R.toUpper)('Ramda') //=> 'RamdaRAMDA'
```

`[a → b] → [a] → [b]`我们好理解, 就是笛卡尔积.

`(r → a → b) → (r → a) → (r → b)`我们也能理解, 就是两个函数的串联.

`Apply f => f (a → b) → f a → f b`就有点难理解了, 语法上就有些陌生, 我们先将其翻译成TypeScript 语法:

:), 好吧, 这段类型没法简单地翻译成TypeScript, 因为:

> TypeScript 不支持将*类型构造器*作为类型参数.

举个例子:

```TypeScript
type T<F> = F<number>;
```

报错信息如下:

> Type 'F' is not generic.

在类型签名中`F`是一个类型构造器, 既和`Array`一样的*返回类型的类型*.

然而, TypeScript 里根本无法声明"一个类型参数为类型构造器".

正如示例中`type T<F> = F<number>;`中, 我们无法告诉TypeScript, 这里的`F`是一个类型构造器, 所以当将`number`传入`F`的时候, 就报错了.

OK, 我们假设TypeScript 支持声明"一个类型参数为类型构造器", 让我们再来看看`Apply f => f (a → b) → f a → f b`该怎么翻译:

```TypeScript
type AP = <F extends Appy, A, B>(f: F<((a: A) => B)>) => (fa: F<A>) => F<B>;
```

这里的`F`可以理解为一种*上下文*, 这段类型签名可以先简单地理解为:

> 将一个包裹在上下文中的**函数**取出, 再将另一个包裹在上下文中的**值**取出, 
> 调用函数后, 将函数的返回值重新包裹进上下文中并返回.

这里的*上下文*是一个泛指, 比如我们可以将其特异化(specialize)为Promise :

```TypeScript
type AP = <A, B>(f: Promise<((a: A) => B)>) => (fa: Promise<A>) => Promise<B>;  
  
const ap: AP = (f) => fa => f.then(ff => fa.then(ff));
```

`ap`或说`Apply`作为函数式编程中的一种常见抽象, 有着重要的学习意义, 但其抽象的解析超出本文范围, 在这里我们只聚焦于**是什么**, 暂不考虑**为什么**.

那么, `(r → a → b) → (r → a) → (r → b)`与`Apply f => f (a → b) → f a → f b`是什么关系?

他们之间是同父异母的关系, `(r → a → b) → (r → a) → (r → b)`是对`Apply f => f (a → b) → f a → f b`的特异化, 正如我们对Promise 做的那样.

函数也可以是一个*上下文*?

答案是可以的, 我们可以将一个一元函数`a -> b`理解为"一个包裹在上下文中的`b`, 只不过为了获取这个`b`, 需要先传入一个`a`.

为了减少语法噪音, 让我们先看看Haskell 对ap 的定义:

```haskell
instance Applicative ((->) r) where
    (<*>) f g x = f x (g x)
```

替换为TypeScript 的实现, 我们将上面的Promise 的例子稍微修改下, 得出:

```TypeScript
type F<A> = (a: any) => A;

type AP = <A, B>(f: F<((a: A) => B)>) => (fa: F<A>) => F<B>;  
  
const ap: AP = f => fa => {  
    return (r) => f(r)(fa(r));  
}
```

同样的, 我们得到Apply 特异化为Array 的实现:

```TypeScript
type AP = <A, B>(f: Array<((a: A) => B)>) => (fa: Array<A>) => Array<B>;

const ap: AP = f => fa => {
	return f.flatMap(ff => fa.map(ff));
};
```

综上所述, 我们可以得出结论:

ap的类型签名`[a → b] → [a] → [b]`和`(r → a → b) → (r → a) → (r → b)`是`Apply f => f (a → b) → f a → f b`的特异化.

##  可是为什么Ramda 要这么设计

本文只聚焦于"是什么", 至于"为什么", 这个我们留到下一篇?.再讲. 