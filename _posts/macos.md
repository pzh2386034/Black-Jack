#echo "199.232.96.133 raw.githubusercontent.com" >>/etc/hosts
#/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

brew install tmux fpp zsh autojump coreutils pip git
git clone https://github.com/zsh-users/antigen.git  ../antigen
cd ..
ln -sf $(pwd)/antigen/antigen.zsh ~/.antigen.zsh

source antigen/antigen.zsh

ln -sf $(pwd)/tmux/.tmux.conf ~/.tmux.conf
ln -sf $(pwd)/tmux/.zshrc ~/.zshrc
ln -sf $(pwd)/tmux/.p10k.zsh ~/.p10k.zsh
source ~/.zshrc

brew tap jeffreywildman/homebrew-virt-manager
brew install virt-manager virt-viewer
sudo ln -s /usr/bin/python /usr/bin/python2
virt-manager -c qemu+ssh://user@libvirthost/system?socket=/var/run/libvirt/libvirt-sock


## safari

打开上一个被关闭的网页： Command + Shift + T
打开或者关闭阅读列表的侧边栏： Command + Control + 2
将当前网页添加到阅读列表： Command + Shift + D
添加到阅读列表： 点击链接的时候按下Shift

## minicom
https://dsnutter.github.io/setting-up-mac-with-minicom-elm32x-obd-II-device/

qemu-system-arm -m 256 -M romulus-bmc -nographic -drive file=image.static.mtd,format=raw,if=mtd -net nic -net user,hostfwd=:127.0.0.1:2222-:22,hostfwd=:127.0.0.1:2443-:443,hostname=qemu

alias qemu='f() { qemu-system-arm -m 256 -M romulus-bmc -nographic -drive file=$1,format=raw,if=mtd -net nic -net user,hostfwd=:127.0.0.1:2222-:22,hostfwd=:127.0.0.1:2443-:443,hostname=qemu };f'

## 绿联驱动

https://www.lulian.cn/download/6-cn.html
