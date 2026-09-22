先随便找个能创建文件的目录，创建个文件`ll.bat`

写入内容

```
@echo off
dir /a /o /p %*
```

把这个文件移动到`C:\Windows\ll.bat`。

就可用了。

![image](../png/engineering/56.png)

#ll #windows #CMD #cmd