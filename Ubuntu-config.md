最近一年发现重装机器系统的频率有点高，在最近在公司又重新装了一次之后，终于痛下决心打算记录下配置的踩坑经验。<br />​

本人是在实习的时候，后端当时全员 Linux Desktop，装了 Ubuntu 16.04 之后，一直稳定工作了大半年，算是初入 Linux，当时 Ubuntu 的稳定给本人留下极深的印象。回想起来倒不是觉得他们有极客精神，而是因为穷 QAQ，没有钱支撑全员 Mac。<br />​

首先先解释为啥要 Linux Desktop，撇开开源精神之类这些意识形态的东西，个人觉得最大的好处就是对比其他系统，Linux **硬件要求最低**和**运行效率最高**，还有不用考虑任何有关乱码或者 docker 这些形形色色的坑。<br />​

​<br />
<a name="M5YZE"></a>
## 发行版选择
​

再解释为啥是 Ubuntu，而不是 Arch 等其他发行版。<br />​

闲着无聊的时候，用笔记本发流行的发行版都折腾了一遍，Arch 当时给本人留下了不少的阴影就是容易挂滚，频率且挺高。而且个人很不喜欢这个发行版的粉丝，堪称邪教，暗搓搓的优越感让本人很不适，不知道为啥玩个发行版还会有鄙视链。Arch 教众言必称 AUR，KISS，滚动发行、时刻最新，虽然牛逼得不行，但是事实上没人会在公司里用 Arch 跑服务。有一说一，Arch wiki 确实可以看看，Arch 教徒里吹 wiki 这一点上算是没得黑的。<br />​

