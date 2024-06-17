---
{"dg-publish":true,"permalink":"/Program/Mixed/最常用的IntelliJ IDEA快捷键/","noteIcon":""}
---

1.  [最新发布](https://www.didispace.com/)
2.  [玩转IDEA](https://www.didispace.com/idea-tips/)
3.  [Shortcutkeys](https://www.didispace.com/idea-tips/shortcutkeys/)
4.  [最常用的IntelliJ IDEA快捷键（含Windows、MacOS双版本）](https://www.didispace.com/idea-tips/shortcutkeys/allinone.html)

2023年11月2日IntelliJ IDEAIntelliJ IDEA大约 8 分钟

* * *

本文Mac快捷键风格为Intellij IDEA Classic，如不是则首先需要在Preferences中切换

![](https://static.didispace.com/images3/1dbff4bcdea6e1931fe2fcaec5cfb22e.png)

一. Mac符号缩写
----------

**Mac电脑键盘的符号缩写说明如下，下面可能会用到**

| **标记** | **按键** |
| --- | --- |
| ⌘ | Command |
| ⇧ | Shift |
| ⇪ | Caps Lock |
| ⌥ | Option |
| ⌃ | Control |
| ↩ | Return/Enter |
| ⌫ | Delete |
| ⌦ | 向前删除键（Fn+Delete） |
| ↑ | 上箭头 |
| ↓ | 下箭头 |
| ← | 左箭头 |
| → | 右箭头 |
| ⇞ | Page Up（Fn+↑） |
| ⇟ | Page Down（Fn+↓） |
| Home | Fn + ← |
| End | Fn + → |
| ⇥ | 右制表符（Tab键） |
| ⇤ | 左制表符（Shift+Tab） |
| ⎋ | Escape (Esc) |
| ⏏ | 电源开关键 |

二. 基础操作
-------

#### 1\. 基础定位与编辑

| **操作** | **Windows** | **Mac(OS X)** |
| --- | --- | --- |
| 剪切 | Ctrl + X | ⌘X |
| 复制 | Ctrl + C | ⌘C |
| 粘贴 | Ctrl + V | ⌘V |
| 从最近的缓冲区粘贴(弹出面板供选择) | Ctrl + Shift + V | ⌘⇧V |
| 撤销 | Ctrl + Z | ⌘Z |
| 删除光标所在行代码 | Ctrl + Y | ⌘Y |
| 复制光标所在行，并把复制内容插入下一行 | Ctrl + D | ⌘D |
| 递进式选择代码块。连续按会扩大选中范围，从词到句到段 | Ctrl + W | ⌘W |
| 在当前文件跳转到某一行的指定处 | Ctrl + G | ⌘G |
| 字面量大小写切换 | Ctrl + Shift + U | ⌘⇧U |
| 注释光标所在行代码，会根据当前不同文件类型使用不同的注释符号 | Ctrl + / | ⌘/ |
| 块注释 | Ctrl + Shift + / | ⌘⇧/ |
| 基础代码补全，默认被输入法占用，需要进行修改，建议修改为 Ctrl + 逗号(KeyMap->Main menu –> Code –> Completion->Basic) | Ctrl + Space | ⌃Space |
| 智能代码补全 | Ctrl + Shift + Space | ⌃⇧Space |
| 删除光标后面的单词或是中文句 | Ctrl + Delete | ⌥Fn⌫ |
| 删除光标前面的单词或是中文句 | Ctrl + BackSpace | ⌥⌦ |
| 光标跳转到当前单词(中文句)/当前行的左侧开头位置 | Ctrl/Alt + 左方向键 | ⌥←/⌘← |
| 光标跳转到当前单词(中文句)/当前行的右侧开头位置 | Ctrl/Alt + 右方向键 | ⌥→/⌘→ |

#### 2\. 代码块级编辑操作

| **操作** | **Windows** | **Mac(OS X)** |
| --- | --- | --- |
| 展开代码块 | Ctrl + 加号 | ⌘+ |
| 折叠代码块 | Ctrl + 减号 | ⌘- |
| 代码块全部折叠 | Ctrl + Shift + 减号 | ⌘⇧- |
| 移动光标到当前所在代码的花括号开始/结束位置 | Ctrl + \]/Ctrl +\[ | ⌘\] / ⌘\[ |
| 选择光标处到代码块结束/开始的范围 | Ctrl + Shift + \]/ Ctrl + Shift + \[ | ⌘⇧\] / ⌘⇧\[ |
| 重写父类方法 | Ctrl + O | ⌘O |
| 实现方法 | Ctrl + I | ⌘I |
| 包围代码（使用if..else, try..catch, for, synchronized等包围选中的代码） | Ctrl + Alt + T | ⌘⌥T |
| 生成代码(set/get方法，构造函数等) | Alt + Insert | ⌃↩/⌃N |
| 插入自定义动态代码模板 | Ctrl + J | ⌘J |
| 动态代码模板环绕 | Ctrl + Alt + J | ⌘⌥J |
| 格式化代码 | Ctrl + Alt + L | ⌘⌥L |
| 优化import | Ctrl + Alt + O | ⌘⌥O |

三. 查询替换定位
---------

此处主要处理对象为**变量(field)**和**方法(method)**

#### 1. 查看定义与文档

| **操作** | **Windows** | **Mac(OS X)** |
| --- | --- | --- |
| 显示代码简要信息 | Ctrl + 鼠标悬浮代码上 | ⌘鼠标悬浮代码上 |
| 快速查看文档(用在变量上，则显示变量初始化语句) | Ctrl + Q | ⌃J/⌃鼠标中键 |
| 方法参数提示显示 | Ctrl + P | ⌘P |
| 在打开的文件标题上，弹出该文件路径 | Ctrl + 左键单击 | ⌘鼠标左键 |

#### 2\. 查询使用情况

| **操作** | **Windows** | **Mac(OS X)** |
| --- | --- | --- |
| 查看选择目标在项目中的使用 | Alt + F7 | ⌥F7(Fn) |
| 查看选择目标在本文件中的使用 | Ctrl + F7 | ⌘F7(Fn) |
| 查看选择目标在本文件中的使用(高亮显示) | Ctrl + Shift + F7 | ⌘⇧F7(Fn) |
| 依次遍历每个选中的目标 | F3 | F3(Fn) |

#### 3. 跳转定义与调用处

| **操作** | **Windows** | **Mac(OS X)** |
| --- | --- | --- |
| 进入选择目标的定义处或使用处 | Ctrl + B/ Ctrl + 鼠标左键 | ⌘B/ ⌘鼠标左键 |
| 进入选择目标的实现处 | Ctrl + Alt + B/ Ctrl + Alt + 鼠标左键 | ⌘⌥B/ ⌘⌥鼠标左键 |
| 前往选择目标的父类的方法 / 接口定义 | Ctrl + U | ⌘U |
| 跳转到返回类型的声明处 | Ctrl + Shift + B | ⌘⇧B |

#### 4\. 高级查询/定位/替换(复杂查询，会直接弹出对话框)

| **操作** | **Windows** | **Mac(OS X)** |
| --- | --- | --- |
| 文本查找(当前文件) | Ctrl + F | ⌘F |
| 文本替换(当前文件) | Ctrl + R | ⌘R |
| 文本查找(全局) | Ctrl + Shift + F | ⌃⇧Fn F |
| 文本替换(全局) | Ctrl + Shift + R | ⌃⇧ R |
| 根据输入的类名，查找类文件 | Ctrl + N | ⌘N |
| 根据输入的文件名，查找文件 | Ctrl + Shift + N | ⌘⇧ N |
| 查找在类中的方法 | Ctrl + Alt + Shift + N | ⌘⌥⇧N |
| 查询任何东西 | 双击Shift | 双击⇧ |
| 查找动作(说明书，很好用，当不记得快捷键时可以用这个查询) | Ctrl + Shift + A | ⇧⌘A |

#### 5. 错误与异常查询

| **操作** | **Windows** | **Mac(OS X)** |
| --- | --- | --- |
| 依次定位每个错误或者警告 | F2 | F2(Fn) |
| 在光标所在的错误代码处显示错误信息 | Ctrl + F1 | ⌘F1(Fn) |
| 显示意向动作和快速修复代码 | Alt + Enter | ⌥↩ |
| 查看外部文档（在某些代码上会触发打开浏览器显示相关文档） | 未知 | (⇧)F1(Fn) |

四. 导航
-----

#### 1\. 代码文件结构

| **操作** | **Windows** | **Mac(OS X)** |
| --- | --- | --- |
| 弹出当前文件结构层，可以在弹出的层上直接输入进行筛选（可用于搜索类中的方法） | Ctrl + F12 | ⌘F12 (Fn) |
| 显示当前类的层次结构 | Ctrl + H | ⌃H |
| 显示方法层次结构 | Ctrl + Shift + H | ⌘⇧H |
| 显示调用层次结构 | Ctrl + Alt + H | ⌃⌥H |

#### 2\. 操作记录查询

| **操作** | **Windows** | **Mac(OS X)** |
| --- | --- | --- |
| 显示最近打开的文件记录列表 | Ctrl + E | ⌘E |
| 显示最近修改的文件记录列表 | Ctrl + Shift + E | ⌘ ⇧E |
| 查看最近的变更记录 | Alt + Shift + C | ⌥⇧C |

#### 3\. 跳转回退

| **操作** | **Windows** | **Mac(OS X)** |
| --- | --- | --- |
| 退回 / 前进到上一个操作的地方(windows有可能与系统快捷键翻转屏幕冲突，需要修改：**桌面右键->图形选项->选项和支持，将旋转屏幕的几个快捷键修改即可**) | Ctrl + Alt + 方向左键/方向右键 | ⌘⌥← / ⌘⌥→ |
| 跳转到最后一次编辑的地方 | Ctrl + Shift + BackSpace | ⌘⇧⌫ |

#### 4\. 面板切换

| **操作** | **Windows** | **Mac(OS X)** |
| --- | --- | --- |
| 左右切换打开的编辑tab页 | Ctrl + ← / Ctrl + → | 未知 |
| 显示所有的编辑tab页 | Ctrl + tab | ⌃⇥ |
| 返回到前一个工具窗口 | F12 | F12 |

#### 5\. 标签与收藏夹

| **操作** | **Windows** | **Mac(OS X)** |
| --- | --- | --- |
| 选中文件/文件夹，使用助记符设定/取消书签 | Ctrl + F11 | ⌘F11 (Fn) |
| 直接设置数字标签 | Ctrl + Shift + 1,2,3...9 | ⌃⇧**1,2,3...9** |
| 定位到对应数值的书签位置 | Ctrl + 1,2,3...9 | ⌃**1,2,3...9** |
| 添加到收藏夹 | Alt + Shift + F | ⌥⇧F |
| 查看已经设置的标签与收藏夹(Favorites面板--Bookmarks中可以查看) | Alt + 2(Favorites面板) | ⌘2(Favorites面板) |
| 删除favorites、Bookmarks | 在Favorites面板中，选中要删除的对象，按delete | 在Favorites面板中，选中要删除的对象，按⌫ |

五. 重构
-----

| **操作** | **Windows** | **Mac(OS X)** |
| --- | --- | --- |
| 复制文件到指定目录 | F5 | F5 |
| 移动文件到指定目录 | F6 | F6 |
| 安全重命名文件、变量等 | Shift + F6 | ⇧F6 |
| 更改签名 | Ctrl + F6 | ⌘F6 |
| 将选中的代码提取为方法 | Ctrl + Alt + M | ⌘⌥M |
| 提取变量 | Ctrl + Alt + V | ⌘⌥V |
| 提取字段 | Ctrl + Alt + F | ⌘⌥F |
| 提取常量 | Ctrl + Alt + C | ⌘⌥C |
| 提取参数 | Ctrl + Alt + P | ⌘⌥P |

六.  调试
------

| **操作** | **Windows** | **Mac(OS X)** |
| --- | --- | --- |
| 进入下一步，如果当前行断点是一个方法，则不进入当前方法体内 | F8 | F8(Fn) |
| 进入下一步，如果当前行断点是一个方法，则进入当前方法体内， 如果该方法体还有方法，则不会进入该内嵌的方法中 | F7 | F7(Fn) |
| 智能步入，断点所在行上有多个方法调用，会弹出进入哪个方法 | Shift + F7 | ⇧F7 (Fn) |
| 智能跳出 | Shift + F8 | ⇧F8 (Fn) |
| 恢复程序运行，如果该断点下面代码还有断点则停在下一个断点上 | F9 | F9(Fn) |
| 运行到光标处，如果光标前有其他断点会进入到该断点 | Alt + F9 | ⌥F9(Fn) |
| 计算表达式（可以更改变量值使其生效） | Alt + F8 | ⌥F8 (Fn) |
| 切换断点（若光标当前行有断点则取消断点，没有则加上断点） | Ctrl + F8 | ⌘F8 (Fn) |
| 查看断点信息 | Ctrl + Shift + F8 | ⌘⇧F8 (Fn) |

七. 系统功能
-------

| **操作** | **Windows** | **Mac(OS X)** |
| --- | --- | --- |
| 打开相应编号的工具窗口 | Alt + 1...9 | ⌘1...⌘9 |
| 切换全屏模式 | 未知 | ⌃⌘F |
| 切换最大化编辑器 | 双击tab全屏 | ⌘⇧F12/双击tab全屏 |
| 检查当前文件与当前的配置文件 | Alt + Shift + I | ⌥⇧I |
| 快速切换当前的scheme（切换主题、代码样式等） | 未知 | ⌃` |
| 打开IDEA系统设置 | Ctrl + Alt + S | ⌘, |
| 打开项目结构对话框 | Ctrl + Alt + Shift + S | ⌘; |
| 关闭活动run/messages/find/... tab | 未知 | ⌘⇧F4 |

八. 代码版本管理
---------

| **操作** | **Windows** | **Mac(OS X)** |
| --- | --- | --- |
| 提交代码到版本控制器 | Ctrl + K | ⌘K |
| 从版本控制器更新代码 | Ctrl + T | ⌘T |

九. 快捷键查看工具
----------

#### 1\. 查看某特定快捷键的具体功能

使用IDEA自带的工具: `Setting(Windows快捷键Ctrl+Alt+S)` --\> `Keymap` --\> `Find Shortcut` --\> 按入快捷键，即可筛选出快捷键对应的功能。

如下图：

![](https://static.didispace.com/images3/cb7a393d0a55aeca2ac9c65adb3a0e58.png)

#### 2\. 查看某功能对应的快捷键

通过安装使用IDEA插件：**Key Promoter X**来实现查找功能

安装方式：`Settings` --\> `plugins` --\> `Marketplace`，搜索`Key Promoter X`并安装

![](https://static.didispace.com/images3/b51b91f1e12674f4675b779253a0f5e3.png)

使用方式：安装并重启激活插件后，每当点击IDEA中各个按钮、功能时，如果此功能存在对应的快捷键，Key Promoter X在IDEA右下角都会提示此快捷键；

如果没有，则可能会提示可以设置相应的快捷键操作

![](https://static.didispace.com/images3/a2422c5dc32414489ea223addec13991.png)

也可以通过打开右侧Key Promoter X面板查看曾经使用和提醒过的功能对应的快捷键

![](https://static.didispace.com/images3/5a50bf34bd9e4571e1f49dafeec53918.png)

> 本文转载自：https://blog.csdn.net/u010086122/article/details/100122085

好了，今天的分享就到这里，如果这个小技巧对你有用，那就帮忙点赞、在看、分享、关注，四连支持一下吧！

如果你觉得这个系列还不错，可以关注我在连载的这个专栏：[玩转IntelliJ IDEA](https://www.didispace.com/idea-tips/)，分享各种使用技巧与好用插件！