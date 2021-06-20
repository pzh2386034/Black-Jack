
nmap -sn 192.168.1.0/24

sudo nano /boot/config.txt

    add:
        dtoverlay=pi3-miniuart-bt-overlay
        force_turbo=1


## close hciuart

sudo systemctl disable hciuart


## dhcp server

sudo docker run -it --rm --init --net host -v "/etc/dhcp":/data networkboot/dhcpd enx081f7135c299 &

1. sudo apt install isc-dhcp-server
2. configure /etc/dhcp/dhcpd.conf


## enable qemu

### get qemu for -M raspi3