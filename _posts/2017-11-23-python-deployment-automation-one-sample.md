---
layout:    post
title:     "Python项目自动化部署之一：举个栗子"
date:      "2017-11-23 16:49:57"
permalink: /python-deployment-automation-one-sample
---

本文主要讲述一下我司
（[一个成长中的创业公司][zaihui-intro]）
目前的代码发布流程用到了哪些工具。


<!--MORE-->


## 发布工具：[Jenkins][jenkins]

我们用的发布工具是[很多公司都在用][jenkins-stackshare]的[Jenkins][jenkins]。
举个~~栗子~~图，
在Jenkins Server上可以一键发布后端服务器代码：

![jenkins-demo][jenkins-demo]

按下 Build按钮 以后，
发生的事情如下：

* 在 Jenkins服务器 上触发预先配置的 Bash脚本
  * git命令获取到最新的代码版本，切换合适的分支
  * ~~执行代码风格检测和单元测试~~自从使用了付费版GitLab后，本功能已切换至GitLab CI了
  * 安全检查通过以后，使用fab命令部署代码


## 发布命令：[Fabric][fabric]

这里的fab命令用的就是[Python的Fabric库][fabric]，
这个库类似[ansible][ansible]，
主要包含两套功能：

* **本地命令集成**。
这点大概跟 Java 的 [`ant`][ant], [`gradle`][gradle],
或者是 JS 的 [`npm run`][npm] 有类似功能。
都是可以把数个操作集成到一条简单的工作流命令里。

* **远程ssh工具**。
[Fabric][fabric]里基于[ssh][ssh]，
实现了一套方便的远程命令接口，
比如这么一段代码就可以把配置上传到远程服务器：

```
from fabric.api import *  # NOQA

# 不用试了，这里的两个都是假的domain，对应放上ssh的host/user即可
env.hosts = ['www.kezaihui.com', 'zaihuiwebserver-814613977.cn-north-1.elb.amazonaws.com.cn']
env.user = 'saber'

def update_supervisor_config():
    put('./supervisor/*.conf', '/etc/supervisor/conf.d/', use_sudo=True)
    run('supervisorctl update', use_sudo=True)
```

但[Fabric][fabric]有个比较蛋疼的地方就是它只支持[Python2][which-python]，
假如要用[Python3][which-python]的话，
可以使用[Fabric][fabric]的一个fork分支[Fabric3][fabric3]，
[Fabric3][fabric3]与[Fabric][fabric]大部分功能等价。

只想用里面本地命令集成这部分功能的话，
还有一个库叫[Invoke][invoke]也提供了类似的功能。
这个库主要是名字特别帅，
[dota2里面的卡尔就叫Invoker（祈求者）][invoker]。


## 进程管理：[Supervisor][supervisor]

在正式环境中，
为了保证服务器进程的鲁棒性，
我们使用了 [supervisor][supervisor] 来监控进程状态。

一个简单的 nginx supervisor 的配置会长这样子：

```
[program:nginx]
command=/usr/sbin/nginx
autostart=true
autorestart=true
stdout_logfile=/var/log/supervisor/nginx.log
stderr_logfile=/var/log/supervisor/nginx_error.log
```

把配置文件放到 `/etc/supervisor/conf.d/nginx.conf` 以后，
就可以使用一系列命令把服务起起来：

```
$ supervisorctl update  # supervisorctl 是 supervisor 的命令行工具，更新一波配置
nginx    STARTING    pid 1000, uptime 0:00:00

$ supervisorctl status  # 查看进程状态
nginx    RUNNING     pid 1000, uptime 0:12:34

$ kill -9 1000  # 模拟各种波动，干掉 nginx 进程

$ supervisorctl status  # 再次查看进程状态，可以发现 supervisor 自动重启了
nginx    STARTING    pid 1020, uptime 0:00:00
```

## 总结

负责发版的工程师，
可能只在页面上点下了 `Build` 的一个按钮，
实际上的流程是这样的：

* Jenkins 触发了配置好的 Bash脚本。
* 里面 Bash 脚本跑了 fab 命令。
* fab 命令执行了代码上传的工作，本质上是通过 ssh 执行命令。
* 最终用 supervisor 开启/重启了进程服务。
* 发版完成。

以上大概就是我司目前自动化部署的简陋介绍。
升级之路漫漫，
还是有很多东西要学习/实践/掌握的呀。

> [原文链接][self]，[作者 @苏子岳][about-me]
>
> 本文版权属于再惠研发团队，欢迎转载，转载请保留出处。

[zaihui-intro]: https://www.zhihu.com/question/19596230/answer/152193862
[jenkins-stackshare]: https://stackshare.io/jenkins
[jenkins]: https://jenkins.io/
[jenkins-demo]: /assets/pics/zaihui_jenkins.jpg
[fabric]: https://github.com/fabric/fabric
[ansible]: https://github.com/ansible/ansible
[ant]: http://ant.apache.org/
[gradle]: https://gradle.org/
[npm]: https://www.npmjs.com/
[ssh]: https://en.wikipedia.org/wiki/Secure_Shell
[which-python]: http://docs.python-guide.org/en/latest/starting/which-python/
[fabric3]: https://github.com/mathiasertl/fabric/
[invoke]: http://www.pyinvoke.org/
[invoker]: https://dota2.gamepedia.com/Invoker
[supervisor]: http://supervisord.org/
[self]: /python-deployment-automation-one-sample
[about-me]: http://www.liriansu.com/about/
