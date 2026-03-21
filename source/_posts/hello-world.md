---
title: 日常创建博客到部署的流程
---
Welcome to [Hexo](https://hexo.io/)! This is your very first post. Check [documentation](https://hexo.io/docs/) for more info. If you get any problems when using Hexo, you can find the answer in [troubleshooting](https://hexo.io/docs/troubleshooting.html) or you can ask me on [GitHub](https://github.com/hexojs/hexo/issues).

## Quick Start

![alt text](image.png)

### Step0 clean 

清除缓存，不一定需要。当页面不更新时可以先运行这个命令

``` bash
$ hexo clean
```

### Step1 Create a new post

创建了一个名为My New Post.md的文件
``` bash
$ hexo new "My New Post"
```

More info: [Writing](https://hexo.io/docs/writing.html)


### Step2 Generate static files

生成一个静态页面资源
``` bash
$ hexo g
```

More info: [Generating](https://hexo.io/docs/generating.html)


### Step3 Run server

部署到本地
``` bash
$ hexo s
```

More info: [Server](https://hexo.io/docs/server.html)


### Step4 Deploy to remote sites

如果在本地运行没什么问题后，部署到远程服务器上
``` bash
$ hexo d
```

More info: [Deployment](https://hexo.io/docs/one-command-deployment.html)
