# yarn vs npm

今天想对比一下node的两个JavaScript包管理器 yarn 和 npm.

到目前为止本人一直使用的是npm. 今天上github看了一下, npm star数1w4, yarn star数2w8, 开始对yarn有好感.

接下来我们对比一下二者:

## yarn.lock文件
`package.json` 是 yarn 和 npm 共同的维护项目的依赖的文件, 但 `package.json` 可以定义一个版本区间, 所以版本会不准确, yarn 引入 lock文件. 该文件会记录下精确的版本号. 保证不同机器安装的版本号是一致的.

## 并行安装
npm 是顺序安装包, yarn 是并行安装包.

## 关于安装速度可以跑一下下面的shell
```shell
#!/bin/bash
YARN_START=$(date +%s)
yarn add webpack
YARN_END=$(date +%s)
YARN_DIFF=$(( $YARN_END - $YARN_START ))
echo "yarn took $YARN_DIFF seconds"

rm -rf node_modules

NPM_START=$(date +%s)
npm i -S webpack
NPM_END=$(date +%s)
NPM_DIFF=$(( $NPM_END - $NPM_START ))
echo "npm took $NPM_DIFF seconds"
```

个人测试: 安装webpack, npm 67s, yarn 28s.

## 介绍一下yarn的命令

### yarn init
对应 `npm init`, 一样拥有有 `-y` 参数

### yarn
对应 `npm install`

### yarn remove [package]
对应 `npm uninstall --save [package]`

### yarn add [package]
对应 `npm install --save [package]`

### yarn add [package] --dev
对应 `npm install --save-dev [package]`

### yarn why [package]
显示安装包依赖的大小



## 国内使用 yarn
```shell
npm install -g yarn
yarn config set registry https://registry.npm.taobao.org
```

而 npm 有 nrm 工具可以自由切换包安装源.


