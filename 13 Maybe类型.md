# Maybe类型

本章介绍Maybe类型, 同时从Maybe实现几种类型类的方式上看看几种类型类的意义.

## 类型定义

我们之前也多少用到过`Maybe`, 现在正式的介绍一下:

```haskell
data Maybe a = Nothing | Just a
```

它有两个构造子, `Nothing`和`Just`, 其中`Nothing`不需要参数, `Just`需要一个类型a的参数, a是泛型.

## 函数提升

我们再看一下函子的定义:

```haskell
class Functor f where
  map :: forall a b. (a -> b) -> f a -> f b
```

这意味着, 若某个类型`f`实现了`Functor`, 则存在一个函数map, 输入(a到b的函数), 就会得到另一个函数, 这个函数输入(f a)类型的值, 返回(f b)类型的值.

我们的`Maybe`实现了`Functor`, 所以对于`Maybe`来说, 它的`map`是:

```haskell
map :: forall a b. (a -> b) -> Maybe a -> Maybe b
```

也就是说, 输入一个`(a->b)`的函数给`map`, 就会得到一个`(Maybe a -> Maybe b)`的函数.

不过, 其他类型也可以实现函子, 例如, List也实现了函子, 对于List来说, map的是:

```haskell
map :: forall a b. (a -> b) -> List a -> List b
```

也就是说, 输入一个`(a->b)`的函数给`map`, 就会得到一个`(List a -> List b)`的函数.

但问题是, 一个函数怎么能既是`(Maybe a -> Maybe b)`, 又是`(List a -> List b)`?

答案是, 泛型, 我们会得到一个泛型函数, 来举个例子:

我们写一个`a->b`的函数, 随便写一个没什么意义的, 比如:

```haskell
fun :: Int -> String
fun _ = "a"
```

那么这里, 我们的`a`就是`Int`, `b`就是`String`:

```haskell
> import Main
> import Data.Maybe
> import Data.List
> :t fun
Int -> String
> :t map fun
forall (t1 :: Type -> Type). Functor t1 => t1 Int -> t1 String
> :t map fun (Just 1)
Maybe String
> :t map fun (1:2:3:Nil)
List String
```

可以看到:

- `fun`的类型是`String -> Int`.
- `map fun`的类型是`t1 Int -> t1 String`, 这是泛型函数, 在实际调用的时候, `t1`才会被确定.
- `map fun (Just "abc")`的类型是`Maybe String`, 因为`t1`已经被确定为了`Maybe`, 所以返回的`t1 String`自然是`Maybe String`.
- `map fun (1:2:3:Nil)`的类型是`List String`, 因为`t1`已经被确定为了`List `, 所以返回的`t1 String`自然是`List String`.

至少从结果来说, 将一个`(a->b)`的函数输入`map`, 就可以得到一个`(Maybe a -> Maybe b)`的函数. 甚至是任何实现函子的类型.

这称为函数提升, 将一个普通函数提升到了`Maybe`范畴.

函数提升的话题稍后还会提到.

## 推测

再看看这个签名:

```haskell
map :: forall a b. (a -> b) -> Maybe a -> Maybe b
```

因为柯里化, 也可以理解为, 输入一个`a->b`的函数和一个`Maybe a`的数据, 返回一个`Maybe b`的数据.

进而我们甚至可以猜测: 它可能是先将`Maybe a`解包, 得到`a`, 然后调用输入的函数, 得到了`b`, 再包装回`Maybe`中.

虽然猜测是没问题的, 通常我们也依靠类型签名来推断它做了什么, 事实上`Maybe`的实现也确实如此, 但并不是所有类型都可以这样猜出来.

原理上, 除非你看过某个类型的类型实现或文档, 否则你不会知道某个泛型在这个类型中扮演什么角色.

除非你看过函数实现或文档, 否则你也不会知道函数到底在做什么.

