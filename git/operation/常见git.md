# 一、CLion中的Git

## 1. git clone下载文件

```
git clone 网址
```

## 2. 手动用CLion打开文件

## 3. 拉取当前最新的main

依次点击`终端`、`本地`，输入git pull，拉取当前最新的main。

```
git pull
```

![image](../../png/git/1.png)

## 4. 创建自己的分支

在网页上创建自己的分支。

## 5. 切换到自己的分支

在CLion切换到自己的分支。

![image](../../png/git/2.png)

## 6. 检查分支

在CLion检查当前打开的分支的正确性。

![image](../../png/git/3.png)

## 7. 处理代码冲突

在CLion中处理代码仓中的代码冲突。

依次点击`终端`、`本地`，输入git pull，拉取当前最新的main。

```
git pull
```

![image](../../png/git/4.png)

![image](../../png/git/5.png)

![image](../../png/git/6.png)

![image](../../png/git/7.png)

出现**冲突**弹窗：

![image](../../png/git/8.png)

逐个双击出现代码冲突的代码文件，会出现“当前修改版本”“代码原始版本”“当前远端系统修改版本”三个连着的界面（不可拆分，必要时需拉长为双屏的宽度），随后开始逐个修改。

在每个文件打开之后，右上角有**当前需要处理的冲突数**，右下角有**应用**按钮，在确认所有冲突的解决方案之后，需要按下**应用**。

![image](../../png/git/9.png)

## 8. 新提交之前

要先拉下来

```
git pull
```

再如**7.**中的merge

再提交

```
git push
```

## 9. 分支上出现分叉的进度条

```
git pull --rebase origin <分支名>
```

作用是，强制把当前分支的新代码拉下来，然后把现在的修改放在这些commit的后面。风险是，可能当前的修改会丢，但是也没什么别的办法了。

完事后，再推一下代码。

```
git push
```

#rebase #push #pull #git rebase #git push #git pull