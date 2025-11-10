---
title: "博客自动部署系统Shell版"
date: 2018-01-25T11:28:00+08:00
lastmod: 2018-01-25T11:28:00+08:00
draft: false
---

这是用bash脚本+Crontab结合的自动部署，只依赖md5deep包。

建议设置2-3分钟一次，以免影响性能。

<!--more-->

```bash
#!/bin/bash
RESULT=`md5deep -x /tmp/blogs.md5 -r /path/to/blog/root/ | wc -l`
if [ $RESULT -ne 0 ]; then
    hugo -s /path/to/blog/root/ -d /tmp/blog
    rsync -avz --delete /tmp/blog/ rsync@example.com:/path/to/you/www/root
    rm -rf /tmp/blog
    md5deep -lr /path/to/blog/root/ > /tmp/blogs.md5
fi

```