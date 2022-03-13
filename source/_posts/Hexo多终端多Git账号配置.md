---
title: Hexo多终端多Git账号配置
date: 2022-03-13 17:20:51
tags:
---

# Hexo多终端多Git账号配置

# 1. Hexo多终端方案

## 1.1 解决思路

本质的思路就是在对应的Repository下（例如我的是：[mayflygame.github.io](http://mayflygame.github.io/)）维护master和一个branch（branch的名字可以命名为hexo）

- 主干：master，用于存放hexo生成的所有静态页面，即你要展示的网站页面。
- 分支：hexo（设置为default分支），存放的就是hexo对应的所有文件，例如_config.yml，package.json等文件，source, themes, scaffolds等文件夹。

## 1.2 如何使用？

使用的时候，新建markdown文件，编辑文章。

```
$ hexo new "new blog"
```

在本地对博客进行修改（添加新博文、修改样式等等）后，通过下面的流程进行管理。

```
$ git add .
$ git commit -m "..."
$ git push origin hexo #指令将改动推送到GitHub(此时当前分支应为hexo)
```

最后执行

```
$ hexo g -d # 将hexo生成的静态页面发布到master上。
```

# 2. 多Github账号问题

默认通过hexo g -d能够直接发布到对应的git仓库，但是如果电脑上配置了两个git账号。就会出现问题：

例如需要你输入用户名/密码，但是又回因为安全问题等报错，如下图。

```jsx
remote: Support for password authentication was removed on August 13, 2021. Please use a personal access token instead.
remote: Please see https://github.blog/2020-12-15-token-authentication-requirements-for-git-operations/ for more information.
fatal: Authentication failed for 'https://github.com/MayFlyGame/mayflygame.github.io/'
FATAL {
  err: Error: Spawn failed
      at ChildProcess.<anonymous> (/Users/godwit/Blog/mayflygame.github.io/node_modules/hexo-util/lib/spawn.js:51:21)
      at ChildProcess.emit (node:events:520:28)
      at Process.ChildProcess._handle.onexit (node:internal/child_process:291:12) {
    code: 128
  }
} Something's wrong. Maybe you can find the solution here: %s https://hexo.io/docs/troubleshooting.html
```

可以考虑通过设置token来解决。

这里提供个简单的解决方案，hexo的_config.yml文件，在配置repo的时候，参考如下配置

```jsx
103 # Deployment
104 ## Docs: https://hexo.io/docs/one-command-deployment
105 deploy:
106   type: git
107   # repo：git@用户名.github.com:用户名/xxx.github.io.git
108   repo: git@mayflygame.github.com:MayFlyGame/mayflygame.github.io.git
109   branch: master
110   name: MayflyGame
111   email: mayflygame@foxmail.com
```

以上。