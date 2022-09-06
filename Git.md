### Git



#### 生命周期

![1566540541368](image/1566540541368.png)

暂存区（Index）：执行add命令后的文件区域，第一次add后的内容，肯定会存放在暂存区

仓库（Head）：已跟踪、已保存修改。最后一次commit到的区域

工作区：已跟踪、已修改，但没有进行后续add文件的区域



#### 常用命令

> 【】表示IDEA的操作，origin表示远程仓库名
>

* **git branch**

  查看分支

  -r：查看远程分支

* **git status**

  查看状态

* **git checkout**

  把工作区的改动撤销，回到暂存区或仓库，即最后一次add或commit的状态

  （先stash，然后checkout，再unstash【Smart Checkout】）

* **git add**

  添加内容到下一次提交中（跟踪新文件并把其放到暂存区，或把已跟踪的文件放到暂存区）

  -u：只modified的文件，不添加untracked文件

  . / -A：所有变化【选中目录或文件 -> Add】

* **git commit**

  添加到仓库

  -m "name"：添加描述

  -a：先modified文件，再commit【Commit】

  --amend：替换上一次的commit【Amend Commit】

* **git reset**

  调整HEAD指针位置，可以删除已经提交的commit

  HEAD ：把暂存区的改动撤销，放回到工作区（若继续git checkout，则会回到仓库）

  -soft：只清除仓库（之后的所有commit），不清除暂存区和工作区

  -mixed：只清除仓库和暂存区（暂存区的文件名称从绿色变成红色），不清除工作区

  -hard：三个区域的改动全部清除

* **git revert**

  一个相反的commit来回滚指定版本所做的修改

  HEAD：撤销上一次

* **git stash**

  把暂存区和工作区的改动存起来【Stash】

  -k：不保存暂存区的改动，只保存工作区的改动【Keep Index】

* **git statsh pop**

  把暂存区和工作区的改动取出来【Unstash】

* **git merge**

  合并（把dev-merge的代码合并到dev-3上）

![git-01](image/git-01.png)

--ff：默认方式，条件是原先节点没有修改，合并后不会创造新的节点。删除了dev-merge分支后还是会有该分支的记录【Merge】

![git-02](image/git-02.png)

--no-ff：即使在fast forward条件下也会产生一个新的commit（灰色描述）。删除了dev-merge分支后还是会有该分支的记录【No fast forwad】

--no-commit：合并后需要手动执行一次commit，结果与--no-ff命令相同【No commit】

![git-03](image/git-03.png)

--squash：dev-3分支上添加dev-merge分支的所有改动，再额外执行一次commit，之后在dev-3分支上将看不到dev-merge分支的commit记录【Squash commit】

![git-04](image/git-04.png)

--log：添加合并日志【Add log information】

![1564024722081](image/1564024722081.png)

* **git rebase**

  -i \<commit-id\>：

* **git cherry-pick**

  把其它分支的改动复制到当前分支，并执行commit

  \<commit-id\>：每个ID都会执行一次commit【樱桃按钮】

  \<start-commit-id\>…\<end-commit-id\>：(左开，右闭]

* **git reflog**

  查看所有分支的所有操作，包括已删除的commit和reset

* **git fetch**

  拉取最新的版本到本地仓库

* **git pull**

  fetch + merge

* **git push**

  推到远程仓库

  origin :\<branch\>：删除远程分支（origin后面有空格）

  -f ：强制推




#### IDEA VCS

Reformat code：格式化代码，重排布

Optimize imports：优化导入的包

* 文件颜色

  新创建：红色

  add后：绿色

  commit前：深蓝色

  commit后：基本色

* Tag

  当前分支（HEAD）：黄色

  本地分支（分支名）：绿色

  远程分支（远程地址 / 分支名）：紫色

  普通标签，不代表分支：白色



#### GitHub常用缩写

https://blog.csdn.net/jiajia4336/article/details/99673143



#### Pull Request

1. 发起Issues，描述问题
2. 等待Owner确认和回复
3. Fork项目，修改代码，Push
4. 发起Pull Request
5. 等待Owner审核和合并分支
6. 关闭Issues
