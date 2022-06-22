

# 1、安装

下载地址：https://git-scm.com/download/

安装完成后，在开始菜单里找到“Git”->“Git Bash”，蹦出一个类似命令行窗口的东西，就说明Git安装成功！

安装完成后，还需要最后一步设置，在命令行输入：

```bash
$ git config --global user.name "Your Name"
$ git config --global user.email "email@example.com"
```

注意`git config`命令的`--global`参数用了这个参数，表示你这台机器上所有的Git仓库都会使用这个配置，当然也可以对某个仓库指定不同的用户名和Email地址。

# 2、创建版本库

在一个合适的位置新建一个文件夹，进入文件夹后

```bash
git init
```

就可以建立好仓库，而且会告诉你是一个空仓库

如果你使用Windows系统，为了避免遇到各种莫名其妙的问题，请确保目录名（包括父目录）不包含中文。

# 3、连接远程gitlab\github仓库

两种方式：

远程克隆下来：

```bash
git clone git@gitlab.hengtonggroup.com.cn:zhangjs/test.git
cd test
touch README.md
git add README.md
git commit -m "add README"
git push -u origin master
```

本地新建一个目录再远程连接远程仓库：

```shell
cd existing_folder
git init
git remote add origin git@gitlab.hengtonggroup.com.cn:zhangjs/test.git
git add .
git commit -m "Initial commit"
git push -u origin master
```

# 4、git的使用命令

```bash
git add filename   #添加某个文件
git --all .    #添加当前目录下所有未添加的文件
git commit -m "注释"  #提交，并添加注释
git status #查看仓库当前的状态
git diff  #查看当前仓库的difference
git log   #查看操作历史记录
git log --pretty=oneline  #查看操作历史记录并过滤信息
git reset --hard HEAD^    #回退到上一个版本
git reset --hard 1094a    #回退到指定版本
git reflog #历史使用命令
git pull    #拉取远程仓库
git checkout -b dev  #创建dev分支，-b参数表示创建并切换，相当于：
git branch dev
git checkout dev
git checkout -- <filename>    #撤销修改
git branch   #查看当前分支
git merge    #用户合并指定分支到当前分支，注意到上面的其中的Fast-forward信息，Git告诉我们，这次合并是“快进模式”，也就是直接把master指向dev的当前提交，所以合并速度非常快
git branch -d dev  #删除dev分支
git switch -c dev  #创建并切换到新的dev分支，新版本推荐使用switch切换，以免和checkout混淆

```

