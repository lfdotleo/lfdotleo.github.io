---
title: "Big Sur 使用 Jitouch"
date: 2021-10-19T18:15:33+08:00
draft: false
hideToc: false
enableToc: true
enableTocContent: true
author: dotleo
tags:
- sleepwatcher
categories:
- software

---

没有 Jitouch 的 Mac 没有灵魂，试过多个触摸板增强工具，还是 Jitouch 最舒服，但作为一个很多年不维护的软件，在 Big Sur 上还是需要一些操作才可以稳定运行。

## Jitouch 安装

建议还是 brew 安装。

```
brew install jitouch
```

## 解决 Jitouch 无法启动

直接打开 Jitouch 是无法使用的，可以将下面命令写成一个 shell 脚本，运行启动

```
pid=`ps -ef | grep Jitouch | grep -v 'grep' |  awk '{print $2}'`

if [ -n "$pid" ]
then 
	kill $pid
fi

nohup ~/Library/PreferencePanes/Jitouch.prefPane/Contents/Resources/Jitouch.app/Contents/MacOS/Jitouch > /dev/null 2>&1 &
```

## 解决 Jitouch 休眠唤醒后无法使用

Mac 休眠唤醒后，Jitouch 又不能使用了。这时候需要一个监听唤醒时间的软件，在系统唤醒后执行上面的脚本。推荐使用 sleepwatcher

### 1. 安装 sleepwatcher

```
brew install sleepwatcher
```

### 2. 设置为开机自启动

```
brew services start sleepwatcher
```

### 3. 查看进程是否启动

```
-> ps -ef | grep sleepwatch
502  2132     1   0 四07下午 ??         1:49.69 /usr/local/opt/sleepwatcher/sbin/sleepwatcher -V -s /Users/liufei234/.sleep -w /Users/liufei234/.wakeup
```

### 4. 创建脚本文件

sleepwatcher 执行的是 ~/.sleep 和 ~/.wakeup 文件，前者是睡眠时执行，后者是唤醒时执行。在 home 目录下创建文件 .wakeup 并赋予权限 777

```
touch ~/.wakeup
chmod 777 ~/.wakeup
```
### 5. 编写脚本

脚本还是刚才的启动脚本，如下：

```
pid=`ps -ef | grep Jitouch | grep -v 'grep' |  awk '{print $2}'`

if [ -n "$pid" ]
then 
	kill $pid
fi

nohup ~/Library/PreferencePanes/Jitouch.prefPane/Contents/Resources/Jitouch.app/Contents/MacOS/Jitouch > /dev/null 2>&1 &
```

这样就可以使用了，每次睡眠唤醒后就可以使用了。

## 仍然存在的问题

每次重启后，还是无法使用，需要手动执行启动脚本。但谁的 Mac 又时常重启呢？当然实在没法忍受，可以看看 [Multitouch](https://multitouch.app/)，这个软件收费，60 多吧。

反正我还是选择顺手的 Jitouch 吧。

