---
title: "博客自动部署系统"
date: 2018-01-25T01:32:00+08:00
lastmod: 2018-01-25T01:32:00+08:00
draft: false
tags: ["Hugo", "博客"]
categories: ["博客"]

---

静态网站生成器是一个很好用的东西，问题就是他没有控制面板，就会带来一个问题。

你每次写完文章都要手动去ssh里build一次网站，对于能懒就懒的我来说，这么弄不行。

传统的实现方式是Git+Hook，这种就需要用Git，很不方便。
我选择的方案是 Nextcloud 直接编辑文件，inofity监听文件变化进行更新。

<!--more-->

```python
#!/usr/bin/env python3
import time
import os
from watchdog.observers import Observer
from watchdog.events import FileSystemEventHandler

lastmodify = 0

class Watcher:
    DIRECTORY_TO_WATCH = "/path/to/folder/blogs/"
    DELAY_SECOND = 5
    COMMAND_RUN = "/path/to/script/blog.sh"
    def __init__(self):
        self.observer = Observer()
    def run(self):
        global lastmodify
        event_handler = Handler()
        self.observer.schedule(event_handler, self.DIRECTORY_TO_WATCH, recursive=True)
        self.observer.start()
        try:
            while True:
                if lastmodify != 0 :
                    if (lastmodify + self.DELAY_SECOND) < time.time():
                        os.system(self.COMMAND_RUN)
                        lastmodify = 0
                time.sleep(5)
        except:
            self.observer.stop()

        self.observer.join()


class Handler(FileSystemEventHandler):
    @staticmethod
    def on_any_event(event):
        global lastmodify
        lastmodify = time.time()

if __name__ == '__main__':
    w = Watcher()
    w.run()

```