光看签名有个函数就以为是通过输入函数处理的, 但实际实现是否用到了传入的函数都是问号.

我们常说的拆包打包也只是方便讲解, 并不是所有类型都如此.

但幸运的是, 并不需要熟悉所有的函数, 一个好的类型是一种模型, 一切操作都是自然的, 很快我们会看到这一点.

## 类型建模

一种类型不应该被无端构造出来, 一个类型表示一种模型, 一种抽象.

我们只有理解了类型在抽象什么后, 才能理解类型会实现什么类型类, 怎样实现这些类型类.

这里我们先讨论Maybe, Maybe是抽象"可能失败的数据"的模型.

在编程中我们遇到这种情况还挺多的, 比如去读一个数据库数据, 可能读出来了, 也可能数据没找到, 也可能发生错误.

传统上, 我们会进行数次判断, 可能还会使用`try`将异常捕获, 抛出.

在这里, 我们将这种可能性抽象成为`Maybe`: `Maybe`是一个`容错的值`, 它可能是你需要的数据, 也可能是Nothing.

而`Maybe`的泛型就表示你需要的数据的类型.

当你调用数据库查询时, 他会返回给你一个`Maybe`类型的值, 只是这个值有点特殊, 里面可能是你读回来的数据, 也可能是空的.

那要怎么使用这个值呢?就通过函数了,而这个函数通常会归在某种类型类里. 比如:

- 因为`Maybe`实现了函子, 所以你可以用`map`操作它, 而从`map`的签名可以看出来, 你操作之后, 它还是个`Maybe`, 这样蛮合理的.
- 因为`Maybe`实现了单子, 所以你可以用`bind`把它里面的, 你需要的值取出来, 进行运算, 塞回`Monad`里.(我们之后会讨论到).

而你不需要担心它是不是空的, 是不是出错了, `Maybe`对这些操作的实现会保证, 如果出错了, 无论你怎么计算, 它都是空.

这样, 就算出错了, 你得到的也只是一个空而已, 不会出现运行时错误, 不需要一次一次检查异常.

## Maybe实现的函子

刚才说过, map可以将a->b的函数提升到`(Maybe a -> Maybe b)`范畴, 就是说:

- 正常情况下, 我获得一个普通的值, 然后我可以用a->b的函数操作它.
- 现在, 我获得了一个Maybe值, 但是没关系, 只要把原来的函数用map处理一下, 就可以像以前一样操作这个Maybe值了.

为什么我们可以说是"像以前一样操作"呢? 因为Maybe想表达的模型本来就是`容错的值`的概念.

容错的值, 就是说基本上还是以前的值, 不同是, 如果值出错了, 这个值没读到, `Maybe`也保证对它运算不会报错而已.

所以, 它**应该**具有"像以前一样操作"的特性, 不然就不合理.

那么来看看`Maybe`的实现, 也确实如此:

```haskell
instance functorMaybe :: Functor Maybe where
  map fn (Just x) = Just (fn x)
  map _  _        = Nothing
```

如果值是`Just`构造的, 那么把值模式匹配出来, 用函数调用, 结果再塞回`Just`.

如果值是`Nothing`, 直接返回`Nothing`.

嗯...是不是所有实现函子类型类的类型都可以map后"像以前一样操作"呢?

因为函子这个抽象来自于范畴论,应该去考察范畴论是怎么想的,我还没看完,也不知道orz.

## Maybe实现的应用函子

上面我们说到, `map`可以将`a->b`的函数提升为`f a->f b`的函数. 那么, 如果我要提升`a->b->c`的函数呢?

你可以用`map`的组合试试, 但总之, 做不到.

但我们有另一个类型类可以解决这个问题, 称为应用函子(`Apply`).

它的定义:

```haskell
class Functor f <= Apply f where
  apply :: forall a b. f (a -> b) -> f a -> f b
```

这样就可以解决我们的问题了.

