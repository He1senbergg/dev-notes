当远端有领先于本地的代码，导致VSCode的`git commit`失败时，可以用`git reset`，也可以用`git rebase`。

执行命令

```
git fetch origin
git pull --rebase origin <分支名>
git push origin <分支名>
```

---

更新远程分支信息

```
git fetch origin
```

把远程的提交先拿下来，然后把本地的提交重新接到远程最新提交之后

```
git pull --rebase origin <分支名> 
```

再推送

```
git push origin <分支名>
```