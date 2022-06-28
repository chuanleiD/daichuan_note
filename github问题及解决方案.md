### 在执行小组项目中，出现了贡献无法正常显示的问题

无法正常显示贡献的解决方法

#### 问题原因：

一般是因为`push`的发起者与您的github账户邮箱不能匹配。

请首先保证github账户邮箱设置正确，不要设置隐藏。

请着重关注：本地设置邮箱时是否添加了双引号即

应该为：xxxxxxxxxx@qq.com

而不是：“xxxxxxxxxx@qq.com”

#### 解决问题：

1.点击github账户的`setting`、`Emails`确定邮箱设置

<img src="https://i0.hdslb.com/bfs/album/8035e9877d81b24c4316c0633d0f3476610d8600.png" alt="image-20220518092047825" style="zoom:80%;" /> 

2.修改本地账户信息

```
git config --global --list   // 查看当前git的配置信息
git config user.name
git config user.email
```

```
git config --global user.name username
git config --global user.email email
```

做好这两步后，之后的commit操作应该可以正常进行了

#### 如何补救

请注意，该方法只应该用于紧急场合，不应该随意使用。官方英文教程如下：

[Why are my contributions not showing up on my profile? - GitHub Docs](https://docs.github.com/cn/account-and-profile/setting-up-and-managing-your-github-profile/managing-contribution-graphs-on-your-profile/why-are-my-contributions-not-showing-up-on-my-profile)

步骤：

1.在新文件夹clone：`git clone --bare 克隆要修改的项目地址`

2.`cd 项目目录`

3.拷贝如下脚本，修改其中信息，之后回车

```
#!/bin/sh
git filter-branch --env-filter '
OLD_EMAIL=你的旧邮箱
CORRECT_NAME=你的名字
CORRECT_EMAIL=你的邮箱

if [ "$GIT_COMMITTER_EMAIL" = "$OLD_EMAIL" ]
then
export GIT_COMMITTER_NAME="$CORRECT_NAME"
export GIT_COMMITTER_EMAIL="$CORRECT_EMAIL"
fi
if [ "$GIT_AUTHOR_EMAIL" = "$OLD_EMAIL" ]
then
export GIT_AUTHOR_NAME="$CORRECT_NAME"
export GIT_AUTHOR_EMAIL="$CORRECT_EMAIL"
fi

' --tag-name-filter cat -- --branches --tags
```

4.此时输入`git log`可以查看分支信息是否已经修改成功

5.本地修改成功后，输入命令如下即可成功

`git push --force --tags origin 'refs/heads/*'`

6.本地仓库可以删除掉了。代码仓库可能需要重新clone一下



## 解决 Failed to connect to github.com port 443:connection timed out

可能是因为使用代理（例如翻墙软件）之后出错

先开启代理

```
git config --global http.proxy http://127.0.0.1:1080
git config --global https.proxy http://127.0.0.1:1080
```

后关闭代理

```
取消全局代理：
git config --global --unset http.proxy
git config --global --unset https.proxy
```

