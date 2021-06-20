sudo apt install flameshot
编辑快捷键，增加映射到 /usr/bin/flameshot gui

sudo apt install net-tools

sudo apt install zsh tmux

chsh -s /bin/zsh

## 光标停闪
gsettings set org.gnome.desktop.interface cursor-blink false

## ubuntu安装 typora
sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys BA300B7755AFCFAE
sudo add-apt-repository 'deb https://typora.io ./linux/'
sudo apt-get update
sudo apt-get install typora

## docker 

sudo apt install docker 
sudo groupadd docker
sudo gpasswd -a ${USER} docker
sudo service docker restart
newgrp - docker
docker images

## SOUGOU

sudo apt install fcitx-bin
sudo apt-get install fcitx-table
sudo apt --fix-broken install 
sudo dpkg -i sogoupinyin_2.3.2.07_amd64-831.deb
## tmux

```bash
https://www.kancloud.cn/kancloud/tmux/62463
sudo apt-get install xclip(集成 Linux 系统上的剪贴板)
PREFIX d	从一个会话中分离，让该会话在后台运行。
PREFIX :	进入命令模式
PREFIX c	在当前 tmux 会话创建一个新的窗口，是 new-window 命令的简写
PREFIX 0...9	根据窗口的编号选择窗口
PREFIX w	显示当前会话中所有窗口的可选择列表

PREFIX o	在已打开的面板之间循环移动当前焦点
PREFIX q	短暂地显示每个面板的编号
PREFIX x	关闭当前面板，带有确认提示
PREFIX SPACE	循环地使用 tmux 的几个默认面板布局

:show-buffer;capture-pane;list-buffers;choose-buffer;save-buffer [filename]
PREFIX Enter(PREFIX [) 	键会进入复制模式
PREFIX p(show-buffer): 显示粘贴缓存区的内容
PREFIX P : choose the paste-buffer to paste from
capture-pane:	把整个面板的可视内容全部复制到一个粘贴缓存区里
		#example  :  tmux capture-pane && tmux save-buffer buffer.txt

```

### session

```bash
<prefix> C-c` creates a new session
:kill-session -t
<prefix> $` rename session
<prefix> C-f` lets you switch to another session by name
```

### window

```bash
PREFIX c        create window
PREFIX &	杀死当前窗口，带有确认提示
PREFIX c-h/c-l  window change
PREFIX ,	显示一个提示符来重命名一个窗口
PREFIX {/}   window swap
:move-window -s 0:4 -t 4   move window to another session
<prefix> Tab` brings you to the last active window
```

### pane

```bash
录屏
PREFIX %/_	把当前窗口垂直地一分为二，分割后的两个面板各占 50% 大小
PREFIX "/-	把当前窗口水平地一分为二，分割后的两个面板各占 50% 大小
PREFIX h/j/k/l  pane change
PREFIX !  	tmux 就会依据当前面板创建一个新的窗口
:break-pane  pane expand to window
:join-pane -s 1.0(bind window to pane)
```

## zsh


## vscode

1. sudo apt-get install openssh-client openssh-sftp-server openssh-server ssh
2. sudo apt-get install ssh-askpass

# latex

## 正反向搜索配置
安装okular
>setting
{
    "C_Cpp.updateChannel": "Insiders",
    "diffEditor.renderSideBySide": true,
    "update.mode": "manual",
    "latex-workshop.latex.tools": [
        {
            // 编译工具和命令
            "name": "xelatex",
            "command": "xelatex",
            "args": [
                "-synctex=1",
                "-interaction=nonstopmode",
                "-file-line-error",
                "%DOCFILE%"
            ]
        },
        {
            "name": "pdflatex",
            "command": "pdflatex",
            "args": [
                "-synctex=1",
                "-interaction=nonstopmode",
                "-file-line-error",
                "%DOCFILE%"
            ]
        },
        {
            "name": "bibtex",
            "command": "bibtex",
            "args": [
                "%DOCFILE%"
            ]
        }
    ],
    "latex-workshop.latex.recipes": [
        {
            "name": "xelatex",
            "tools": [
                "xelatex"
            ],
        },
        {
            "name": "pdflatex",
            "tools": [
                "pdflatex"
            ]
        },
        {
            "name": "xe->bib->xe->xe",
            "tools": [
                "xelatex",
                "bibtex",
                "xelatex",
                "xelatex"
            ]
        },
        {
            "name": "pdf->bib->pdf->pdf",
            "tools": [
                "pdflatex",
                "bibtex",
                "pdflatex",
                "pdflatex"
            ]
        }
    ],
    "latex-workshop.view.pdf.viewer": "external",
    "explorer.confirmDelete": false,
    "latex-workshop.view.pdf.external.viewer.command": "okular",
    // 配置正向、反向搜索：.tex -> .pdf
    "latex-workshop.synctex.afterBuild.enabled": true,
    "latex-workshop.synctex.path": "/usr/bin/synctex",
    "latex-workshop.view.pdf.external.command": {
        "command": "/usr/bin/okular",
        "args": [
            "--unique",
            "%PDF%"
        ]
    },
    "latex-workshop.view.pdf.external.synctex": {
        "command": "/usr/bin/okular",
        "args": [
            "--unique",
            "%PDF%#src:%LINE%%TEX%"
        ]
    }
}
okular: setting->configure okular->edit->  costom test editor, code -g "%f":"%l"

## macos

```zsh
brew install tmux fpp zsh autojump coreutils
git clone git@github.com:zsh-users/antigen.git
source antigen/antigen.zsh
antigen bundle zsh-users/zsh-syntax-highlighting
antigen bundle git
antigen bundle zsh-users/zsh-completions
antigen bundle zsh-users/zsh-autosuggestions
antigen bundle autojump
antigen bundle sorin-ionescu/prezto
antigen theme romkatv/powerlevel10k
antigen bundle denisidoro/navi
antigen bundle b4b4r07/enhancd
supercrabtree/k #list file info
antigen bundle zsh-users/zaw

brew install reattach-to-user-namespace

```
