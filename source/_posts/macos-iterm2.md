---
title: macOS iTerm2 开发环境配置
date: 2020-12-25 21:09:12
index_img: https://riger.oss-cn-shanghai.aliyuncs.com/img/image-20201214140102118.png
categories: 
- tools
tags: 
- iTerm2
- macOS
- zsh
---

# macOS iTerm2 开发环境配置

![image-20201214140102118](https://riger.oss-cn-shanghai.aliyuncs.com/img/image-20201214140102118.png)

> 注意：本文很多应用的安装都是走的Github，建议在科学上网环境中进行。

## 1. iTerm2 安装

放弃使用系统自带Terminl， 在 [iTerm2官网](https://iterm2.com/) 下载安装

<div align="center"><img src="https://riger.oss-cn-shanghai.aliyuncs.com/img/image-20201214135558722.png" alt="image-20201214135558722" style="zoom:50%;" /></div>

安装后打开，按上图，点击`Make iTerm2 Default Term`设置为默认终端

## 2. 安装Homebrew

[`Homebrew`](https://brew.sh/index_zh-cn)自称是`MacOS`或`Linux`缺失的软件包管理器，使用它可以一行命令安装各种开发类工具及应用程序

```shell
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install.sh)"
```

## 3. 安装zsh

`MacOS`自带`zsh`，不过版本比较老，我们使用`homebrew`安装最新版，并修改默认`shell`为自己下载的`zsh`

```shell
brew install zsh
echo "/usr/local/bin/zsh" | sudo tee -a /etc/shells
chsh -s /usr/local/bin/zsh
```

## 4. 安装ohmyzsh

运行下面的命名即可安装。 [ohmyzsh官网](https://github.com/ohmyzsh/ohmyzsh)也可以去看看各种插件和主题。

```shell
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```

`zsh`的配置文件在：`~/.zshrc`,后面的配置都在这里面，这里假设你会使用`vim`，不会可自行使用`Sublime`等编辑器。

## 5. 字体主题配置

虽然主题这个东西是一个比较个性化的东西，但是我还是要隆重推荐一下`Grovbox`主题，我 的`IDE`、`iTerm` 和 `Vim`都用的这个主题:

Github 下载 iTerm2 的 [gruvbox-material](https://github.com/AmmarCodes/gruvbox-material-iterm2) 主题, 双击安装

打开设置, Profiles, Default, Colors, 选择 `grovbox-material`

安装Fira字体, 及其对应Nerd版(配合devicons使用)

```shell
brew tap homebrew/cask-fonts
// 这个是给idea, vscode之类用
brew cask install font-fira-code
// iTerm2 vim 专用
brew cask install font-fira-code-nerd-font
```

打开`iTerm2`设置, Profiles, Default, Font,  选择 `Fira Code Nerd`

参考ohmyzsh提供的这些 [主题](https://github.com/ohmyzsh/ohmyzsh/wiki/themes)可以选一个自己喜欢的, 推荐agnoster

```shell
cd ~/Downloads
git clone https://github.com/fcamblor/oh-my-zsh-agnoster-fcamblor.git
cd oh-my-zsh-agnoster-fcamblor/
./install
vi .zshrc
ZSH_THEME="agnoster"
```

去掉路径中的用户名，设置`DEFAULT_USER`和你的用户名一直即可:

```shell
# .zshrc 里增加：
DEFAULT_USER="riger"
```

## 6. 插件安装及配置

安装一些好用的插件:

```shell
brew install autojump
git clone git://github.com/zsh-users/zsh-autosuggestions $ZSH_CUSTOM/plugins/zsh-autosuggestions
git clone git://github.com/zsh-users/zsh-syntax-highlighting $ZSH_CUSTOM/plugins/zsh-syntax-highlighting

vi .zshrc
# 修改如下
plugins=(
    git 
    sublime
    zsh-autosuggestions
    autojump
    zsh-syntax-highlighting
)
```

简单说明一下我开启的这些插件：

- `git`: 默认就开启的，可以看到文章第一张图，路径上即可显示当前分支是在`Master`，还有很多方便使用的`alias`

我常用的有：

```shell
ga='git add'
gst='git status'
gcmsg='git commit -m'
glola='git log --graph --pretty='\''%Cred%h%Creset -%C(auto)%d%Creset %s %Cgreen(%cr) %C(bold blue)<%an>%Creset'\'' --all'
# 命令行输入 alias 即可看到所有的alias
```

- `sublime`：`st 文件或路径`  使用`sublime`打开文件或路径；`stt` 打开当前路径

- `zsh-autosuggestions`：命令提示，用过的命令，按方向→自动完成

![image-20201227231810507](https://riger.oss-cn-shanghai.aliyuncs.com/img/image-20201227231810507.png)

- `autojump`: `j + 关键字` 进入你最近去过的含关键字的路径，比如我进过`jdk`的路径，以后只要输入：

```sh
j jdk
```

就可以回到`jdk`路径，是不是很方便？

![image-20201227232454896](https://riger.oss-cn-shanghai.aliyuncs.com/img/image-20201227232454896.png)

- `zsh-syntax-highlighting`: `shell` 命令高亮，没有人能拒绝高亮。

![image-20201228101251109](https://riger.oss-cn-shanghai.aliyuncs.com/img/image-20201228101251109.png)

## 7. 其他

以后开发相关的配置都是放在`.zshrc`中的，比如`PATH`配置，一些脚本参数配置等。

还可以自行配置`Alias` 方便自己的使用，比如我配置的一些：

```shell
alias py3="python3"
alias openapk="open -a Finder app/build/outputs/apk"
alias releaseapp="./gradlew assemblerelease"
alias sqllogin="mysql -uroot -p123456"
alias redisstart="redis-server /usr/local/etc/redis.conf"
alias logcasher="pidcat net.worthtech.worthcasher"
alias showAct="adb shell dumpsys activity top | grep ACTIVITY"
alias starttom="~/Library/tomcat/bin/startup.sh"
alias stoptom="~/Library/tomcat/bin/shutdown.sh"
alias signapk="signtest app/build/outputs/apk/release/app-release-unsigned.apk"
alias jmeter="~/dev/apache-jmeter-5.3/bin/jmeter"
alias uninstallcasher="adb shell pm uninstall -k net.worthtech.worthcasher"
```

配置完后不要忘记让配置生效：

```shell
source .zshrc
```

Enjoy it !

