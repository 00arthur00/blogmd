---
title: ubuntu18.04剪切版工具
date: 2018-07-16 22:36:05
tags: [生产力工具, Parcellite, Ditto]
category: [ENV]
---
## Gpaste安装
GPaste 是一个剪贴板管理系统，它包含了库、守护程序以及命令行和 Gnome 界面（使用原生 Gnome Shell 扩展）。

1. 运行如下命令
``` bash
sudo apt-get install gnome-shell-extensions-gpaste gpaste
sudo apt-get install gnome-tweaks
```
2. 搜索tweaks，并打开
3. 点击 Extensions->Gpaste，设置为on
4. 按Alt+F2，输入r并回车，重启gnome使gpaste生效

## Gpaste使用
你可以使用快捷键从顶栏开启 GPaste 历史记录（Ctrl + Alt + H）或打开全部的 GPaste GUI（Ctrl + Alt + G）。

该工具还包含这些键盘快捷键（可以更改）：

从历史记录中删除活动项目： Ctrl + Alt + V

将活动项目显示为密码（在 GPaste 中混淆剪贴板条目）： Ctrl + Alt + S

将剪贴板同步到主选择： Ctrl + Alt + O

将主选择同步到剪贴板：Ctrl + Alt + P

将活动项目上传到 pastebin 服务：Ctrl + Alt + U

## 安装parcellite
```
sudo add-apt-repository ppa:rickyrockrat/parcellite-appindicator
sudo apt-get update && sudo apt-get install parcellite
```
ubuntu18.04收到404错误

## 使用parcellite
* 在activity中搜索parcellite，并点击程序，则其图标会出现。
* 右击可以通过preferences查看快捷键。
* 默认的查看历史快捷键是"Ctrl+Alt+H"
现在可以复制(ctrl+C)，然后ctrl+alt+H来显示以前的复制信息了，选择好之后，直接ctrl+v就行了。