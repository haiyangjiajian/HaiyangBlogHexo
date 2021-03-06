---
layout: post
title: git实用技巧
tags: [tools]
category: tool
---

在工作之中使用git，除了常用的clone，add，commit，push，fetch，pull，merge等还会经常出现一些其他需要解决的问题，现在将其总结下:

<!-- more -->

1. 忘记添加.gitignore


	GIT新手最容易犯的一个错误就是没有添加.gitignore，而把不该上传的东西上传了， 而GIT老手有时会因为在规则里面写了个.*而导致.gitignore没有加入到版本控制当中， 事后才发现，但此时项目中已充斥了“垃圾”。
	
	此时项目管理者才追悔莫及，修改.gitignore并提交到版本控制当中。 但大错已铸成，新添的.gitignore不会影响已经加入到项目中的文件，GIT老手此时也可能没有什么好办法， 只能把不该有的东西手动删除掉，再重新提交。但更麻烦的是，这些“垃圾”可能还有用， 如Java项目中依赖的一些*.jar库文件，直接删了会出问题，要在修好项目后重新加回来。 如果只有几个文件还好，如果成百上千，这样操作，一天都不用干别的了。
	
	但问题总会有聪明办法解决。GIT中用git rm --cached xxx可以在不动项目当前工作空间的情况下， 将文件从当前（未提交）版本中移除。如此而来简单方法就出来了：
	
	```bash
	git rm -r --cached .
	git add .
	git commit -m ".gitignore is now working"
	```
	[参照这里](http://davidaq.com/technique/share/2015/04/22/gitignore-update.html)
	
2. git 取消push

	```
		方法1:
			1. git reset -\-hard HEAD~1
			2. 然后再使用git push origin "your branch name" -\-force将本次变更强行推送至服务器。这样在服务器上的最后一次错误提交也彻底消失了。
		
		方法2:
			1. 在本地git revert，覆盖掉上一次的commit
			2. git push origin "your branch name"
	```
3. git reflog

	git reflog/git log -g记录每次修改head的操作，可以查看所有历史修改记录,然后	通过git reset命令进行恢复
4. git merge和git rebase

	如果需要merge远程分支的东西，尽量不要用git pull。 可以使用：
	
	```bash
	git fetch origin
	git rebase origin/master   // 可以理解为：将服务器master分支映射到本地的一个临时分支上，然后将本地分支上的变化合并到这个临时分支，然后再用这个临时分支初始化本地分支
	```
5. git stash

	git stash 将未commit改变缓存,不保存新加的文件
	
	git stash list 展示栈列表
	
	git stash apply {stash@{2}} 应用某次栈中的内容
	
	git stash pop 应用栈顶内容并弹栈
	
	git stash clear 清空队列
6. git revert和git reset

	git revert是用一个新的提交取消掉了之前的一个错误提交
	
	git reset取消掉最近一次commit后add的
	
	git reset []回到那一次commit，并且将撤销掉的文件的提交恢复成未提交的状态
	
	git reset --hard 回到那一次commit，所有过程中的文件都没了
7. git自动补全

	[参见github](https://github.com/git/git/blob/master/contrib/completion/git-completion.bash)
8. 单独统计每个人的增删行数

	```bash
	git log --format='%aN' | sort -u | while read name; do echo -en "$name\t"; git log --author="$name" --pretty=tformat: --numstat | awk '{ add += $1; subs += $2; loc += $1 - $2 } END { printf "added lines: %s, removed lines: %s, total lines: %s\n", add, subs, loc }' -; done
	```
9. 打tag
	```bash
	git tag -a v1.1.2 -m 'my version' 新增tag
	git push origin v1.1.2 将新建的tag推到远端
	git tag 列出现有标签
	git show v1.1.2 查看相应标签的信息
	```
10. 错误的创建了一个文件config.java提交了以后，发现应该改为大写Config.java。但是执行git status后没有任何提示。研究后发现原因是：git 默认对于文件名大小写是不敏感的，上面修改了首字母大写,但是git并没有发现代码任何改动。解决方案是配置git config使它对大小写敏感

	```bash
	git config core.ignorecase false
	```
11. git rebase
	
	最低要求是没有别人正在这个分支上工作，但是即使只有自己工作在这个分枝上，如果曾经将这个分枝push到server过，本地rebase分枝B，再push到server，本地的B会和server的B发生冲突，不得不merge或者git push -f。所以最好rebase一个从没有push到server过的分枝。
	
	rebase并不是将B的commit如C直接移动到了别的分枝上，而是创建了一个副本C1，二者的hashcode并不相同
	
	```bash
	git rebase -i master 交互式的rebase
	```
12. WIP标志
	一些merge request可能并不希望被merge可以将其title以[WIP] 或者 WIP:开头，这样不会被merge，知道标题中移除了[WIP] 或者 WIP:
	
	
	
	