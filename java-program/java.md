---
title: Java
date: '2019-02-13T09:48:14.000Z'
tags:
  - Java
---

# Java

## 安装 JDK 环境

### 在 Mac 下

```text
# 安装 JDK 8
brew tap caskroom/versions
brew cask install java8

# 安装最新版本
brew cask install java

# 使用 jenv 管理 JDK 版本
brew install jenv
ls -1 /Library/Java/JavaVirtualMachines 
mkdir ~/.jenv/versions
jenv add /Library/Java/JavaVirtualMachines/jdk1.8.0_202.jdk/Contents/Home/
jenv versions
jenv global 1.8
java -version

# 检测 jenv 是否正常安装
jenv doctor

# 在 zsh 中启用
vim ~/.zshrc
## 在末尾追加
export JENV_ROOT="/usr/local/opt/jenv"
if which jenv > /dev/null; then eval "$(jenv init -)"; fi
## 使其生效
source ~/.zshrc

# 设置 Java Home
jenv enable-plugin export
exec $SHELL -l
echo ${JAVA_HOME}
```

## What is the Limit to the Number of Threads You Can Create?

The time it takes to create a thread increases as you create more thread. For the 32-bit JVM, the stack size appears to limit the number of threads you can create. This may be due to the limited address space. In any case, the memory used by each thread's stack add up. If you have a stack of 128KB and you have 20K threads it will use 2.5 GB of virtual memory.

| Bitness | Stack Size | Max threads |
| :--- | :--- | :--- |
| 32-bit | 64K | 32,073 |
| 32-bit | 128K | 20,549 |
| 32-bit | 256K | 11,216 |
| 64-bit | 64K | stack too small |
| 64-bit | 128K | 32,072 |
| 64-bit | 512K | 32,072 |

参考：[Java: What is the Limit to the Number of Threads You Can Create?](https://dzone.com/articles/java-what-limit-number-threads)

## 更新 Amazon Linux JDK 从 7 到 8

```text
sudo yum install java-1.8.0
sudo yum remove java-1.7.0-openjdk
sudo yum install java-1.8.0-devel
```

