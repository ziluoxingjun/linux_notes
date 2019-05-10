### Github 上 fork 项目后与原项目保持同步

方法是在本地建立两个库的中介，把两个远程库都 clone 到本地，然后拉取原项目更新到本地，合并更新，最后 push 到你的 github。

假设：
来源为： https://github.com/_original/_project.git
fork 项目为： https://github.com/_your/_project.git

1、准备一个本地目录，并克隆自己 fork 的项目到本地。
```bash
$ git clone https://github.com/_your/_project.git
```

2、进到 _project 目录下，查看远程仓库
```bash
$ git remote -v
```

3、增加远程分支(fork 的分支)，名为 update_linux（名字任意），clone 原项目到仓库
```bash
$ git remote add update_linux https://github.com/_original/_project.git
```

4、查看远程分支，多出一个 update_linux
```bash
$ git remote -v
```

5、把远程原始分支 update_linux 的代码拉到本地  
```bash
$ git fetch update_linux
```

6、查看分支，一个本地分支 master，三个远程分支，红色的就是要合并的 merge
```bash
$ git branch -av
```

7、合并远程分支 update_linux 的代码
```bash
$ git merge update_linux/master
```

8、最后把最新的代码推送到你的 github 上
```bash
$ git push -u origin master
$ git push origin +master
# 强制 push
```
