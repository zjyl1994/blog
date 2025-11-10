---
title: "VSCode 配置任意位置 F5 调试 Golang"
date: 2022-11-16T20:44:00+08:00
---

VSCode 安装 Golang 插件后，如果安装了配套的gotools, 实际上已经可以支持调试了。

但是我们按下F5后会报错，提示 `could not launch process: not an executable file`。

原因是你没有给工作区配置 `launch.json` 导致VSCode找不到你要运行的程序位置。
默认的设置，你按下F5会尝试运行当前打开的页面，然而大部分情况下我们都不会选择 `main` 包的文件而是正在某个文件内部设置断点，
所以仅仅编译当前文件是不够的。

如何解决呢？

在当前项目中创建 `.vscode/launch.json`，填入以下内容：

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Launch Package",
            "type": "go",
            "request": "launch",
            "mode": "auto",
            "program": "${workspaceRoot}"
        }
    ]
}
```

保存后即可在任意位置按F5启动调试了。