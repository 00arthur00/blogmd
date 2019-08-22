---
title: ubuntu18.04环境配置
date: 2019-06-11 23:37:20
tags: [ENV]
---
## 设置最大打开文件数
1. 图形界面登录: 设置/etc/systemd/user.conf及/etc/systemd/system.conf中的DefaultLimitNOFILE
```
    DefaultLimitNOFILE=65535
```
2. 非图形界面登录:修改/etc/security/limits.conf或在/etc/security/limits.d目录下新创建一个文件，然后设置
```
    * soft nofile 65535
    * hard nofile 65535
```

**NB**:设置之后需要重新启动