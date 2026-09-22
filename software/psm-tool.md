# 下载工具

W3上看到的两个版本的，不知道有啥区别。

[psm_tool.zip](../installer/psm/psm_tool.zip)
[psm-tool.rar](../installer/psm/psm-tool.rar)

# 调用方式

1.加密数据

```
./psm-tool --optype=endata --inputdata=<需加密的字符串>
```

2.解密数据

```
./psm-tool --optype=dedata --inputdata=<需解密的字符串>
```

3.加密文件

```
./psm-tool --optype=encrypt --infile=<需加密的文件路径> --outfile=<生成的密文文件路径>
```

4.解密文件
```
./psm-tool --optype=decrypt --infile=<需解密的文件路径> --outfile=<生成的明文文件路径>
```

# 报错

如果报错缺失文件，在容器里找一下`root.key`、`common_shared.key`。

![image](../png/software/28.png)

找到了的话，设置一下环境变量。

```
export PAAS_CRYPTO_PATH=这两个东西所在的路径
```

如果没找到，在容器里找一下`common1`、`common2`，把它们拷走，放到别的目录下，并分别改名一下。再设置一下环境变量。（不设置环境变量也行，把psm-tool和这两个东西放在同一个目录就行）

```
common1->root.key
common2->common_shared.key
```


#解密软件PSM-TOOL #解密软件 #PSM #TOOL #psm_tool #psm-tool