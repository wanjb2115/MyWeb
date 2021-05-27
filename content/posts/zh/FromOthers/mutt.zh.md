---
title: "[转]Mutt：命令行的邮件大师"
date: 2020-02-22T19:14:44+08:00
draft: true
tags: ["转载"]
categories: ["装机必备"]
Summary: "正式的介绍和配置Mutt"
---
**转载自[正式的介绍「Mutt」：命令行的邮件大师 (一文详解)](https://segmentfault.com/a/1190000018131615)，仅供学习参考，如有侵权，请联系作者删除**

为什么要用Mutt？
这个世界已经有了成百上千的漂亮邮件客户端，为什么还要用命令行里的？
其实说什么功能都没用。说到本质上，其实是一种Geek精神，一种爱折腾的精神，一种Customizability的精神。就像明明有WhatsApp，还要用IRC一样的精神；明明有Finder，还要用Ranger的精神。
在终端里待久了，会比较烦GUI，所以不管什么软件都会寻求终端的替代方案。
对于这个需求来说，在Linux的世界里，似乎就只有一个选择：Mutt。

Mutt的可配置性，强如Vim。配置起来也和Vim差不多，有专门的~/.muttrc供你配置软件本身。

需要理解的是：Mutt本身是一个框架而已。收件、发件、编辑邮件等功能，是要通过搭配不同的程序来做到的。

## Mutt的模块搭配方案
**mutt + fetchmail + msmtp + procmail**

[参考Arch Wiki：Mutt (极详细，但对人类不友好)](https://wiki.archlinux.org/index.php/mutt)

安装：
```bash
# Mac
$ brew install mutt fetchmail msmtp procmail

# Ubuntu
$ sudo apt-get install mutt fetchmail msmtp procmail -y
```
Mutt或各个写协作程序在配置前都是不能使用的，学习曲线还是比较陡峭的，所以要做好准备去花好一段去了解和学习各个部件。

大概的配置流程是：
- 配置收件：~/.fetchmailrc
- 配置过滤：~/.procmailrc
- 配置发件：~/.msmtprc
- 配置mutt框架本身：~/.muttrc

注意：初学过程中，不要一上来就配置mutt。最好是先从各个部件开始：收件->过滤邮件->阅读邮件->发件->mutt界面，按照这种顺序。

## 收件：配置Fetchmail
>Fetchmail是由著名的《大教堂与集市》作者 Eric Steven Raymond 编写的。

Fetchmail是一个非常简单的收件程序，而且是前台运行、一次性运行的，意思是：你每次手动执行fetchmail命令，都是在前台一次收取完，程序就自动退出了，不是像一般邮件客户端一直在后台运行。

**注意：fetchmail只负责收件，而不负责存储！所以它是要调用另一个程序如procmail来进行存储的。**

fetchmail的配置文件为~/.fetchmailrc。然后文件权限最少要设置chmod 600 ~/.fetchmailrc

[参考：Using Fetchmail to Retrieve Email](https://www.linode.com/docs/email/clients/using-fetchmail-to-retrieve-email/)

比如我们要设置多个邮箱账户同时收取，那么配置如下：
```bash
poll pop.AAA.com protocol POP3 user "me@AAA.com" password "123"
poll pop.BBB.com protocol POP3 user "me" there with password "123" is falko here fetchall
poll pop.CCC.com protocol POP3 user "me" there with password "123" is till here keep
poll pop.DDD.com
  protocol POP3
  user "me"
  password "123"

# 全局选项
mimedecode
mda "/usr/local/bin/procmail"
```
其中，
- 各种参数可以不按顺序，也可以不在一行。 空格隔开每个参数，poll隔开每个账户。
- here, there, with, is等等，都不是关键词，随便写不影响参数。
- poll后面是邮件服务器的地址，一般是pop.xxx.com
- protocol后面是收件协议，一般是pop或pop3
- user后面是用户名，可以是username，也可以是邮箱地址
- password后面是密码
- 以上是必填，其它都不是必要的
- 四选一：fetchall, nofetchall, keep, nokeep
- mimedecode用来自动解码MIME
- mda后面指定本机安装的邮件过滤分类程序。如果不填，则收取的邮件在本地不会保存。可以用$ which procmail查一下路径。

配置完成后，可以运行fetcmail -v来看看是否有错误信息，如果能够正常显示很多行的收取信息，那么就能正确登录邮箱收取了。

一般收取的命令如下：
```bash
# 只收取未读邮件
$ fetchmai

# 收取所有邮件
$ fetchmail -a

# （重要）收取新邮件，但不在服务器端删除已经收取的邮件
$ fetchmail -k
```
但是fetchmail只负责收取，不负责“下载”部分，你找不到邮件存在哪了。
所以还需要配置MDA分类器，如procmail，才能看到下载后的邮件。

>注意：Fetch其实不是在Mutt“里”使用的，而是脱离mutt之外的！也就是说，Mutt只负责读取本地存储邮件的文件夹更新，而不会自动帮你去执行fetchmail命令。

你必须自己手动执行，或者用Crontab定期收取，或者设为Daemon守护进程，还可以在Mutt中设置快捷键执行Shell命令。

设置Mutt快捷键收取邮件的方法是在~/.muttrc中加入macro：
```bash
macro index,pager I '<shell-escape> fetchmail -vk<enter>'
```

这样的话，你就可以在index邮件列表中按I执行外部shell命令收取邮件了。

## 邮件过滤：配置Procmail
Procmail是单纯负责邮件的存储、过滤和分类的，一般配合fetchmail收件使用。

在Pipline中，fetchmail把收到的邮件全部传送到Procmail进行过滤筛选处理，然后Procmail就会把邮件存到本地形成文件，然后给邮件分类为工作、生活、重要、垃圾等。

当然，分类规则是自己可以指定的。可以根据发信人、主题、长度以及关键字 等对邮件进行排序、分类、整理。

[参考：Procmail](http://lifegoo.pluskid.org/wiki/Procmail.html)

Procmail 的配置文件是 ~/.procmailrc ，记得改权限：chmod 600 ~/.procmailrc。
内容也非常简单，前面是邮件位置、日志等默认选项，后面则是一块一块的过滤规则。

基本配置：
```bash
MAILDIR=$HOME/Mail   #邮件存储地址
DEFAULT=$MAILDIR/inbox   #默认：收件箱
VERBOSE=off
LOGFILE=/tmp/procmaillog

# 某个垃圾邮件规则
:0
* ^From: webmaster@st\.zju\.edu\.cn
/dev/null    #垃圾文件的存储位置

# 其它所有都存到收件箱中
:0:
inbox/
```
其中，$HOME/Mail是设定的邮件存储位置。
我们需要手动创建mkdir ~/Mail，否则程序会报错。

配置好后，我们再测试一下，假设邮箱里有一封未读邮件，就会看到:
```bash
$ fetchmail
1 message for Jason@aliyun.com at pop3.aliyun.com (7833 octets).
reading message Jason@aliyun.com@pop3.aliyun.com:1 of 1 (7833 octets) flushed

$ tree ~/Mail
/Users/Jason/Mail
└── inbox
    ├── cur
    ├── new
    │   └── 1549706227.89809_0.Jason-mba.lan
    └── tmp
```

## 发件：配置msmtp

msmtp是作为替代sendmail发邮件程序的更好替代品。
msmtp的配置文件为~/.msmtprc，记得改权限：chmod 600 ~/.msmtprc
配置内容比收件还简单，因为发件永远比收件简单。

>Tip: 发件的服务器是smtp协议。收件才是pop3协议。

基本配置：
```bash
account default
  auth login
  host smtp.XXX.com
  port 587
  from ME@XXX.com
  user ME
  password passwd
  tls on
  tls_certcheck off

logfile /tmp/msmtp.log

```

其中注意，关于tls，如果是阿里云则不用写，如果是Outlook的话，必须写：
```bash
    tls on
    tls_starttls on
    tls_certcheck off
```

## 主界面：配置Mutt
Mutt的配置文件为~/.muttrc，记得改权限：chmod 600 ~/.muttrc

>另外：mutt的配置文件还可以放在~/.mutt/muttrc。这种方法有一个好处，即~/.mutt/目录下可以放很多主题、插件等文件。

基本配置：
```bash
# 通用设定
set use_from=yes
set envelope_from=yes
set move=yes    #移动已读邮件
set include #回复的时候调用原文
set charset="utf-8"
auto_view text/html   #自动显示HTML

# 发送者账号
set realname="Solomon Xie"
set from="solomonxie@aliyun.com"

# 分类邮箱
set mbox_type = Maildir #Mail box type
set folder = "$HOME/Mail"
set spoolfile = "$HOME/Mail/inbox" #INBOX
set mbox="$HOME/Mail/seen"  #Seen box
set record="$HOME/Mail/sent"  #Sent box
set postponed="$HOME/Mail/draft"  #Draft box

# 关联程序（需要自己用which命令确定一下）
set editor="vim -nw"
set sendmail="/usr/local/bin/msmtp"
```

## 确认邮箱服务器有没有问题

即使上面配置一切OK，也不一定能正常收发邮件。因为你用的Gmail、QQ、网易、阿里云等等，后台都有一系列的第三方收取设置。这是各不相同的。

比如QQ和网易，现在几乎已经不能用了（2019），为什么？因为它们完全阻止了第三方客户端收发件。即使你去后台设置面板，可以通过手机短信验证之类设置，但是会发现实际上总是验证不了总是通过不了。所以本质上，他们只允许自己的官方客户端不允许任何别的手机、PC客户端（流氓行径）

Gmail在国内用不了众所周知。现在比较好用的只有阿里云和微软的Outlook了。

除了第三方客户端的允许，我们还要设置POP。最好放开全部邮件或者最近30天，然后禁止客户端删信。这是什么意思呢？POP默认客户端在收件后，服务器上的邮件就自动删除了！这个不太合适，所以必须要禁止

## Mutt主界面的基本操作

### 邮件列表操作：

- 基本：q:Quit, d:删除当前邮件, s:将邮件移动至指定文件夹, m:创建新邮件, r:回复当前邮件, ?:帮助
- 移动：j/k 上下移动邮件, z/Z上下翻页, <Number> 跳至序号处（不进入邮件）
- <Enter> 打开选中的邮件
- /在当前文件夹搜索
- d 将选中邮件标记为删除, N 将选中邮件标记为未读, $ 让标记的东西生效，如删除、未读等。
- f 转发选中邮件, e 编辑选中邮件
- c切换文件夹(inbox/seen/draft等), 需要输入文件夹名称，或按?在列表里选择，j/k上下移动。

### 在邮件中的操作：

- j/k 上一封／下一封邮件, <Space>: 向下翻页, <Enter>: 向下滚动
- e 编辑当前邮件, t编辑TO，c编辑CC，b编辑BCC，y发送邮件，a添加附件，Return查看附件，E编辑附件，D删除附件

### 使用命令操作

Mutt如同Vim一样，不光可以把命令绑定为快捷键，还能直接输入:直接输入命令。
但是稍有不同的是，Mutt称之为Action，而且需要用:exec <命令>这样格式执行。

比如sidebar侧边栏的移动，命令是：sidebar-next, sidebar-prev。
那么我们可以直接输入:exec sidebar-next，按下回车执行。

## Mutt乱码问题
locale大法：
在命令行里分别输入$ locale命令，查看Shell中的语言设置:
```bash
↳ $ locale
LANG=en_GB.UTF-8
LANGUAGE=
LC_CTYPE=en_GB.UTF-8
LC_NUMERIC="en_GB.UTF-8"
LC_TIME="en_GB.UTF-8"
LC_COLLATE="en_GB.UTF-8"
LC_MONETARY="en_GB.UTF-8"
LC_MESSAGES="en_GB.UTF-8"
LC_PAPER="en_GB.UTF-8"
LC_NAME="en_GB.UTF-8"
LC_ADDRESS="en_GB.UTF-8"
LC_TELEPHONE="en_GB.UTF-8"
LC_MEASUREMENT="en_GB.UTF-8"
LC_IDENTIFICATION="en_GB.UTF-8"
LC_ALL=
```
也许问题就出在这里：Shell的设置出了问题，而不是mutt的设置！

解决方法很简单：

```bash
export LANG=en_US.UTF-8
```

然后再输入locale命令就可以看到正常的语言编码设置了。
再打开mutt也是正常显示。

但是直接这样export是临时的，需要把这个加入到~/.zshrc或~/.bash_profile中。