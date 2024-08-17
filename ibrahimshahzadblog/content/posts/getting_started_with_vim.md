---
title: "Set up your vim"
date: 2021-06-11 18:30:00
image: ./code.jpg
banner: ./codebanner.jpg
description: Set up your favourite editor -- VIM --
---

Shruberry!

## Vimrc

Get a good vimrc file, you can use mine

```bash
cd ~
wget https://raw.githubusercontent.com/IbrahimShahzad/My-Vimrc-file/master/.vimrc ~/.vimrc
```

## Get VIm (version > 8)

### CentOS

- Install the Dependencies and clone vim (version > 8.1 )

```bash
sudo yum install -y gcc make ncurses ncurses-devel git python3 pip3
sudo yum list installed | grep vim
sudo yum remove -y vim-enhanced vim-common vim-filesystem
sudo git clone https://github.com/vim/vim.git

```

### Ubuntu

```bash
sudo apt install -y gcc make libncurses5-dev libncursesw5-dev git python3 pip3 
sudo git clone https://github.com/vim/vim.git

```

## Install Vim

> Following should work for both CentOS and Ubuntu


- You can configure with simple options with following

```bash
pushd vim
./configure --with-features=huge \
  --enable-multibyte \
  --enable-pythoninterp \
  --enable-luainterp
sudo make
sudo make install
popd

```
**OR**
```bash
pushd vim
./configure --with-features=huge \
            --enable-multibyte \
	    	--enable-rubyinterp=yes \
	    	--enable-pythoninterp=yes \
	    	--with-python-config-dir=/usr/lib/python2.7/config \ # pay attention here check directory correct
	    	--enable-python3interp=yes \
	    	--with-python3-config-dir=/usr/lib/python3.5/config \
	    	--enable-perlinterp=yes \
	    	--enable-luainterp=yes \
            --enable-gui=gtk2 \
            --enable-cscope \
	   		--prefix=/usr/local
popd

```

> On `CentOS7` change `--with-python-config-dir=/usr/lib/python2.7/config` to `--with-python-config-dir=/lib64/python2.7/config`


- Finally check Vim Version

```bash
vim -version | less
```

## Get Vundle

> Following should work for both CentOS and Ubuntu

- Download vunlde and install all the plugins mentioned in `~.vimrc` file.

```bash
git clone https://github.com/VundleVim/Vundle.vim.git ~/.vim/bundle/Vundle.vim
vim +PluginInstall +qall
```

- The above should download all the plugins in `/.vim/bundle/` directory.

## For CLANG support

### CentOS

- Run the following

```bash
sudo yum install cmake -y
sudo yum install centos-release-scl -y
sudo yum install llvm-toolset-7 -y
scl enable llvm-toolset-7 bash
clang --version
```

### Ubuntu

- Run the following

```bash
sudo apt-get install cmake clang -y
clang --version
```

## Install YOUCOMPLETEME for autocompletion

- Run the following commands

```bash
pushd ~/.vim/bundle/YouCompleteMe/
git submodule update --init --recursive
./install.py --clang-completer
popd

```
## Done

Finally
- Run the `vim` command.
- Enjoy the `vim` in all its glory
- Checkout the plugins in `~/.vimrc` file
- Need a cheatsheet for vim commands? Look [<font color="blue">here</font>](https://www.cs.cmu.edu/~15131/f17/topics/vim/vim-cheatsheet.pdf)
- Give yourself a cookie
 
![VIM](./vim.png)