![image.png](https://cdn.nlark.com/yuque/0/2022/png/137440/1645850507474-9b1e81e5-39f0-4d35-9369-a3c56f0212db.png#clientId=ub1121d58-9461-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=666&id=u29721291&margin=%5Bobject%20Object%5D&name=image.png&originHeight=833&originWidth=658&originalType=binary&ratio=1&rotation=0&showTitle=false&size=867887&status=done&style=none&taskId=u83a4f3ac-3515-4663-a41d-15c03fa678b&title=&width=526.4)<br />​

个人是 Debian 粉。毕业前某个周末快速看了一遍 Debian 手册和 wiki 之后心里有种感觉，就是它了。个人是比较喜欢线上 Debian 的系统，本地 Ubuntu 的开发机器。Debian 系的发行版很稳定，且内核版本都很新。论真 · 开源精神 Debian 最纯正，坚持 GNU，没有商业公司驱动，都是社区用爱发电，且社区不弱于商业公司。Arch 粉丝可以说 Debian 本身不算好用，但是不能反驳他的通用性和社区，Debian 支持最多的架构，不管是树莓派这种板子，还是路由器，x86 服务器，还是个人电脑上。不管是 github 上开源的软件还是商业软件 Linux 这块基本都有 deb 包。选择了 Debian，就选择了最广大的社区。还有可以仔细观察下每年各种 Linux 杂七杂八的榜单，看一下 Debian 系的发行版占了多少。<br />​

选择 Ubuntu 是因为他比 Debian 多了很多驱动，对新手很友好，你遇到的问题在 ask ubuntu，stackoverflow 上都能搜到。如果不是很喜欢 GNOME，也可以尝试基于 Debian 的 KDE 的发行版，不再赘述。最后复读一下为什么选 Ubuntu，第一是基于 Debian，比 Centos 系内核要新的多，又比 Arch 相对稳定。对新手最友好，遇见问题在社区基本都能搜到答案。其次是对桌面来说比较省心，虽然臃肿，占用资源大，但是基本不用去额外安装什么。<br />​

推荐当前最新版本（以 20.04 为最新稳定发行版为例）的上一个版本，比如 18.04 是出到 18.04.5 以上了，优先选择 18.04 的最新版 18.04.6 。现在 20.04 出到了 20.04.4 了，个人是觉得一般发行版出到了 4 和 4 以上也可以尝试入坑了。<br />​<br />
<a name="x7IRf"></a>
## 开始
​

进入正题，安装 Ubuntu 遇到的一些问题和推荐的配置。首先个人是推荐不要过分折腾桌面的 UI 和一些美化的效果，这方面不是 Linux 真正的优点，相对来说没那么有价值，工作的时候接触过几个 Linux 高手，基本都是很原生朴素的桌面，不影响他们解决核心问题。当然如果对这方面特别有兴趣还是兴趣第一。<br />​

首先安装的时候启动盘可以选用 [ventoy](https://github.com/ventoy/Ventoy) 这种比较新且好用的开源工具。推荐安装的时候语言**选用英语**，选用中文的话，路径会变成中文，有时候会出现一些意想不到的 bug，可以安装完成后把语言环境设置成中文但是记得**不要把路径也改成中文**。安装的时候选用最 mini 模式的安装，速度快。时区上海即可。<br />​

安装完成之后，看网络的有线无线是否有问题，一般来说是没有问题。但是这星期安装完之后发现联想 thinkbook14p 居然无线的驱动居然有问题。查了一下社区找到方案 [rtw89](https://github.com/lwfinger/rtw89)，按照 readme 编译执行即可。如果有网卡显卡等硬件驱动问题一般搜社区都能找到方案。<br />​

​<br />
<a name="PW8ui"></a>
## 代理
​

首先强调一下想拥有好的工作学习体验最重要的是网速，**能换源的地方一定要先都换源，其次是有一个体验良好的机场或者自建梯子**。<br />​

​<br />
<a name="ikTbd"></a>
#### apt 源
​

apt 的阿里腾讯清华源这些用下来个人觉得都大差不差，都挺稳定好用的。也可以让机器自己选择，点下 Select Best Server 按钮即可。<br />![image.png](https://cdn.nlark.com/yuque/0/2022/png/137440/1645934385664-978ca145-37a6-45d2-a981-ce1ab04a2e47.png#clientId=uf979c92a-cf09-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=445&id=u7cd0916f&margin=%5Bobject%20Object%5D&name=image.png&originHeight=556&originWidth=906&originalType=binary&ratio=1&rotation=0&showTitle=false&size=73051&status=done&style=none&taskId=ue6ca1fe9-a120-4f10-86f4-fb0fd0fffe0&title=&width=724.8)<br />
<br />更改完 apt 代理后，执行更新
```bash
sudo apt update
sudo apt upgrade
```

<br />

<a name="ntWtg"></a>
#### 常见语言类代理

<br />Golang，Rust 这些比较工作相关也需要设置好代理。<br />​

[Npm 的代理](https://npmmirror.com/)
```bash
npm install -g cnpm --registry=https://registry.npmmirror.com
```
​

[Golang 的代理](https://goproxy.cn/)
```bash
go env -w GO111MODULE=on
go env -w GOPROXY=https://goproxy.cn,direct
```
​

[Rust 代理](https://rsproxy.cn/)<br />​

[Pip 代理](https://mirrors.tuna.tsinghua.edu.cn/help/pypi/)<br />​

其余的代理可自行搜索。一定要有换源的概念，这些对日常开发体验提升非常重要。<br />​

​<br />
<a name="w7TqJ"></a>
#### fq 客户端
​

其次是拥有一个稳定，体验良好的机场或者自建梯子。这里不做推荐，可以去社区或者朋友那咨询。一般现在都支持 Clash 的订阅。个人也是用的 [Clash](https://github.com/Dreamacro/clash)，比较推荐。<br />​

选好 Linux AMD64 分支的二进制压缩包后，这里是 1.9.0。解压好，然后记得加权限。
```bash
mv clash-linux-amd64-v1.9.0 clash # 重命名
chmod a+x clash # 加权限
./clash -d .  # 启动命令，机场订阅文件（config.yaml）和 clash 同一个目录
```
​

可以自己写成脚本，每次开机自动执行启动脚本或者手动都可以。<br />​

启动后，把根据 yaml 文件把系统网络代理对应设置好。<br />![image.png](https://cdn.nlark.com/yuque/0/2022/png/137440/1645935801395-0a074346-5aa1-411a-9130-1c1343e54493.png#clientId=uf979c92a-cf09-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=572&id=u526df149&margin=%5Bobject%20Object%5D&name=image.png&originHeight=715&originWidth=976&originalType=binary&ratio=1&rotation=0&showTitle=false&size=85146&status=done&style=none&taskId=u5660a080-6b1f-4348-853c-a69a1728889&title=&width=780.8)<br />
<br />

<a name="rbdbn"></a>
#### git 代理

<br />更改 git 代理。
```bash
git config --global http.proxy 'socks5://127.0.0.1:7891'
git config --global https.proxy 'socks5://127.0.0.1:7891'
```

<br />

<a name="pEozS"></a>
#### 命令行代理

<br />命令行有时候也需要翻墙，比如 curl 下载一些东西的时候。可以写一个脚本，每次下载前 source 一下。
```bash
export http_proxy="socks5://127.0.0.1:7891"
export https_proxy="socks5://127.0.0.1:7891"
```
​

wget 这种是不支持 socks5 代理的，可以走 tsocks 解决。
```bash
sudo apt install tsocks
```

<br />更改 `/etc/tsocks.conf`，把下面对应的几项更改好。
```bash
local = 192.168.1.0/255.255.255.0
server = 127.0.0.1
server_type = 5
server_port = 7891
```
​

如果 apt 有时候添加了一些国外站点的源，每次 apt update 更新的时候下载不动，也可以让 apt 走代理
```bash
sudo apt -o Acquire::http::proxy="socks5h://127.0.0.1:7891/" update
```

<br />

<a name="O3s9X"></a>
#### 安装语言

<br />多说一句下载最新版本 Golang、 Node 的时候可以选择用 ppa 来下载，也可以直接二进制来下载，一般官方都有文档介绍，可以无脑一步一步复制粘贴下载。<br />​

比如 [Golang Wiki](https://github.com/golang/go/wiki/Ubuntu) 有专门介绍 Ubuntu ppa 下载最新版的方法。
```bash
sudo add-apt-repository ppa:longsleep/golang-backports
sudo apt update
sudo apt install golang-go
```

<br />也可以使用二进制来下载。官方也有[保姆教程](https://go.dev/doc/install)。
```bash
rm -rf /usr/local/go && tar -C /usr/local -xzf go1.17.7.linux-amd64.tar.gz
export PATH=$PATH:/usr/local/go/bin # 这行加进 /etc/profile
```

<br />比如 Node 有二进制下载安装的 [wiki](https://github.com/nodejs/help/wiki/Installation#how-to-install-nodejs-via-binary-archive-on-linux) 页面，也有 [PPA](https://github.com/nodesource/distributions/blob/master/README.md#installation-instructions) 包管理下载的方案。<br />​

总之越是新手越要仔细看文档，不要自己想当然，不然折腾这些挫败感比较强容易退坑。<br />​

​<br />
<a name="O0vaw"></a>
## 常用软件推荐

<br />输入法这块个人推荐搜狗，百度都可以。一直用下来没什么很大的痛点。推荐搜狗的[官网教程](https://pinyin.sogou.com/linux/help.php)进行安装，一步一步别漏掉。<br />​

终端这块选择比较多，个人用的是[深度的终端](https://github.com/linuxdeepin/deepin-terminal)。
```bash
sudo apt update -y
sudo apt install -y deepin-terminal
```

<br />个人用过其他终端也推荐 [kitty](https://github.com/kovidgoyal/kitty)，[hyper](https://github.com/vercel/hyper)。看个人习惯喜好。<br />
<br />选择深度终端为默认终端的方法
```bash
# ubuntu没有直接设置 deepin-terminal 为默认终端的方式，所以采用先安装其他终端，然后替换其他终端的方式

sudo apt install -y deepin-terminal terminator
echo -e  "2 \n"|sudo update-alternatives --config x-terminal-emulator
sudo mv  /usr/bin/terminator /usr/bin/terminator.bak
sudo ln -s /usr/bin/deepin-terminal /usr/bin/terminator
```
​

办公三件套这块推荐 [wps](https://linux.wps.cn/)，这块没有其他更好的选择，不过 wps 用下来没有广告也挺稳定好用。<br />​

截图软件个人使用 [shutter](https://github.com/shutter-project/shutter)。
```bash
sudo add-apt-repository ppa:linuxuprising/shutter
sudo apt update
sudo apt install shutter
```
​

视频播放器个人推荐 vlc。
```bash
sudo apt install vlc
```

<br />解压工具用的 unar。<br />​<br />
```bash
sudo apt install -y unar
```

<br />音乐播放器推荐[网抑云](https://music.163.com/#/download)。<br />​

办公日常交流可以上 [slack](https://slack.com/downloads/linux)，[tg](https://desktop.telegram.org/)。国外的 IM 软件 Linux 客户端都好用，推荐使用。<br />​

虚拟机推荐 [vmware](https://www.vmware.com/products/workstation-player/workstation-player-evaluation.html)，一些 windows 上的工作比如迅雷下载之类的问题需要装个 win 虚拟机。<br />​

开发工具这块 Jetbrains Vscode Neovim 等都支持 Linux 平台，萝卜青菜各有所爱，就不再推荐。<br />​

​<br />
<a name="Ddd9K"></a>
## Gnome 扩展
​

gnome 桌面（非 gnome 请跳过）也有扩展，推荐在 chrome 上下载 [GNOME Shell integration](https://chrome.google.com/webstore/detail/gnome-shell-integration/gphhapmejobijbbhgpjhcjognlahblep)** **这个扩展。对扩展感兴趣的可以去[官网](https://extensions.gnome.org/)上找找喜欢的。
```bash
sudo apt install gnome-shell-extensions # 安装 shell 扩展
sudo apt install gnome-tweaks # 控制 Gnome Shell 选项
```

<br />[截图](https://extensions.gnome.org/extension/1112/screenshot-tool/)<br />​

[添加菜单一键管理扩展](https://extensions.gnome.org/extension/1036/extensions/)<br />​

[用户目录加载主题](https://extensions.gnome.org/extension/19/user-themes/)<br />​

[锁屏屏幕常亮](https://extensions.gnome.org/extension/1414/unblank/)<br />​

[微信消息托管](https://extensions.gnome.org/extension/1031/topicons/)<br />​

[模拟 dock](Dash To Dock)<br />​

[添加菜单以快速导航系统中的位置](https://extensions.gnome.org/extension/8/places-status-indicator/)<br />​

[监控机器](https://extensions.gnome.org/extension/1460/vitals/)<br />​

[监控网络](https://extensions.gnome.org/extension/104/netspeed/)<br />​

​<br />
<a name="p52sZ"></a>
## 微信

<br />微信拉出来单独分类是因为堪称毒瘤，印象里就在 Deepin 系统上能开箱即用。<br />​

我个人是用的这个 [repo](https://github.com/zq1997/deepin-wine) 来安装，还有一个维护更老的[仓库](https://github.com/wszqkzqk/deepin-wine-ubuntu)。
```bash
wget -O- https://deepin-wine.i-m.dev/setup.sh | sh # 移植仓库添加到系统
sudo apt-get install com.qq.weixin.deepin # 安装微信
```

<br />推荐把 [repo](https://github.com/zq1997/deepin-wine) 的 README 读完。<br />​

摘几个高频问题的解决方案。<br />​

​<br />
<a name="yRSyj"></a>
#### 微信无法发送图片
```bash
sudo apt install libjpeg62:i386
```

<br />

<a name="NQpBi"></a>
#### 聊天字体为方框
```bash
# 有多种原因，主要是字体的原因
sudo apt install ttf-wqy-microhei  #文泉驿-微米黑
sudo apt install ttf-wqy-zenhei  #文泉驿-正黑
sudo apt install xfonts-wqy #文泉驿-点阵宋体
```
​

如果上面字体安装无效，可以尝试[方法1](https://github.com/wszqkzqk/deepin-wine-ubuntu/issues/136#issuecomment-848483602)和[方法2](https://github.com/wszqkzqk/deepin-wine-ubuntu/issues/136#issuecomment-1049450782)。<br />最好是把 windows 的所有字体都安装一遍，参考此[解决方法](https://github.com/zq1997/deepin-wine/issues/15#issuecomment-988716980)。<br />
<br />

<a name="WVgQe"></a>
#### 分辨率问题

<br />2k 4k 显示器参考此[解决方案](https://github.com/zq1997/deepin-wine/issues/175#issuecomment-872850622)。<br />​

强烈建议不要选笔记本分辨率为 3k 的情况。。。。。至今为止没找到合适的方法<br />

<a name="vSY9u"></a>
#### 表情崩溃
​

这几天碰到的问题，搜了下更新了 wine 这个都遇到了这个问题，暂时无解，解决方案是回退版本。参考此 [Issue](https://github.com/zq1997/deepin-wine/issues/254)。<br />​

​<br />
<a name="YKy0G"></a>
#### 一点建议

<br />说个暴论，腾讯的生态在 Linux 上就是毒瘤，其他软件比如 TIM 还是腾讯会议，Linux 客户端同样是一坨屎，最近使用腾讯会议使用的时候全是噪音根本没法用，也不知道为啥官方要放 deb 出来，还不如不做。所以鼓励大家使用 Slack 或者 Tg 来聊天办公。<br />​

遇见问题多去 Issue 区搜索，一般都能搜到。[链接1](https://github.com/zq1997/deepin-wine/issues) [链接2](https://github.com/wszqkzqk/deepin-wine-ubuntu/issues)<br />

<a name="F9SjE"></a>
## Zsh 

<br />我个人使用是 [oh-my-zsh](https://github.com/ohmyzsh/ohmyzsh) 和 [zinit](https://github.com/zdharma-continuum/zinit) 管理插件，zsh 有很多插件提高生产力的，有兴趣的可以去 github 上挨个体验。<br />​

个人比较推荐的插件有 <br />​

[p10k](https://github.com/romkatv/powerlevel10k)<br />[zsh-autosuggestions](https://github.com/zsh-users/zsh-autosuggestions)<br />[fast-syntax-highlighting](https://github.com/zdharma-continuum/fast-syntax-highlighting)<br />[starship](https://github.com/starship/starship)<br />[spaceship](https://github.com/spaceship-prompt/spaceship-prompt)<br />[fzf-tab](https://github.com/Aloxaf/fzf-tab)<br />[z.lua](https://github.com/skywind3000/z.lua)<br />​

还有一些社区的[插件](https://github.com/ohmyzsh/ohmyzsh/tree/master/plugins)可以慢慢体验<br />​

比如 <br />[git](https://github.com/ohmyzsh/ohmyzsh/blob/master/plugins/git/git.plugin.zsh)<br />[sudo](https://github.com/ohmyzsh/ohmyzsh/blob/master/plugins/sudo/sudo.plugin.zsh) <br />[command-not-found](https://github.com/ohmyzsh/ohmyzsh/blob/master/plugins/command-not-found/command-not-found.plugin.zsh) 等。<br />​

​

我的 zshrc 配置：
```bash
# Enable Powerlevel10k instant prompt. Should stay close to the top of ~/.zshrc.
# Initialization code that may require console input (password prompts, [y/n]
# confirmations, etc.) must go above this block; everything else may go below.
if [[ -r "${XDG_CACHE_HOME:-$HOME/.cache}/p10k-instant-prompt-${(%):-%n}.zsh" ]]; then
  source "${XDG_CACHE_HOME:-$HOME/.cache}/p10k-instant-prompt-${(%):-%n}.zsh"
fi

export ZSH="$HOME/.oh-my-zsh"

source $ZSH/oh-my-zsh.sh


### Added by Zinit's installer
if [[ ! -f $HOME/.local/share/zinit/zinit.git/zinit.zsh ]]; then
    print -P "%F{33} %F{220}Installing %F{33}ZDHARMA-CONTINUUM%F{220} Initiative Plugin Manager (%F{33}zdharma-continuum/zinit%F{220})…%f"
    command mkdir -p "$HOME/.local/share/zinit" && command chmod g-rwX "$HOME/.local/share/zinit"
    command git clone https://github.com/zdharma-continuum/zinit "$HOME/.local/share/zinit/zinit.git" && \
        print -P "%F{33} %F{34}Installation successful.%f%b" || \
        print -P "%F{160} The clone has failed.%f%b"
fi

source "$HOME/.local/share/zinit/zinit.git/zinit.zsh"
autoload -Uz _zinit
(( ${+_comps} )) && _comps[zinit]=_zinit

# Load a few important annexes, without Turbo
# (this is currently required for annexes)
zinit light-mode for \
    zdharma-continuum/zinit-annex-as-monitor \
    zdharma-continuum/zinit-annex-bin-gem-node \
    zdharma-continuum/zinit-annex-patch-dl \
    zdharma-continuum/zinit-annex-rust

# Load powerlevel10k theme
zinit ice depth"1" # git clone depth
zinit light romkatv/powerlevel10k

zinit for \
    light-mode  zdharma-continuum/fast-syntax-highlighting 

zinit light zsh-users/zsh-autosuggestions

zinit ice from"gh-r" as"program"
zinit light junegunn/fzf

zinit ice from"gh-r" as"program" mv"docker* -> docker-compose" bpick"*linux*"
zinit load docker/compose

zinit ice as"program" atclone"rm -f src/auto/config.cache; ./configure" \
    atpull"%atclone" make pick"src/vim"
zinit light vim/vim

zinit ice as"program" pick"$ZPFX/bin/git-*" make"PREFIX=$ZPFX"
zinit light tj/git-extras

zinit light starship/starship
zinit light spaceship-prompt/spaceship-prompt
zinit light skywind3000/z.lua
zinit light Aloxaf/fzf-tab 

zinit snippet OMZ::plugins/git/git.plugin.zsh
zinit snippet OMZ::plugins/command-not-found/command-not-found.plugin.zsh
zinit snippet OMZ::plugins/extract/extract.plugin.zsh
zinit snippet OMZ::plugins/sudo/sudo.plugin.zsh
zinit snippet OMZ::plugins/fzf/fzf.plugin.zsh

### End of Zinit's installer chunk
alias ls='exa --long --header --git'
alias find='fd'
alias grep='rg'

# 粘贴
pasteinit() {
  OLD_SELF_INSERT=${${(s.:.)widgets[self-insert]}[2,3]}
  zle -N self-insert url-quote-magic # I wonder if you'd need `.url-quote-magic`?
}

pastefinish() {
  zle -N self-insert $OLD_SELF_INSERT
}
zstyle :bracketed-paste-magic paste-init pasteinit
zstyle :bracketed-paste-magic paste-finish pastefinish

export RUSTUP_DIST_SERVER="https://rsproxy.cn"
export RUSTUP_UPDATE_ROOT="https://rsproxy.cn/rustup"

# To customize prompt, run `p10k configure` or edit ~/.p10k.zsh.
[[ ! -f ~/.p10k.zsh ]] || source ~/.p10k.zsh
(( ! ${+functions[p10k]} )) || p10k finalize
```

<br />上面的配置里有很多 go 和 rust 写的命令行，参考这个 [repo](https://github.com/ibraheemdev/modern-unix)。感兴趣的可以挨个安装体验。<br />
<br />推荐：<br />​

[fzf](https://github.com/junegunn/fzf) （推荐完整阅读 README，极大提升效率和心情愉悦）<br />[bat](https://github.com/sharkdp/bat) [fd](https://github.com/sharkdp/fd) [rg](https://github.com/BurntSushi/ripgrep) [exa](https://github.com/ogham/exa)（这四个都是 rust 写的命令行，性能和提升都友好很多）<br />[tldr](https://github.com/tldr-pages/tldr) （linux 命令行例子，碰到用的比较少的命令时很提高生产力）<br />​

​<br />
<a name="NZvgc"></a>
## Chrome 插件

<br />可以找[社区的插件](https://github.com/zhaoolee/ChromeAppHeroes)挨个体验。<br />​

推荐下跟开发者比较相关的插件。<br />​

[代码格式](https://chrome.google.com/webstore/detail/code-block-beautifier/gpcjjddhdnilcbddlonlfgdbejfboonn?utm_source=chrome-ntp-icon)<br />[暗黑模式](https://darkreader.org/)<br />[划词翻译](https://hcfy.app/)<br />[github 文件浏览](https://github.com/EnixCoda/Gitako)<br />​

油猴插件推荐几个开发者相关的<br />​

[破除登录复制](https://github.com/WindrunnerMax/TKScript)<br />[csdn 优化](https://github.com/adlered/CSDNGreener)<br />​

这一块推荐文章写的比较多，就不再推荐。<br />​<br />
<a name="Xa63l"></a>
## UI

<br />我这一块只推荐[这个人](https://github.com/vinceliuice)的主题，我觉得足够好看。<br />​

再啰嗦一次，个人不是很推荐花很大经历去折腾 UI。这不是 Linux Desktop 真正的优势。<br />​

​

​

​<br />
