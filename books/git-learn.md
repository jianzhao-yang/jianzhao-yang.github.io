---
title: git-learn
date: 2019-03-21 16:42:59
tags:
  - git
---

##  Git学习笔记
***
## 一、基本操作
【创建版本库】

1. git init 初始化一个git仓库  
1. git add <文件名> 添加文件到git记录(git不会像svn一样循环扫描文件夹内的变动,性能更好)  
1. git commit -m "提交文件改动的说明"  提交改动  
    git commit --amend 补充提交,如果在上次提交时有遗漏或提交信息填写错误,可以使用这个覆盖上次提交  
注意：每次改动之后都要使用git add 提醒git记录,否则无法成功使用git commit提交!  
 关于git add 和git commit区别的详细说明自行百度 

【查看版本区别】    
<!-- more -->
4. git status 查看当前版本仓库(当前文件夹)各个文件是否都在git记录之下  
    如果有红色提示,说明当前文件夹下还有没有被git记录的改动  
    如果有蓝色提示,说明当前文件夹下还有没有提交的改动  
5. git diff 查看当前版本库中的内容(默认为暂存区)跟当前工作空间中的内容之间的区别  
    git diff --staged 查看暂存区里缓存未提交的内容

【版本回溯】  
6. git log 查看提交的各个历史版本信息  
     git log --pretty=oneline 美化历史版本输出  

7. git reset 版本回退  

  ```sh
  git reset  HEAD 用于清除暂存区记录的改动  
   		HEAD^^ 回退到上上一个版本  
   		HEAD~10 回退到上10个版本      
   		--hard   回退到指定版本,工作区和暂存区都被清理    危险命令！！！可能导致工作区修改丢失  
   		--soft   回退到指定版本,指定版本之后的修改都被添加到暂存区保存起来  
   		--mixed  回退到指定版本,工作区内容不修改,暂存区内容被清理  
  ```

  

8. git reflog 查看所有增删改操作,版本回退的后悔药  

【配置】  
9. 控制台中文乱码  
    使用git-bash命令行,右键->选项->文本->字符集选择utf-8 
10. git diff正文中文乱码  
    在命令行下输入以下命令：

    ```sh
     git config --global core.quotepath false
     git config --global gui.encoding utf-8
     git config --global i18n.commit.encoding utf-8
     git config --global i18n.logoutputencoding utf-8 
     export LESSCHARSET=utf-8
    ```

【暂存区】  
暂存区是git中的一个很重要的概念，它是git对本地修改的记录，但是还没有提交到版本仓库中的内容
需要自行学习详细了解

11. git checkout -- <文件名> 如果暂存区无该文件改动，从版本仓库拉取最新版本文件；  
      如果暂存区有该文件改动，同步暂存区的修改到工作区  
      无论如何，本地改动都会被覆盖  
      git checkout <分支名> 拉取分支
12. git rm <文件名> 从git中删除文件（需要commit）
13. git mv <原文件名> <新文件名> 移动文件/重命名文件
14. git rm 删除文件  
    git rm 删除版本库中的文件和本地文件(如果暂存区有数据则报错)  
    git rm -f 连暂存区中的文件也删除  
    git rm --cached 只删除版本库和暂存区的数据,保留本地文件  

【忽略文件】    
.gitignore 描述git应该忽略哪些文件的文件   
文件中注释以#开头  
示例:

\# 忽略.a结尾的文件  
\*.a  
\# 忽略.a到.z,.0到.9,以及以.~结尾的文件  
\*.[a-z0-9~]  
\# 忽略.a加任何一个字符结尾的文件  
\*.a?  
\# 但是不忽略lib.a结尾的文件  
!lib.a  
\# 忽略TODO文件夹下面的所有内容,但是不忽略子文件夹  
/TODO  
\# 忽略build文件夹下的所有内容,包括子文件夹  
build/  
\# 忽略TODO文件夹下的所有.txt结尾的文件,但是不忽略子文件夹里的  
doc/\*.txt  
\# 忽略TODO文件夹下的所有.pdf结尾的文件,包括子文件夹里的  
doc/**/\*.pdf  

【打标签】  
1. git tag 显示当前仓库中的所有标签  
2. git tag -l '1.\*' 列出名称为1.\*的所有标签  
3. git show <标签名> 查看标签信息  
4. git tag -a <标签名> <版本号> 给指定版本打标签  
5. git push <远程仓库url/别名> <已有标签名> 推送标签到远程仓库  
    git push <远程仓库url/别名> --tags 推送本地所有标签到远程  
