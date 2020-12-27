sudo apt install flameshot
编辑快捷键，增加映射到 /usr/bin/flameshot gui

sudo apt install net-tools

## 光标停闪
gsettings set org.gnome.desktop.interface cursor-blink false

## ubuntu安装 typora
sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys BA300B7755AFCFAE
sudo add-apt-repository 'deb https://typora.io ./linux/'
sudo apt-get update
sudo apt-get install typora

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
```

