---
layout: post
title: 	如何向openstack社区提交代码
description: "openstack"
tags: [OpenStack]
categories: [OpenStack]
---



有幸参加OpenStack bug smash活动，对“我的第一个patch”做了一下总结，写了如下教程  

![image](/images/openstack-bug-fix/4.png)




## 前期准备
* 创建一个 Launchpad（ https://launchpad.net/openstack ）账号，加入OpenStack社区。
* 在（ https://www.openstack.org/profile ）上注册账号（这里的账号与1.中的账号，邮箱应该一致），成为Foundation Member（否则后面提交会出现问题）。
* 进入（ https://review.openstack.org ），登陆。
* 进入（ https://review.openstack.org/#/settings/ ）在里面填写如下信息： 
	*  在Profile中的Username 。
	* 在Agreements中签署协议（个人是ICLA）。 
	* 在Contact Infomation中填写所有内容，注意如果之前不是Foundation Member就会出现无法提交问题。
	* 在HTTP Password中Generate Password，生成一串代码。后续提交代码时需要用到这串密码。
* 获取所参与的工程的代码（此处以openstackclient项目为例，不同的项目有不同的路径）：
	$git clone http://git.openstack.org/openstack/python-openstackclient.git
	之后进入项目目录：$cd python-openstackclient

##  配置git和git-review
* 安装git-review

```
	$sudo apt-get install git-review
```

* 配置

```
	$ git config gitreview.username xxxxxxx <= Gerrit登录的username
	$ git config user.name "xxxxxx"          　 <= Gerrit登录的Full Name
	$ git config user.email "xxxxxx"        <= Gerrit登录的邮件地址
	默认是使用ssh方式，（建议）用如下方法变更为https方式。
	$ git config gitreview.scheme https
	$ git config gitreview.port 443
	$ git remote add gerrit https://gerrit-username:http- password@review.openstack.org:443/openstack/python-openstackclient.git
 	(链接中的 gerrit-username改为在Gerrit中的username, http-password改为第一章4.4步骤中所获取的http密码，链接最后的python-openstackclient.git为项目名称，这里以python-openstackclient项目为例）
	完成之后执行命令：
	$ git review -s -v
   （HTTP 密码不能要有“\”符号）
	如果这里报错没有.git/hooks/commit-msg文件，从https://review.openstack.org/tools/hooks/commit-msg获取commit-msg文件并且放置在.git/hooks/目录下，然后再执行一次git review -s -v命令。
```


## 修改并提交代码
* 创建分支

```
	参照Launchpad上Bug的编号，根据bug/${bug-number}的命名规则创建并且切换Branch。
	$ git checkout -b bug/123456789
```

* 修复bug

```
	对相应的文件进行修改，修复这个bug。
```

* 提交

```
	$ git add 将要提交的文件名
	$ git commit 将要提交的文件名
```
	
执行了以上命令之后会启动编辑器，进行提交信息的填写。提交信息的填写规范如下：  
第一行：标题，概括你此次提交代码的功能或者目的。  
第二行：换行。	  
第三行以及之后：具体地说明提交的内容、功能、目的等。	  
倒数第二行：换行。  
最后一行：Closes-Bug:#xxx或者Partial-Bug: #xxx（其中xxx为bug的编号）。  

* 提交完之后可以用git log命令看到你提交的信息


```
	$ git log
```

log中最上面的的一条commit即是最新的commit，注意看看刚刚所提交的commit有没有change-id,如果没有的话之后会提交失败，可能是配置git review的时候缺少commit-msg文件的问题（见第二部分中的2.配置）。


* 执行完以上命令后执行git review完成提交

```
	$ git review
```


执行成功后会出现Review: https://review.openstack.org/xxxxx（其中xxxxx是数字）。这里的Review:的URL就是Gerrit的URL。相关的测试将自动被实施，从Zuul Status可以看到自己的测试的Status。通过此页面可以用自己的Gerrit的ID来检索。

##  评审和接受


测试通过的话，各个工程的Core Developer会进行代码评审。如果有两名Core Developer分别进行了+1操作，代码就会被合并。
如果被指出有问题的话，修改后执行以下命令再次实施测试。然后，务必在Gerrit的Reply处对指正的人表示感谢。  

```
	$ git add --all
	$ git commit --amend
	再次执行git review。
	$ git review
	之后等待修改被合并即可。
```

## 补充：git的其他相关功能


* git 制作patch
在commit 之后，使用命令  

```
	git format-patch -n（其中n表示patch的数量，一个patch对应一个commit）
```

* git send email

