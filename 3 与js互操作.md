# 与js互操作

## js调用PureScript

### 编译为模块

```
spago build
```

- 会编译模块和依赖模块,都在out文件夹里.
- 他们会互相引用,如果要用,可以把out文件夹整个拷走.

### 打包为可执行程序

```
spago bundle-app
```

- 可以直接运行
- 会自动处理依赖关系

###  打包为模块

```
spago bundle-module
```

- 会暴露Main模块指定的内容
- 会自动处理依赖关系

## PureScript调用js

todo