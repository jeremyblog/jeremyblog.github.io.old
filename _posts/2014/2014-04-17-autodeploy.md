---
layout: post
title: "利用git hooks实现自动部署"
date: 2014-04-17T23:30:00-08:00
description: 基于git实现的自动化部署方案
keywords: git,hooks,自动化,部署,脚本
categories:
  -- recent
  -- it
---

 和大多数优秀的开源项目一样，git让用户可以编写脚本来实现功能的扩展，那就是hooks。在git目录下的hooks目录，内置一些shell和perl脚本编写的样本，但是git同时支持golang、ruby、python等脚本作为钩子。为了不让git去执行它，这些文档都以.sample结尾，同样的为了启动这些钩子，你需要去掉文件后缀的.sample，并使用chmod
 +x 让文件成为可执行文档。

 git支持许多的钩子，这里只介绍原型可能涉及的客户端的钩子。

 **提交工作流程钩子**

 有四个钩子处理提交的流程。最新执行的是pre-commit钩子，它用来检查提交的快照，钩子返回非零值时，git放弃此次提交(可使用git
 commit --no-verify来忽略)。

 接下来是prepare-commit-msg钩子，在信息编辑器显示之前被执行，信息编辑器一般显示默认的信息，比如邮箱、用户、SHA-1编码等，然后要求用户输入此次关于快照的描述信息等。钩子的用处仅是修改这些默认的信息，一般作用不大。

 随后是commit-msg钩子，其返回非零值时，git将放弃提交。钩子用来修改用户在编辑器中输入的描述信息。

 最后是post-commit钩子，用来在提交完成时实现通知的功能。

 提交工作流程的钩子一般适合开发者向服务器提交代码时实现自定义的扩展，比如执行本地的单元测试，或者checkstyle等，一般将这些钩子放在源码管控中以供其它开发者使用。

 值得提醒的是，这些钩子在clone时并不会被触发。

 **其它客户端钩子**

 pre-rebase在rebase版本之前执行，同样以非零值终止代码合并。

 在git
 checkout成功执行后，post-checkout钩子会被执行。这个钩子非常有用，你可以利用这个钩子将一些大文件和自动产生的文档，以及其它不想纳入版本控制的文档在此时移进你的工作目录。

 最后是在merge命令后，post-merge钩子，它也可以使用post-checkout的作用，但是这里利用它来实现自动部署。

更多关于钩子的介绍，请查看[文档][]

## 一个例子

在github上创建一个项目，然后利用代理服务可以连接外网的优势，脚本实现在这台服务器获取、合并代码，并利用maven实现自动化测试、打包等，顺利通过后利用ssh部署至内网的目标机器上。

**以下为一些必备的环境准备：**

首先，我下载了apache-maven的安装包，由于项目中使用了一些内部的依赖，所以我干脆将本地的.m2目录一并复制至代理机器上。

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
 小提示：

 如果你和我一样，拷贝了.m2目录，建议下面的构建过程中使用mvn
 -o参数，实现离线构建的功能，以避免maven下载更多的其它依赖组件。
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
接着在代理服务器上，创建一个和github同名的本地仓库，初始化时有一点可以分享，就是--bare。对于分布式开发的服务器端版本库，要求使用git
init
--bare方式初始化，原因是非bare方式的版本库会在并发提交时造成的版本遗漏等问题。但是bare方式的仓库仅有.git目录，没有工作目录，所以不能使用git的许多指令，比如merge、rebase等，非常不方便。客户端的版本库建议使用非bare方式初始化。

完整代码

    git config --global user.email "guo-hua.huang@foxconn.com"
    git config --global user.name "robot"   mkdir cqjw.mobile
    cd cqjw.mobile/
    git init
    git remote add origin ssh://robot@10.148.73.13:29418/cqjw.mobile.git
    git fetch orgin

源码采用的是ruby语言，要求安装ruby环境

    sudo apt-get install ruby

最后还要求tomcat、ssh等不再叙述。

[源码][]参考

## 关于时间控制条

为了有较好的效果，如下图

![][]

我加入了时间控制条的实现，原理非常简单:利用'\\r'的字符，将光标移到行首，不换行的前提下每隔一定时间比如1秒，不断刷新打印的信息即可。我实现的ruby代码如下

![][1]

我创建了两个线程，一个线程执行任务，另一个打印时间进度。由于ruby的特性，mytask的线程与print的线程任一线程停止时，另一线程将一同停止，所以我估算了mytask可能执行的最大时间300s，故设置print执行的时间为300s，然后在print线程利用上述'\\r'的特性，每隔3s打印一次进度信息(也就是执行时间/总时间的百分比)。为了避免mytask执行完毕，而print进程还继续执行，我在print线程中，时时判断mytask是否执行完毕(alive代表还没执行完)，完成则显示100%，然后退出。

另外，代码中还使用了线程变量Thread.current["mvn\_msg"]，来实现线程与线程间的数据共享，这个特性比较类似map类型的操作。

    printf "\r%s\n",'打包model完成100%,执行结果为:' + mytask["mvn_msg"]

  [文档]: http://git-scm.com/book/en/Customizing-Git-Git-Hooks
  [源码]: https://github.com/51write/autodeploy.git
  []: /images/2014/progressbar.jpg
  [1]: /images/2014/barCode.jpg
