---
title: visual studio code在terminal中下划线不显示
date: 2018-07-15 18:15:52
tags: [生产力工具, vscode, 下划线]
category: [ENV]
---
visual studio code 在terminal中看不到下划线，网上有两种解决方案：
* 修改主题
    * File->Preferences->Color Theme[Ctrl+K Ctrl+T]
    * 或者直接快捷键[Ctrl+K Ctrl+T]。
* 修改字体（仅测试过过ubuntu）。
    *  File->Preferences->Settins[Ctrl+,]
    *  或者直接快捷键Ctrl+。在右边的User setting中添加如下内容。
```
"editor.fontFamily": "'Noto Mono', 'Droid Sans Mono', 'Courier New', monospace, 'Droid Sans Fallback'",
或者
"editor.fontFamily": "'Ubuntu Mono', monospace",
```