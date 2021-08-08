---
title: Eclipse插件收藏
date: 2021-07-31 13:19:05
tags:
toc: true
---

记录一下自己使用的Eclipse插件：

### 1.Commander

对于使用 Sublime 和 Visual Studio Code 的开发者来说，CTRL+P 快捷键打开的命令工具非常好用。Commander 就是 Eclipse 这样的一个插件，它支持输入命令，然后执行一个操作。比如，对于我记性不好的人来说，一直记不起将变量转换为大写（或者小些），那么我可以调出 Commander 输入框，然后按下 *Tab* 键，输入 uppercase 关键字，就可以执行转换为大写的操作。它实际上是执行一个快捷键。效果图如下：

![eclipse_plugin_commander](/images/eclipse_plugin_commander.gif)



**安装方法**：在 Eclipse 的 Marketplace 输入 Commander 安装即可。

**配置**：参考 [https://github.com/d-akara/eclipse-plugin-commander](https://github.com/d-akara/eclipse-plugin-commander)



### 2.CompleteLine

使用 *Ctrl+Shift+Enter* 自动补全自动补全单前行，看看效果图：

![eclipse_plugin_complete_line](/images/eclipse_plugin_complete_line.gif)

**安装方法**：参考 [https://github.com/henri5/completeline](https://github.com/henri5/completeline)



### 3.EasyShell

在 Eclipse 中，选中文件，可以立即在文件浏览器中打开，或者跳转到命令号中，效果图如下：

![eclipse_plugin_easyshell](/images/eclipse_plugin_easyshell.gif)

**安装方法**：参考 [https://github.com/anb0s/EasyShell](https://github.com/anb0s/EasyShell)



### 4.SpotBugs

代码审查工具，可以发现代码潜在的漏洞。

使用方法：

1.先点击 Find Bugs 菜单，如下图；

![eclipse_plugin_spot_bugs_1](/images/eclipse_plugin_spot_bugs_1.jpg)

2.在 Commander 输入 spotbugs 打开 spotbugs 的视图，如下图：

![eclipse_plugin_spot_bugs](/images/eclipse_plugin_spotbugs.jpg)

**安装方法**：参考 [https://spotbugs.readthedocs.io/en/stable/eclipse.html](https://spotbugs.readthedocs.io/en/stable/eclipse.html)