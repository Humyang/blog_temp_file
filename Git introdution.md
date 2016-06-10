
https://git-scm.com/book/en/v2/Getting-Started-About-Version-Control

Git 是一个版本控制系统

以前除了用 Github Destop 点几下按钮 commit 一下修改之外就对 Git 了解得不多了，所以现在研究一下进阶的用法。


## commit

用得最多的就是 commit 了，那么他是如何工作的呢？

commit 记录你每次提交的变动，例如：

- 增加内容
- 删除内容
- 修改内容:先删除，后增加

所以 commit 就是一系列删除和增加操作的集合，后面的 reset，revert 就是基于这样的原理操作。


## 版本管理系统历史

### 本地版本控制系统

大多数人都是手动复制文件到其它文件夹，这是最简单的方式，也是最多人使用的方式。

它是实在太简单，只需要复制文件到不同位置即可。这也是最容易出错的方式，例如无意的覆盖文件。

为了解决这个问题，程序员开发出了本地版本控制系统，将文件所有变动写到本地数据库中，实现版本控制。

最流行的本地版本控制系统是 RCS，至今仍有许多计算机使用。甚至连现代操作系统 Mac OS X 也内置了 `rcs` 命令，如果你安装了开发中工具。

RCS 以特定格式保存文件的路径 (不同的文件之间) 在磁盘中，它可以在任何时间点重新创建文件，就像打补丁一样。

### 中心化本控制系统

解下来下一个主要问题是人们需要与其它系统共通协作。为了解决这个问题，中心化版本控制系统被开发出来了，这些系统如 CVS, Subversion, 和 Perforce 等，在一个中心服务器中包含所有版本文件，数个客户端从中心获取文件。

在许多年中，它是版本控制系统的标准化。

这种设置提供了许多好处，对比本地版本控制系统尤其明显，例如大家一定程度上知道其他人在项目中做了什么。管理员可以细粒度的控制谁可以做什么，比起要在每台机器设置的本地版本控制系统它方便并简单。

但是，它也有一些问题，最有代表性的是当中心服务器出现问题时，如果服务器发生了几个小时的故障，期间任何人都无法共同合作或者保存工作内容的版本变更到服务器。如果磁盘出现问题，并且没有适当的备份，你会丢失一切－除非某个人的电脑有项目的整个历史快照。

本地版本控制系统同样有这个问题，你把整个项目保存在一个地方，你可能会失去所有

### 分布式版本控制系统

This is where Distributed Version Control Systems (DVCSs) step in. In a DVCS (such as Git, Mercurial, Bazaar or Darcs), clients don’t just check out the latest snapshot of the files: they fully mirror the repository. Thus if any server dies, and these systems were collaborating via it, any of the client repositories can be copied back up to the server to restore it. Every clone is really a full backup of all the data.

分布式版本控制系统来了 (DVCS) ，在 DVCS 中 (例如 Git，Mercurial，Bazaar，Darcs) ，客户端不需要获取最新的文件快照：他们存储了完整的镜像库。因此如果服务器挂了，通过这些系统，任何客户端都可以将仓库复制回服务器并恢复它。每份克隆都是完整的数据备份。