6. git checkout -b <分支名> <标签名> 检出指定分支的指定标签到本地  

【别名】  
git config --global alias.<别名> '原命令'   
例如:  
git config --global alias.unstage 'reset HEAD --'  
git unstage fileA 相当于 git reset HEAD -- fileA  
    
## 二、分支操作
1. ```
      git branch       列出所有分支  
      		-v          +列出最后一次提交   
      		-vv         +列出跟踪的远程分支  
      		--merged    查看已经合并到当前分支的分支  
      		--no-merged 查看尚未合并到当前分支的分支   
      ```

         

2. git branch <分支名> 创建分支  

3. git branch -d <分支名> 删除分支  

4. git checkout <分支名> 切换分支  
    git checkout -b <分支名>  创建并切换分支  

5. git merge <分支名> 把目标分支合并到当前分支上  
    git merge --no-ff -m "提交说明" dev 将dev分支合并到当前分支,合并操作生成一次commit记录
    【变基】慎用!
    前提：不要对在你的仓库外有副本的分支执行变基。（只能对只存在于你本地的分支进行变基操作）
    变基：修改一个分支的来源（分叉位置）

6. git rebase <分支名> 把当前工作目录分支的分叉调整到指定分支上

7. git rebase <A分支> <B分支> 把B分支上做的修改都提交到A分支上(不是二合一的合并,而是一 一串联)

8. git rebase --onto master server client 
    分支server来源于master,client来源于server
    把client分支上做的修改提交到master上,server分支上进行的修改(即server和client的交集)不合并
        
## 三、远程仓库
1. git clone <远程仓库url地址> 克隆远程仓库到本地
2. git remote 查看远程仓库名
      -v 列出所有远程仓库
3. git remote add <别名> <远程仓库url> 为远程仓库创建别名,方便使用
4. git remote rename <旧别名> <新别名> 重命名远程仓库
5. git remote rm <别名> 删除别名
6. git fetch <远程仓库url/别名> 从指定远程仓库拉取当前分支最新版本文件 不合并
7. origin 指代上一次拉取文件的远程仓库(git自动添加的别名)
8. git push <远程仓库url/别名> <分支名> 推送本地改动到指定远程仓库的分支
9. git remote show <远程仓库名> 显示远程仓库与本地工作空间的区别

## 四、git rebase 变基
merge是把两个分支合并为一个新的节点,新节点为目标分支的head    

rebase是指把当前分支的来源分支修改为指定分支,参见https://git-scm.com/book/zh/v2/Git-%E5%88%86%E6%94%AF-%E5%8F%98%E5%9F%BA    


rebase黄金法则 : 绝不要在公共的分支上使用它(只有这个分支是你自己一个人独用的时候才可以使用)    

rebase -i head~3 把当前分支的来源重定向到head的前3个版本    

## 五、git stash 暂存当前的改动    
git stash 会将当前工作区的修改存入暂存区    
          show 查看暂存区的改动    
          list 查看暂存区每次stash的列表   
          apply stash@{n} 应用最近的第n次储藏，不指定默认为最新的  
          pop 取出并删除最新的储藏  
          drop stash@{n} 删除最近第n次储藏  
          clear 清除所有储藏!!  
          save 名字 本次储藏并命名  
          branch 分支名 从储藏中创建分支  

## 六、git cherry-pick  只引入某次提交  
可以理解为”挑拣”提交，它会获取某一个分支的单笔提交，并作为一个新的提交引入到你当前分支上。 
当我们需要在本地合入其他分支的提交时，如果我们不想对整个分支进行合并，而是只想将某一次提交合入到本地当前分支上  
git cherry-pick 版本号 把某个版本所做的提交提交到当前分支,默认自动提交  
                        -n 不自动提交  
                        -e 重新填写提交信息  
                 中断后的后续操作         
                -–continue 如果提交信息跟当前版本库有冲突,会停下来进行手动合并,之后用此命令继续  
                --quit  不再继续本次提交  
                --abort 撤销本次cherry-pick所做的修改  
                
                
[本地仓库先创建,推送到远程仓库的办法]  
把远程拉下来然后将当前的分支rebase到master上  
git pull --rebase origin master  