```
	安装：sudo apt-get install git-email
	配置：打开～/.gitconfig文件，写入以下内容
	[sendemail] 
     	smtpserver = smtp.ym.163.com 
        	smtpuser = jqjiang@bnc.org.cn 
        	smtpserverport = 25 
	发送email:
	git send-email 要发送的文件
	之后会提示你输入收信人的email地址和你的email密码等，按提示正确输入即可。

```

---

##  有用的git命令：




创建并切到一个topic:  


```
git checkout -b trivial
```

查看自己的分支：  

```
root@devstack:/home/python-openstackclient/doc/source# git branch
  master
* trivial

```


查看自己的改动：  

```
root@devstack:/home/python-openstackclient/doc/source# git diff
diff --git a/doc/source/developing.rst b/doc/source/developing.rst
index 399e4a5..cf92661 100644
--- a/doc/source/developing.rst
+++ b/doc/source/developing.rst
@@ -103,7 +103,7 @@ only want to run the test that hits your breakpoint:

 .. code-block:: bash

-    $ tox -e debug opentackclient.tests.identity.v3.test_group
+    $ tox -e debug openstackclient.tests.identity.v3.test_group

 For reference, the `debug`_ ``tox`` environment implements the instructions
```



查看自己改动哪些文件:  

```
root@devstack:/home/python-openstackclient/doc/source# git status
On branch trivial
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git checkout -- <file>..." to discard changes in working directory)

        modified:   commands.rst
        modified:   developing.rst

no changes added to commit (use "git add" and/or "git commit -a")


```

commit代码：  

```
root@devstack:/home/python-openstackclient/doc/source# git commit -a
```


查看commit之后的状态(注意颜色变化)：  

```
root@devstack:/home/python-openstackclient/doc/source# git status
On branch trivial
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git checkout -- <file>..." to discard changes in working directory)

        modified:   commands.rst
        modified:   developing.rst

no changes added to commit (use "git add" and/or "git commit -a")
root@devstack:/home/python-openstackclient/doc/source# ^C
root@devstack:/home/python-openstackclient/doc/source# git add --all
root@devstack:/home/python-openstackclient/doc/source# ^C
root@devstack:/home/python-openstackclient/doc/source# git status
On branch trivial
Changes to be committed:
  (use "git reset HEAD <file>..." to unstage)

        modified:   commands.rst
        modified:   developing.rst

```



进入界面：

![image](/images/openstack-bug-fix/1.png)


格式如下：  

```
fix some spelling mistakes in doc/

I check all the files under doc/ directory and find three
spelling mistakes
 - exeuction should be execution
 - Fefora should be Fedora
 - opentackclient should be openstackclient

# Please enter the commit message for your changes. Lines starting
# with '#' will be ignored, and an empty message aborts the commit.
# On branch trivial
# Changes to be committed:
#       modified:   commands.rst
#       modified:   developing.rst
```


然后退出编辑，执行git commit -a  

```
root@devstack:/home/python-openstackclient/doc/source# git commit -a
[trivial 1b05a6d] fix some spelling mistakes in doc/
 2 files changed, 3 insertions(+), 3 deletions(-)
```



git log可以看到自己的commit，还有唯一的ID  

```
root@devstack:/home/python-openstackclient/doc/source# git log
commit 1b05a6dff9faaabac030794423d781b647818f0a
Author: jqjiang.1@gmail.com <jqjiang.1@gmail.com>
Date:   Wed Jul 6 16:09:21 2016 +0800

    fix some spelling mistakes in doc/

    I check all the files under doc/ directory and find three
    spelling mistakes
     - exeuction should be execution
     - Fefora should be Fedora
     - opentackclient should be openstackclient

    Change-Id: If9e5d07b6558871bb3f8d55b52bf8f1d9db0897e

commit 4ce7dd53e8bbd70a97a667c7b39078d73495ec1f
```



最后一步是git review  

```
root@devstack:/home/python-openstackclient/doc/source# git review
remote: Processing changes: new: 1, refs: 1, done
remote:
remote: New Changes:
remote:   https://review.openstack.org/338097 fix some spelling mistakes in doc/
remote:
To https://jqjiang.1:mK7+T+WV3NbMIzh31ym9g6rVnrUsZ94XSugvXpvIZQ@review.openstack.org:443/openstack/python-openstackclient.git
 * [new branch]      HEAD -> refs/publish/master/trivial
```

这个就是我的第一个patch提交界面  

![image](/images/openstack-bug-fix/2.png)



社区的core给我review完了之后，代码就被merge到master主分支上了  


![image](/images/openstack-bug-fix/3.png)
