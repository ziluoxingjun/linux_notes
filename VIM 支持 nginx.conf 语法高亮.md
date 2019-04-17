### 手动修改方法

#### 下载 Nginx 配置文件的语法文件:nginx.vim
```bash
$ wget https://www.vim.org/scripts/download_script.php?src_id=19394 -O nginx.vim
```

#### 复制文件到 syntax 目录
```bash
$ mv nginx.vim /usr/share/vim/vim74/syntax
$ mv nginx.vim ~/.vim/syntax/ // 单用户目录
```

#### 修改 filetype.vim 文件，增加一行
```bash
$ vim /usr/share/vim/vim74/filetype.vim
$ vim ~/.vim/filetype.vim
au BufRead,BufNewFile /etc/nginx/*,/usr/local/nginx/conf/* if &ft == '' | setfiletype nginx | endif
```
> 根据自己安装的 Nginx 目录修改配置

### 自动化脚本
```
$ vim nginxSyntaxHightlight.sh
#!/bin/bash
mkdir -p ~/.vim/syntax && cd ~/.vim/syntax
wget http://www.vim.org/scripts/download_script.php?src_id=19394 -O nginx.vim >/dev/null
echo "au BufRead,BufNewFile /usr/local/nginx/conf/* set ft=nginx" > ~/.vim/filetype.vim
# 其中路径/usr/local/nginx/conf/* 为你的 nginx.conf 文件路径

$ sh nginxSyntaxHightlight.sh
```
