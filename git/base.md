



```
git config --local  某个仓库

Git config --global  全局

git config --system  对系统所有登陆用户

git config --list --local
```



git add -u 只增加追踪了的文件



git mv oldname newname



git reset --hard



git log -n10 --oneline

git log --all --graph





git checkout -b dev 902ad33{commitID或者分支名字}



git checkout -b feature origin/feature



Git branch -av 输出带commitid

git branch -d|D featurename



.git/refs/heads/ 当前有多少分支

.git/HEAD 当前head指向那个分支



.git/refs/heads/master 6a8102e2d3fdc90fd0d6e3ffd4736a8bf8f1b152

git cat-file -t 6a8102e2d3fdc90fd0d6e3ffd4736a8bf8f1b152



git diff HEAD HEAD~{n} -- filename

 git diff commitID commitID

git diff --cached head 暂存区 历史

git diff 工作区 暂存区



git commit --amend

git commit --amend --allow-empty 只修改commi信息



git rebase -i

git rm file



git stash

git stash list

 git stash apply|pop



移除版本库追踪

```bash
git  rm  -r  --cached   文件  
```

-n 只运行命令 不实际执行



git remote add branchname file://package/a.git

git clone --bare 地址 名字



git fetch



git merge --abort 取消合并