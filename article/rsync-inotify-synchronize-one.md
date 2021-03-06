# rsync+inotify实现文件实时同步（一）

## rsync简介

rsync是一款开源、快速的、多功能的、可实现全量及增量的本地或远程数据同步的优秀的工具，rsync软件适用于unix/linux/windows等多种操作系统平台。

rsync英文全称为Remote Rynchronization，从软件的名称可以看出来，rsync具有可使本地和远程两台主机之间的数据快速复制同步镜像、远程备份的功能，这个功能类似ssh带的scp命令。当然，rsync还可以在本地主机的不同分区或目录之间全量及增量的复制数据，这又类似cp命令，但同样也优于cp命令，cp每次都是全量拷贝，而rsync可以增量拷贝。

**小提示：**利用rsync还可以实现删除文件和目录的功能，这有相当于rm命令.

一个rsync相当于scp,cp,rm但是还优于他们每一个命令。

在同步备份数据时，默认情况下rsync通过其独特的"quick check"算法，它仅同步大小或者最后修改时间发生变化的文件或目录，当然也可根据权限、属主等属性的变化同步。

**提示：**传统的cp，scp工具拷贝每次均为完整的拷贝，而rsync除了可以完整拷贝外，还具备增量拷贝的功能，应此，从同步数据的性能及效率上，rsync工具更胜一筹。

## rsync的特性

rsync的特性如下：

1. 支持拷贝特殊文件如链接文件，设备等；
2. 可以有排除指定文件或目录同步的功能，相当于打包命令tar的排除功能；
3. 可以做到保持原文件或目录的权限、时间、软硬链接、属主、组等所有属性均不改变； -p
4. 可实现增量同步，即只同步发生变化的数据，因此数据传输效率很高；
5. 可以使用rcp,rsh,ssh等方式来配合传输文件（rsync本身不对数据加密）；
6. 可以通过socket(进程方式)传输文件和数据(服务端和客户顿)；
7. 支持匿名的或认证（无需系统用户）的进程模式传输，可实现方便安全的进行数据备份及镜像。

## rsync企业工作场景说明

1. 两台服务器之间数据同步
2. 把所有客户服务器数据同步到备份服务器
3. rsync结合inotify的功能做实时的数据同步

### 生产场景集群架构服务器备份方案

1. 全网服务器数据备份解决方案提出及负责实施 200x.03 – 200x.09
2. 针对公司重要数据备份混乱状态和领导提出备份全网数据的解决方案
3. 通过本地打包备份，然后rsync结合inotify应用把全网数据统一备份到一个固定存储服务器，然后在存储服务器上通过脚本检查并报警管理员备份结果。
4. 定期将IDC机房的数据备份到公司的内部服务器，防止机房地震及火灾问题导致数据丢失


## rsync的工作方式

### rsync大致的三种传输方式

1. 单个主机本地之间的数据传输（此时类似于cp命令的功能）；
2. 借助rcp,ssh等通道来传输数据（此时类似于scp命令的功能）；
3. 此守护进程（socket）的方式传输数据（这个是rsync自身的重要功能）

### 本地数据传输模式（local-only mode）

rsync本地传输模式的语法为：

    rsync [OPTION...] SRC... [DEST]

语法说明：

1. rsync为同步命令；
2. `[OPTION…]`为同步时的桉树选项；
3. SRC为源，即待拷的分区、文件或目录等；
4. `[DEST]`为目的的分区、文件或目录等；


把/opt目录拷贝到/mnt下

```bash
[root@ansheng /]# rsync -avz /opt /mnt/
```

### 通过远程shell进行数据传输（remote shell mode）

通过远程shell(rcp、ssh等)传输可以分为两种情况，其语法分为

```bash
 Access via remote shell:
        拉取： Pull: rsync [OPTION...] [USER@]HOST:SRC... [DEST]
        推送： Push: rsync [OPTION...] SRC... [USER@]HOST:DEST
```

