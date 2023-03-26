---
title: "AutoHotKey 模拟多媒体控制按键"
date: 2023-03-26T13:47:00+08:00
lastmod: 2023-03-26T13:47:00+08:00
draft: false
tags: ["AutoHotKey","小工具"]
categories: ["AutoHotKey"]

---

公司台式机标配的联想键盘属于非常中规中矩的那种，能打字，但是一点额外的媒体控制功能都没有。

当我们听着音乐的时候，如果同事过来问问题，不能及时暂停可是大问题。

所以借助大名鼎鼎的`AutoHotKey`可以绑定一组快捷键来模拟高级键盘上的多媒体控制功能。

热键定义脚本如下：

```
^!Space::Send {Media_Play_Pause}
^!Right::Send {Media_Next}
^!Left::Send {Media_Prev}
^!Up::Send {Volume_Up}
^!Down::Send {Volume_Down}
^!AppsKey::Send {Volume_Mute}
XButton1::Send {Media_Play_Pause}
XButton2::Send {Volume_Mute}
```

|  按键   | 功能  |
|  ----  | ----  |
| Ctrl+Alt+空格  | 播放/暂停 |
| Ctrl+Alt+方向键右  | 下一曲 |
| Ctrl+Alt+方向键左  | 上一曲 |
| Ctrl+Alt+方向键上  | 音量加 |
| Ctrl+Alt+方向键下  | 音量减 |
| Ctrl+Alt+菜单键  | 静音 |
| 鼠标侧键1  | 播放/暂停 |
| 鼠标侧键2  | 静音 |

至于为什么不使用音乐软件里的快捷键？
我大部分时间都是油管上找工作用BGM，浏览器可是不支持设置快捷键控制媒体的。

通过`AutoHotKey`触发的都是多媒体键盘上的标准功能按键，Windows上大部分软件都是吃这个设定的。起码Chrome/Firefox都支持。

如果想分享这个功能给其他人，通过`Ahk2exe`将ahk文件转换为exe就可以轻松的分发了。

亲测了一天，这个多媒体按键真的好用。
Windows 平台上的神级软件真是多，小时候就听说过AutoHotKey，现在真正的帮到了我，很赞。