---
title: "小工具导航页"
date: 2022-05-01T22:37:00+08:00
lastmod: 2022-05-01T22:37:00+08:00
draft: false
tags: ["Vue", "玩具"]
categories: ["玩具"]

---

上班的时候经常需要使用工具网站，公司环境不方便登录自己的账号进行云同步。
所以我简单的糊了一个 Vue 的小工具导航页。

![成品](https://blog.zjyl1994.com/post/superutils/su.png)

<!--more-->

# 开发

构想是一个简简单单的页面，可以通过名称进行搜索。对于纯前端筛选，Vue 无疑是最合适的选择，

最基本的页面结构是这样：

```html
<div id="app">
    <input id="search-box" type="text" v-model="keyword" placeholder="Type keyword to search on name field">
    <br>
    <table>
        <tr>
            <th>Name</th>
            <th>URL</th>
        </tr>
        <tr v-for="item in searchInList()">
            <td>{{item.name}}</td>
            <td><a v-bind:href="item.url" target="_blank">{{item.url}}</a></td>
        </tr>
    </table>
</div>
```

然后，搭配简单的 Vue 代码

```javascript
Vue.createApp({
    data() {
        return {
            items: [],
            keyword: ""
        }
    },
    mounted() {
        var self = this;
        fetch('data.json').then(response => response.json()).then(data => {
            self.items = data.sort(function (x, y) {
                if (x.name.length < y.name.length) {return -1;}
                if (x.name.length > y.name.length) {return 1;}
                return 0;
            });
        });
    },
    methods: {
        searchInList() {return this.items.filter(item => item.name.toLowerCase().indexOf(this.keyword.toLowerCase()) != -1)}
    }
}).mount('#app');
```

简单解释一下这些 Vue 在做什么：

- 启动的时候，fetch拉取`data.json`，下载后读取数据进行按长度排序。
- 方法：searchInList，过滤包含 keyword 的条目（全转小写，方便搜索）。 

# 部署

由于代码是托管在自己的Gitea服务器上，没法使用传统的CI/CD方案，所以找到个简单的
Webhook方案进行触发。（本想自己写，但是觉得没必要）

这里使用的方案是：https://github.com/adnanh/webhook

Debian 上直接 `apt install webhook` 可以安装。

默认端口 9000，自己修改systemd单元的Exec段增加 `-port` 切换到闲置端口。

简单列出一个我使用的配置文档

```yaml
- id: deploysu
  execute-command: "/opt/scripts/deploysu.sh"
  command-working-directory: "/opt/scripts"
  trigger-rule:
    match:
      type: value
      value: refs/heads/master
      parameter:
        source: payload
        name: ref
```

写好后启动webhook服务，此时访问你的 `http://server:9000/hooks/deploysu` 就会调用你设定的
`/opt/scripts/deploysu.sh`。

再写一个简简单单的部署脚本。

```bash
#!/usr/bin/env bash
rm -rf yp
git clone ssh://git@server:22/zjyl1994/yellow_page.git yp
cd yp
cp index.html /var/www/su.zjyl1994.com/index.html
cp data.json /var/www/su.zjyl1994.com/data.json
```

最后去 Gitea 配置你的 Webhook

![Gitea](https://blog.zjyl1994.com/post/superutils/webhook.png)

修改后 push 到 master 分支就可以自动更新服务端的部署了。