**提示：**此方式一般是配合SSH Key免密钥登录来完成数据同步的

**语法说明**

1. rsync为同步的命令；
2. `[OPTION…]`为同步时的参数选项；
3. `[USER@] HOST…`为rsync同步的远程的连接用户和主机地址；
4. SRC为源，即待拷的分区、文件或目录等，和HOST之间用一个冒号链接；
5. `[DEST]`为目的的分区、文件或目录等；

其中拉去（get），表示从远端主机把数据同步到执行命令的本地主机相应目录；推送（put）表示从本地主机执行命令把本地的数据从不到远端主机指定目录下。

拉取实例语法：

```bash
rsync -vzrtopgP -e 'ssh -p 22' root@10.10.10.10:/opt/ /tmp
```

推送实例语法：

```bash
rsync -vzrtopgP -e 'ssh -p 22' /tmp root@10.10.10.10:/opt/
```
关键语法说明：

1. `–vzrtopgP`相当于上文的-avz表示同步时文件和目录属性不变；
2. `–progress`显示同步的过程，可以用-P替换；
3. `–e 'ssh –p 22'`,表示通过ssh的通道传输数据，-p 22可省略；
4. `root@10.10.10.10:/opt`远程的主机用户，地址 ，路径
5. `/tmp`为本地目录

### 使用守护进程的方式数据传输（daemon mode）


语法说明：

```bash
Access via rsync daemon:
	Pull: rsync [OPTION...] [USER@]HOST::SRC... [DEST]
	rsync [OPTION...] rsync://[USER@]HOST[:PORT]/SRC... [DEST]
	Push: rsync [OPTION...] SRC... [USER@]HOST::DEST
	rsync [OPTION...] SRC... rsync://[USER@]HOST[:PORT]/DEST
```

## rsync命令同步参数选项

**语法：**

 `rsync [OPTION...] SRC... [DEST]`

**参数：**

|参数|说明|
|:--|:--|
|-v|详细模式输出，传输时的进度等信息|
|-z|传输时进行压缩以提高传输效率，--compress-level=NUM可按级别压缩|
|-a|归档模式，表示以递归方式传输文件，并保持所有文件属性，等同于-rtopgDl|
|-r|对子目录以递归模式，即目录下的所有目录都同样传输，注意是小写r|
|-t|保持文件时间属性|
|-o|保持文件属主信息|
|-P|显示同步的过程及传输时的进度等信息|
|-D|保持设备文件信息|
|-l|保持软连接|
|-e|使用的信道协议，指定替代rsh的shell程序。例如：ssh|
|--exclude=PATTERN|指定排除不需要传输的文件模式|
|--exclude-form=file|排除指定文件里的文件或目录|

### 保持同步目录及文件属性

这里的-avzP相当于-vzrtopgDIP（还多了DI功能），生产环境常用的参数选项为-avzP或-vzrtopgP如果放入脚本中，也可以把-v和-P去掉，这里的—progress可以用-P代替

## rsync+inotify组合的起源

rsync(remote sync)远程同步工具，通过rsync可以实现对远程服务器数据的增量备份同步，但rsync自身也有瓶颈，同步数据时，rsync采用核心算法对远程服务器的目标文件进行对比，只进行差异同步，我们可以想象一下，如果服务器的文件数量达到了百万甚至千万级，那么文件对比将是非常耗时的。而且发生变化的往往是其中很少的一部分，这是非常低效的方式，inotify的出现，可以缓解rsync不足之处，取长补短。

## inotify简介

inotify是一种强大的、细粒度的、异步的文件按系统时间监控机制，Linux内核从2.6.13起，加入inotify支持，通过inotify可以监控文件系统中添加、修改、删除、移动等各种事件，利用这个内核接口，第三方软件就可以监控文件系统下文件的各种变化情况，而inotify-tools正式实施这样监控的软件。sersync