我们的问题是, 如何把`a->b->c`的函数变成`f a -> f b -> f c`的函数.

那么假设现在有这样一个函数, 叫`fun`. 还有它的两个参数, x和y, 那么x应该是f a类型的值, y应该是f b类型的值.

那么, 考虑这个式子:

```haskell
apply (map fun x) y
```

先看`map fun x`, 这意味着把函数fun提升, 然后用x调用.

因为柯里化, fun可以看作`a->(b->c)`, 对它用`map`提升就会得到`f a -> f (b->c)`, 再拿x调用它, 会得到`f (b->c)`.

这时, 再看`apply`, 它接受一个`f (a->b)`的函数, 返回一个`f a->f b`的函数.

注意, 这里的`a`和`b`不是上面的`a`和`b`, 转换一下的话, 这里的`a`是`b`, `b`是`c`.

于是`apply (f (b->c))`会得到`(f b->f c)`, 此时再传入`y`, 因为`y`正好是`f b`类型的值, 类型匹配, 最后会得到`f c`.

把这个过程封装一下:

```haskell
lift2 :: forall f a b c. Apply f => (a -> b -> c) -> f a -> f b -> f c
lift2 f a b = f <$> a <*> b
```

其中, `<$>`是`map`的中缀符号, `<*>`是`apply`的中缀符号.

那么, 这个`lift2`就是一个输入`(a -> b -> c)`, 返回`f a -> f b -> f c`的函数了.

如果想提升`a->b->c->d`呢? 我们有`lift3`, 以此类推. 那么, `lift3`的写法是:

```haskell
lift3 :: forall a b c d f. Apply f => (a -> b -> c -> d) -> f a -> f b -> f c -> f d
lift3 f a b c = f <$> a <*> b <*> c
```

这些函数都在`Control.Apply`模块里.

和上面说的一样, `Apply`也只能保证这个形式, 而具体如何实现, 会有怎样的效果, 是取决于类型如何实现它的.

幸运的是, 类型并不是随心所欲来实现类型类的, 如果一个类型想要有用, 它必须是某种建模的反应, 我们可以通过把握类型的模型来把握它对类型类的实现.

对于`Maybe`, 它只是一个`容错的值`, 基本和普通值一样, 唯一的不同就是, 如果计算牵扯到`Nothing`就直接返回`Nothing`.

那么因为`Maybe`建模的想法, 对于普通值`x`,`y`,`z`, 和函数`fun`, (其中`x`是`a`类型的值, `y`是`b`类型的值, `z`是`c`类型的值, `fun`是`a->b->c`类型的函数):

他应该保证:

- 若`x`,`y`调用函数`fun`, 得到`z`. 则`Just x`, `Just y`, 调用函数`lift2 fun`, 应该得到`Just z`.
- 而如果调用`lift2 fun`的参数有一个或以上是`Nothing`, 那应该得到`Nothing`.

而`Just x`, `Just y`, 调用函数`lift2 fun`, 应该得到`Just z`, 参考上面讨论的`lift2`的实现, 意味着:

```haskell
apply (map fun (Just x)) (Just y) = Just z

-- 看看map的实现: map fn (Just x) = Just (fn x)
-- 所以这里: map fun (Just x) 可以替换为 Just (fun x), 得到:
apply (Just (fun x)) (Just y) = Just z

-- 我们设 fn = fun x
apply (Just fn) (Just y) = Just z

-- 同时我们知道fun x y=z, 也就是说fn y=z, 替换等式右边:
apply (Just fn) (Just y) = Just (fn y)

-- 由于map的实现: map fn (Just x) = Just (fn x), 反过来用, 那么右边可以替换:
apply (Just fn) (Just y) = map fn (Just y)

-- 我们知道
--   fun的类型是a->b->c, 那么fn的类型是b->c, (Just fn)的类型就是Maybe b->c
--   y的类型是b, 那么(Just y) 的类型是Maybe b
--   fn y的类型是c, 那么Just (fn y)的类型是Maybe c, 那么与之等价的map fn (Just y)的类型也是Maybe c
-- 将上面的符号替换, b替换成a, c替换成b, 写成签名, 得到:
apply :: Maybe (a -> b) -> Maybe a -> Maybe b
apply (Just fn) (Just y) = map fn (Just y)

-- 这就是Maybe对apply的实现, 签名也对的上.

-- 还可以把两边的Just y同时用一个符号替代:
apply :: Maybe (a -> b) -> Maybe a -> Maybe b
apply (Just fn) x = map fn x
```

