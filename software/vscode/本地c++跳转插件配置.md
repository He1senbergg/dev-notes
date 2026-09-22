# 1. 下载插件`clangd`

![image](../../png/software/35.png)

# 2. 修改设置的配置文件

- 在VSCode中打开wsl的任意文件夹，进入wsl。
- 点击上面的框。
    ![image](../../png/software/30.png)
- 键入`>setting`。
    ![image](../../png/software/31.png)
- 跳转wsl的设置。
    ![image](../../png/software/32.png)
- 点击json显示
    ![image](../../png/software/33.png)
- 键入内容
    ```
    {
    "C_Cpp.intelliSenseEngine": "disabled",
    "clangd.arguments": [
    "--compile-commands-dir=${workspaceFolder}/.vscode"
    ]
    }
    ```
    ![image](../../png/software/34.png)