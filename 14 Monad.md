# Monad

本章介绍Monad的几个相关应用, Monad的概念已经在12章讲过, 如果感到困惑可以回去看.

## 读文件

因为purescript最终会编译为js, 如果你用node执行最终的js的话, 就可以使用node的api.

于是有node的api的[相关封装](https://pursuit.purescript.org/packages/purescript-node-fs/5.0.1/docs/Node.FS.Async).我们来使用里面的[读文件操作](https://pursuit.purescript.org/packages/purescript-node-fs/5.0.1/docs/Node.FS.Sync#v:readTextFile).

为了简单, 我们使用的是同步的读文件, 先看他的签名:

```haskell
readTextFile :: Encoding -> FilePath -> Effect String
```

很简单, 输入编码, 文件路径, 最后返回一个带有`Effect`上下文的字符串, `Effect`就是IO, Unit就是Void, 写法不一样而已.

试试:

```haskell
module Main where

import Prelude

import Effect (Effect)
import Effect.Console (log)
import Node.Encoding (Encoding(..))
import Node.FS.Sync (readTextFile)

main :: Effect Unit
main = do
  s <- readTextFile UTF8 "file"
  log $ show $ s
```

就像前面所说的, do是monad相关的语法糖, 本质上, do会返回一个多层嵌套的monad数据, 这里这个实现monad的数据是Effect.

### 异常处理

那么如果文件不存在呢? 会报错:

```shell
internal/fs/utils.js:314
    throw err;
    ^

Error: ENOENT: no such file or directory, open 'file'
    at Object.openSync (fs.js:498:3)
    at Object.readFileSync (fs.js:394:35)
    at D:\code\test_pureScript\output\Node.FS.Sync\index.js:43:23
    at Object.__do [as main] (D:\code\test_pureScript\output\Main\index.js:8:72)
    at Object.<anonymous> (D:\code\test_pureScript\.spago\run.js:3:48)
    at Module._compile (internal/modules/cjs/loader.js:1072:14)
    at Object.Module._extensions..js (internal/modules/cjs/loader.js:1101:10)
    at Module.load (internal/modules/cjs/loader.js:937:32)
    at Function.Module._load (internal/modules/cjs/loader.js:778:12)
    at Function.executeUserEntryPoint [as runMain] (internal/modules/run_main.js:76:12) {
  errno: -4058,
  syscall: 'open',
  code: 'ENOENT',
  path: 'file'
}
```

毕竟是js代码嘛.

但我们有一个try函数可以包装这种错误:

```haskell
main :: Effect Unit
main = do
  s <- try $ readTextFile UTF8 "file"
  log $ show $ s
```

看看得到了什么:

```shell
(Left Error: ENOENT: no such file or directory, open 'file'
    at Object.openSync (fs.js:498:3)
    at Object.readFileSync (fs.js:394:35)
    at D:\code\test_pureScript\output\Node.FS.Sync\index.js:52:23
    at D:\code\test_pureScript\output\Effect\foreign.js:12:16
    at D:\code\test_pureScript\output\Effect\foreign.js:12:20
    at D:\code\test_pureScript\output\Effect.Exception\foreign.js:37:16
    at Object.__do [as main] (D:\code\test_pureScript\output\Main\index.js:9:97)
    at Object.<anonymous> (D:\code\test_pureScript\.spago\run.js:3:48)
    at Module._compile (internal/modules/cjs/loader.js:1072:14)
    at Object.Module._extensions..js (internal/modules/cjs/loader.js:1101:10))
```

是一个左值.

另外我们也可以自己抛出异常:

```haskell
exceptionHead :: List Int -> Effect Int
exceptionHead l = case l of
  x : _ -> pure x
  Nil -> throwException $ error "empty list"
```

但不推荐这样做, 推荐用Either,Maybe之类的管理错误.

## STmonad

在通常的编程语言里, 变量意味着一个内存地址, 对变量的操作意味着对该内存地址值的操作.

共享内存地址是不安全的, 这意味着, 你不应该返回一个在函数里创建的引用对象.

但通常编程语言都不会进行这种检查, 你需要自己注意.

在这里, 我们有[ST类型](https://pursuit.purescript.org/packages/purescript-st/5.0.1/docs/Control.Monad.ST), 允许你使用内存引用, 又能保证安全.

这里有两种类型, ST和STRef. STRef是内存地址的引用, 对应一般编程语言的变量. ST是它的包装.

操作它包含几个函数:

```haskell
new :: forall a r. a -> ST r (STRef r a)
read :: forall a r. STRef r a -> ST r a
write :: forall a r. a -> STRef r a -> ST r a
modify :: forall r a. (a -> a) -> STRef r a -> ST r a
run :: forall a. (forall r. ST r a) -> a
```

new是新建一个变量, a是变量的类型, 另外虽然声明了r是自由变量, 但却不需要给它赋值, 那么r是一个`幻影类型`.

在这里, 这个幻影类型可以保证ST r里一定装着(STRef r a)类型的值, 他们的r是一个类型.

run用来计算ST效果, 也就是获得内存引用, 注意, 它的签名很奇怪. 这叫`Rank2Types`, 允许你以泛型的方式定义函数的参数的类型.

ST是一个Monad, 我们可以取它的值, 所以可以写成这样:

```haskell
fun :: Int
fun = run do
  v1 <- new 0
  v2 <- write 1 v1
  pure v2
```

在do这个区域里, 你可以像使用变量一样过程式编程, 当然也有循环之类的, 在ST模块都有.

do区域返回一个ST, 最后我们用run执行它. 当然你也可以把ST作为函数的返回, 然后在函数外使用run.

但你不可以返回STRef, 考虑这个例子:

```haskell
fun :: forall r. STRef r Int
fun = run (new 0)
```

我通过run得到一个`STRef r Int`类型的值, 这个值是危险的, 他直接意味着内存引用. 接下来我把他作为返回值.

但这样会得到错误, 因为run函数说明了它要计算的ST r a类型中, r是一个自由变量, 而我们fun上声明的r也是一个自由变量, 这两个不是一个东西.

我无法在函数类型签名上定义一个和`ST r a`相同的那个`r`的`STRef r a`, 意味着new出来的这个内存引用不能返回给其他函数. 要返回一定要带着ST.

这就保证了内存引用不会共享, 保证了安全.

## 读monad

这个模块: [Control.Monad.Reader - purescript-transformers - Pursuit](https://pursuit.purescript.org/packages/purescript-transformers/5.1.0/docs/Control.Monad.Reader).

读monad有两个参数, 一个是环境的值r, 一个是最后的返回值a.

读monad允许你计算monad时传入一个"环境值", 这个值在读monad中全局可见.

这是一个例子:

```haskell
module Main where

import Prelude

import Control.Monad.Reader (Reader, ask, runReader)
import Effect (Effect)
import Effect.Console (log)

fun :: Reader { a :: Int } Int
fun = do
  x <- ask
  pure $ x.a + 1 

main :: Effect Unit
main = do
  log $ show $ runReader fun {a:1}
```

会得到2.

在读monad中可以通过ask获得环境的值.

最后通过runReader来计算, 另外还可以用withReader来改变环境值, 具体参考文档.

如果合并两个读monad呢?

```haskell
module Main where

import Prelude

import Control.Monad.Reader (Reader, ask, runReader)
import Effect (Effect)
import Effect.Console (log)

fun1 :: Reader { a :: Int } Int
fun1 = do
  x <- ask
  pure $ x.a + 1

fun2 :: Reader { a :: Int } Int
fun2 = do
  x <- ask
  pure $ x.a + 2

main :: Effect Unit
main = do
  log $ show $ runReader (do
    _ <- fun1
    fun2
  ) {a:1}
```

会得到3.

也就是合并没啥用, 会以后面那个为准.

## 写monad

这个模块: [Control.Monad.Writer - purescript-transformers - Pursuit](https://pursuit.purescript.org/packages/purescript-transformers/5.1.0/docs/Control.Monad.Writer#t:Writer).

写monad有两个值, 一个累加值w和一个结果值a.

在计算monad的时候, 会将所有累加值通过Monoid的接口合并.

看一个例子:

```haskell
module Main where

import Prelude

import Control.Monad.Writer (Writer, runWriter)
import Control.Monad.Writer.Class (tell)
import Effect (Effect)
import Effect.Console (log)

val1 :: Writer (Array String) Int
val1 = do
  tell ["1"]
  pure 1

main :: Effect Unit
main = do
  log $ show $ runWriter val1 -- 得到(Tuple 1 ["1"])
```

如果合并多个写monad:

```haskell
module Main where

import Prelude

import Control.Monad.Writer (Writer, runWriter)
import Control.Monad.Writer.Class (tell)
import Effect (Effect)
import Effect.Console (log)

val1 :: Writer (Array String) Int
val1 = do
  tell ["1"]
  pure 1

val2 :: Writer (Array String) Int
val2 = do
  tell ["2"]
  pure 2

main :: Effect Unit
main = do
  log $ show $ runWriter (do
    _ <- val1
    val2
  )
-- 得到(Tuple 2 ["1","2"])
```

runWriter是同时计算累加值和结果, 还有execWriter之类的只计算累加值, 丢弃结果.

具体可以参考文档.

## 状态monad

这个模块: [Control.Monad.State - purescript-transformers - Pursuit](https://pursuit.purescript.org/packages/purescript-transformers/5.1.0/docs/Control.Monad.State).

状态monad有两个值, 一个状态值s和一个结果值a.

同时要操作状态monad内部的状态, 还需要这几个函数: [Control.Monad.State.Class - purescript-transformers - Pursuit](https://pursuit.purescript.org/packages/purescript-transformers/5.1.0/docs/Control.Monad.State.Class).

看一个例子:

```haskell
module Main where

import Prelude

import Control.Monad.State (State, put, runState)
import Effect (Effect)
import Effect.Console (log)

val1 :: State Int Int
val1 = do
  put 1
  pure 1

main :: Effect Unit
main = do
    log $ show $ runState val1 0
```

会得到`(Tuple 0 1)`.

定义了一个状态monad, 它的状态类型是Int, 返回值是Int.

put可以设置monad的状态.

runState可以执行状态monad, 当然还有evalState, execState之类的可以丢弃状态或者值, 还有withState可以改变状态.

再看这个例子:

```haskell
module Main where

import Prelude

import Control.Monad.RWS (get)
import Control.Monad.State (State, put, runState)
import Effect (Effect)
import Effect.Console (log)

val1 :: State Int Int
val1 = do
  put 1
  pure 0

val2 :: State Int Int
val2 = do
  a <- get
  pure $ a + 1

main :: Effect Unit
main = do
  log $ show $ runState (do
    _ <- val1
    val2
  ) 0
```

会得到`(Tuple 2 1)`.

状态monad可以合并, 合并后他们就共享一个状态, 可以用get获取状态, modify修改状态之类的.

## 单子转换器

以状态单子做例子, 有一个类型是`StateT`, 它是`State`的扩展.

它的完整类型是`StateT s m a`, s是状态类型, m是结果包裹, a是结果.

一个例子:

```haskell
module Main where

import Prelude

import Control.Monad.Cont (lift)
import Control.Monad.State (StateT)
import Control.Monad.State.Trans (get, put, runStateT)
import Data.Maybe (Maybe(..))
import Effect (Effect)
import Effect.Console (log)

val1 :: StateT String Maybe Int
val1 = do
  s <- get
  put "nihao"
  case s of
    "" -> lift $ Nothing
    _ -> lift $ Just 1

main :: Effect Unit
main = do
  log $ show $ runStateT val1 "" -- 得到 Nothing
  log $ show $ runStateT val1 "abc" -- 得到 (Just (Tuple 1 "nihao"))
```

对于`StateT s m a`(这里是`StateT String Maybe Int`), 当runStateT时, 会得到`m (Tuple a s)`, 这里就是`Maybe (Tuple Int String)`. 

当然也有对应的execStateT, evalStateT.

你依然可以使用get等函数操作状态, 状态的类型是s, 这里是String.

而注意到, 整个计算的结果是`包装类型 (Tuple 结果类型 状态类型)`. 就是说把结果做了一层包装.

相对的, 在返回结果的时候, 就不能单写`m a`类型(这里是`Maybe Int`类型)的值, 需要多一层lift.

这个lift在`MonadTrans`中定义, 将`m a`包装成`t m a`. 在这里就是把`Maybe Int`包装为`StateT String Maybe Int`.

事实上, `State s a`的就是`StateT s Identity a`. 是StateT的一种特殊形式.

那么如果把两个`StateT String Maybe Int`组合呢? 

```haskell
module Main where

import Prelude

import Control.Monad.Cont (lift)
import Control.Monad.State (StateT)
import Control.Monad.State.Trans (get, put, runStateT)
import Data.Maybe (Maybe(..))
import Effect (Effect)
import Effect.Console (log)

val1 :: StateT String Maybe Int
val1 = do
  put "a"
  lift $ Just 1

val2 :: StateT String Maybe Int
val2 = do
  s <- get
  put $ s <> "b"
  lift $ Just 2

main :: Effect Unit
main = do
  log $ show $ runStateT (do
    _ <- val1
    val2
  ) ""
-- 得到 (Just (Tuple 2 "ab"))
-- 如果val1或val2返回lift Nothing, 则结果为Nothing.
```

StateT就是把状态monad和一个输入上下文合在一起, 那为什么要这么做呢? 因为这样可以方便组合.

改写上面组合的示例, 如果我不使用StateT, 而使用State:

```haskell
module Main where

import Prelude

import Control.Monad.State (State, get, runState)
import Control.Monad.State.Trans (put)
import Data.Maybe (Maybe(..))
import Effect (Effect)
import Effect.Console (log)

val1 :: State String (Maybe Int)
val1 = do
  put "a"
  pure $ Just 1

val2 :: State String (Maybe Int)
val2 = do
  s <- get
  put $ s <> "b"
  pure $ Just 2

main :: Effect Unit
main = do
  log $ show $ runState (do
    _ <- val1
    val2
  ) ""
-- 得到 (Tuple (Just 2) "ab")
```

虽然上下文的顺序不同, 但更重要的是, 这里获得的结果永远是val2的结果, 即使val1返回Nothing, 也会得到`(Tuple (Just 2) "ab")`而不是`Nothing`.

StateT允许你在结果上套一层上下文, 使用do表达式, 一方面你可以正常的共享状态, 另一方面你又可以用给定的上下文对结果进行组合.

而State的结果没有上下文, do表达式只能共享状态, 结果是不会组合的, 结果值只会取最后一个计算的结果.

## 类型分析

我们先停一下, 研究以下这些monad的类型.

拿StateT来做例子, 我们已经会使用它了, 总之就是声明一个返回`StateT s m a`的值, 其中s是状态值, a是结果值, m是包装类型.

然后你可以写它的表达式, 表达式可以写一句话, 也可以写很多, 总之只要最后返回`StateT s m a`就可以了. 怎么返回? 他提供了lift给你.

特别的是, 写表达式的时候, 我们可以先用do语法糖写出一个代码块, 在这个代码块里可以用put, get之类的函数来操作状态.

这是怎么做到的?

来看一个例子:

```haskell
module Main where

import Prelude

import Control.Monad.Cont (lift)
import Control.Monad.State (StateT, runStateT)
import Control.Monad.State.Trans (put)
import Data.Maybe (Maybe(..))
import Effect (Effect)
import Effect.Console (log)

val1 :: StateT String Maybe Int
val1 = do
  put "nihao"
  lift $ Just 1

main :: Effect Unit
main = do
  log $ show $ runStateT val1 ""
```

把do写成一般的形式:

```haskell
val1 :: StateT String Maybe Int
val1 = bind (put "nihao") (\_ -> lift $ Just 1)
```

我们知道bind是类型类规定的函数, 在这里:

```haskell
class Apply m <= Bind m where
  bind :: forall a b. m a -> (a -> m b) -> m b
```

就是说对于某个类型m, 要有`bind :: forall a b. m a -> (a -> m b) -> m b`, 对于StateT来说, 实现是:

```haskell
instance monadStateT :: Monad m => Monad (StateT s m)
```

就是说, 有`bind :: forall a b. StateT s m a -> (a -> StateT s m b) -> StateT s m b`.

所以`(put "nihao")`应该是`StateT s m a`, `(\_ -> lift $ Just 1)`应该是`(a -> StateT s m b)`, 而最后的返回值应该是`StateT s m b`.

而我们知道最终返回值应该是`StateT String Maybe Int`, 所以我们知道s是String, m是Maybe, b是Int.

所以bind的签名是`forall a. StateT String Maybe a -> (a -> StateT String Maybe Int) -> StateT String Maybe Int`.

那么`(put "nihao")`是`StateT String Maybe a`, `(\_ -> lift $ Just 1)`是`(a -> StateT string Maybe Int)`. 其中a还不确定是什么.

接下来考察put函数, 看看签名(为了不和上面的符号混淆, 我们把符号都改了一下):

```haskell
put :: forall m1 s1. MonadState s1 m1 => s1 -> m1 Unit
```

输入一个s1类型的值, 返回一个m1 Unit类型的值.

我们输入的是"nihao", 所以s1就是String了, 我们又知道在我们的情况里`(put "nihao")`是`StateT String Maybe a`. 所以可以推理出以下信息:

- m1是`StateT String Maybe`.
- MonadState String (StateT String Maybe)应该被实现.
- a是Unit.
- bind的签名是`StateT String Maybe Unit -> (Unit -> StateT String Maybe Int) -> StateT String Maybe Int`.

先看看MonadState String (StateT String Maybe)是否真的被实现了:

```haskell
instance monadStateStateT :: Monad m => MonadState s (StateT s m) where
  state f = StateT $ pure <<< f
```

是的, 实现了.

接下来看`(\_ -> lift $ Just 1)`是不是真的是`(Unit -> StateT String Maybe Int)`.

bind的签名保证了这里的`_`肯定是Unit.

所以重点是`lift $ Just 1`是不是`StateT String Maybe Int`, 那么看看lift, 它是一个类型类, 同样, 为了避免混淆, 我们修改了里面的字母:

```haskell
class MonadTrans t3 where
  lift :: forall m3 a3. Monad m3 => m3 a3 -> t3 m3 a3
```

显然, m3应该是Maybe, a3应该是Int, 那么应该会有某种MonadTrans的实现, 可以得到`lift Maybe Int -> StateT String Maybe Int`:

```haskell
instance monadTransStateT :: MonadTrans (StateT s) where
  lift m = StateT \s -> do
    x <- m
    pure $ Tuple x s
```

确实有. t3就是(StateT s).

总之, 通过神奇的类型类约束, 确保了我们可以按想象中一样使用put这样的函数.

## 异常转换器

这里: [Control.Monad.Except.Trans - purescript-transformers - Pursuit](https://pursuit.purescript.org/packages/purescript-transformers/5.1.0/docs/Control.Monad.Except.Trans#t:ExceptT).

ExceptT是一个类型, 完整写法是`ExceptT e m a`.  和`StateT s m a`很像, e是错误类型, m是包裹类型, a是结果类型.

他在monad里提供了三个函数: catchError, throwError, lift.

看一个例子:

```haskell
module Main where

import Prelude

import Control.Monad.Cont (lift)
import Control.Monad.Except (ExceptT, runExceptT)
import Control.Monad.Except.Trans (throwError)
import Data.Maybe (Maybe(..))
import Effect (Effect)
import Effect.Console (log)

val1 :: ExceptT String Maybe Int
val1 = do
  _ <- throwError "err"
  lift $ Just 1

main :: Effect Unit
main = do
  log $ show $ runExceptT val1 -- 得到(Just (Left "err"))
```

## 组合

可以玩出更多花样, 看一个例子:

```haskell
module Main where

import Prelude

import Control.Monad.Except (ExceptT, lift, runExceptT, throwError)
import Control.Monad.Writer (WriterT, runWriter, tell)
import Data.Identity (Identity)
import Effect (Effect)
import Effect.Console (log)

writerAndExceptT :: ExceptT String (WriterT (Array String) Identity) Int
writerAndExceptT = do
  lift $ tell ["1"]
  lift $ tell ["2"]
  _ <- throwError "err"
  lift $ tell ["3"]
  pure 1

main :: Effect Unit
main = do
  log $ show $ runWriter $ runExceptT writerAndExceptT
  -- 得到(Tuple (Left "err") ["1","2"])
```

只要把包装换成写monad, 就可以去操作写monad了, 只需要最后把runExceptT的结果再runWriter一次.

这就相当于你在monad里同时拥有了写monad和异常monad的能力, 既可以tell又可以throwError.

这是怎么做到的?看一个简化的例子:

```haskell
module Main where

import Prelude

import Control.Monad.Except (ExceptT, lift, runExceptT)
import Control.Monad.Writer (WriterT, runWriter, tell)
import Data.Identity (Identity)
import Effect (Effect)
import Effect.Console (log)

writerAndExceptT :: ExceptT String (WriterT (Array String) Identity) Int
writerAndExceptT = bind (lift $ tell ["1"]) (\_ -> pure 1)

main :: Effect Unit
main = do
  log $ show $ runWriter $ runExceptT writerAndExceptT
  -- 得到(Tuple (Right 1) ["1"])
```

先把do写成函数调用的形式:

```haskell
writerAndExceptT :: ExceptT String (WriterT (Array String) Identity) Int
writerAndExceptT = bind (lift $ tell ["1"]) (\_ -> pure 1)
```

参考之前的分析, 这里bind的类型应该是:

```haskell
ExceptT String (WriterT (Array String) Identity) Unit
  -> (Unit -> ExceptT String (WriterT (Array String) Identity) Int)
  -> ExceptT String (WriterT (Array String) Identity) Int
```

首先, 为什么pure 1就可以变成`ExceptT String (WriterT (Array String) Identity) Int`?

因为pure将1提升到了`ExceptT String (WriterT (Array String) Identity)`, 这不难理解.

那为什么tell前要加lift? 看一下tell的定义和WriterT对它的实现:

```haskell
class (Semigroup w, Monad m) <= MonadTell w m | m -> w where
  tell :: w -> m Unit
```

```haskell
instance monadTellWriterT :: (Monoid w, Monad m) => MonadTell w (WriterT w m) where
  tell = WriterT <<< pure <<< Tuple unit
```

在这里`tell ["1"]`的签名是`(Array String) -> (WriterT (Array String) Identity) Unit`.

但bind需要的是`ExceptT String (WriterT (Array String) Identity) Unit`, 而lift刚好可以做这件事.

所以类型对上了, 当然, 实现也如你想的那样, 你可以通过lift操作包在里面的这个WriterT.

当然, 你还可以组合更多monad:

```haskell
module Main where

import Prelude

import Control.Monad.Except (ExceptT, lift, runExceptT, throwError)
import Control.Monad.State (StateT, get, runStateT)
import Control.Monad.Writer (WriterT, runWriterT, tell)
import Data.Identity (Identity)
import Effect (Effect)
import Effect.Console (log)

val1 :: StateT String (WriterT (Array String) (ExceptT (Array String) Identity)) String
val1 = do
  s <- get
  lift $ tell ["The state is " <> s]
  _ <- lift $ lift $ throwError ["Empty string"]
  pure "nihao"

main :: Effect Unit
main = do
  log $ show $ runExceptT $ runWriterT $ runStateT val1 "test"
  -- 得到(Identity (Left ["Empty string"]))
```

你同时拥有了StateT, WriterT和ExceptT. 只是这样写比较繁琐.

幸运的是, 这些类型里互相实现了各自的函数, 比如, 对于刚才的:

```haskell
writerAndExceptT :: ExceptT String (WriterT (Array String) Identity) Int
writerAndExceptT = bind (lift $ tell ["1"]) (\_ -> pure 1)
```

把lift去掉也能用:

```haskell
writerAndExceptT :: ExceptT String (WriterT (Array String) Identity) Int
writerAndExceptT = bind (tell ["1"]) (\_ -> pure 1)
```

为什么?答案是, 在ExceptT类型里也实现了tell:

```haskell
instance monadTellExceptT :: MonadTell w m => MonadTell w (ExceptT e m) where
  tell = lift <<< tell
```

## RWS

这里: [Control.Monad.RWS - purescript-transformers - Pursuit](https://pursuit.purescript.org/packages/purescript-transformers/5.1.0/docs/Control.Monad.RWS#t:RWS).

依据上面的原理, 封装出了一个RWS单子, 当然他还有对应的转换器形式RWST.

顾名思义, 就是在这个单子里你可以自由的使用读,写,状态monad的那些函数.

它的形式是`RWST r w s m a`.

r是读monad的环境类型, w是写monad的类型, s是状态monad的类型, m是包装类型, a是结果类型.

而RWS也和之前一样, 将RWST的m设置为Identity:

```haskell
type RWS r w s = RWST r w s Identity
```