所以Maybe对应用函子的实现是:

```haskell
instance applyMaybe :: Apply Maybe where
  apply (Just fn) x = fn <$> x
  apply Nothing   _ = Nothing
```

如果是`Nothing`, 直接返回`Nothing`.

如果是`Just`构造的, 把函数取出来, 提升, 调用, 最后得到`Maybe b`类型.

但能这样做的前提是, `x`必须实现函子类型, 所以在声明类型类的时候有约束:`class Functor f <= Apply f where`.

## Maybe对Bind的实现

这里的`Bind`, 表示用`a->m b`类型的函数操作类型的能力.

类型类`Bind`的定义:

```haskell
class Apply m <= Bind m where
  bind :: forall a b. m a -> (a -> m b) -> m b

infixl 1 bind as >>=
```

考虑我们有两个函数:

```haskell
f1 :: c -> m a
f2 :: a -> m b
```

这种情况还挺多的, 比如之前举例的, 读数据库, 你可能输入一个条件, 然后这个还是返回给你一个`Maybe`值.

那么, 现在我想将这两个函数组合, 把`f1`的结果输入给`f2`去用, 最后得到`f2`的返回结果.

但这是不可以的, 因为`f1`的返回值类型是`m c`, 而`f2`的参数类型是`c`.

当然你可以用`map`试试, 不过最后你会得到`m (m b)`.

而`bind`可以解决这个问题, 我们假设x是一个c类型的值:

```haskell
bind (f1 x) f2
```

其中, `(f1 x)`的类型是`m a`, `f2`的类型是`(a -> m b)`

参考`bind`的签名, 可以知道这个式子得到的结果的类型是`m b`.

如果是多个函数呢? 比如:

```haskell
f1 :: a -> m b
f2 :: b -> m c
f3 :: c -> m d
```

那么,(设`x`为`a`类型的值):

```haskell
bind (f1 x) (\b -> bind (f2 b) f3)
```

因为`bind`的中缀符号是`>>=`, 所以可以写成:

```haskell
f1 x >>= f2 >>= f3
```

这还有另一种写法:

```haskell
do
  i <- f1 x
  j <- f2 i
  k <- f3 j
  pure k
```

这里`do`是语法糖, 等价于:

```haskell
f1 x >>= \i -> f2 i >>= \j -> f3 j >>= \k -> pure k
```

不过直接看`do`表达式的形式, 可以有另一种理解:

注意到, `f1 x`是`m b`类型, 而i是b类型, 下面也一样, f2 i是mc类型, j是c类型.

那么`<-`是一种转换, 将ma类型转换为a类型. 当然, 具体是怎么转换的是由类型确定的.

另外,bind的意思是"输入一个m a的值和a->b的函数,得到一个m b的值",do表达式用到的"<-"则是"将ma类型转换为a类型",

他们其实是等价的,可以互相转换,不信只要看看do的语法糖:

```haskell
f1 x >>= \i -> f2 i >>= \j -> f3 j >>= \k -> pure k

-- 把中间表达式的变量都约掉
f1 x >>= f2 >>= f3 >>= pure

-- pure是将一个a类型的值转换为ma类型, 正好和<-相反, 这里我们可以直接把他去掉.
-- 这就是用bind写的m函数组合嘛
f1 x >>= f2 >>= f3
```

