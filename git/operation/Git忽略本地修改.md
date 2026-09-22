
# 添加忽略本地的文件。

```
git update-index --skip-worktree
git update-index --skip-worktree CMakeLists.txt
git update-index --skip-worktree compile.sh
git update-index --skip-worktree src/CMakeLists.txt
git update-index --skip-worktree src/*/CMakeLists.txt
```

![image](../../png/git/10.png)

# 查看已被忽略的本地文件。

```
git ls-files -v | grep ^S
```

![image](../../png/git/11.png)

# 将已被忽略的本地文件重新进入git commit

```
git update-index --no-skip-worktree CMakeLists.txt
git update-index --no-skip-worktree compile.sh
```

#Git忽略本地修改的文件 #.gitignore #ignore