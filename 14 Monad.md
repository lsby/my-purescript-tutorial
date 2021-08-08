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

## 引用

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

// todo 也许后面还有

