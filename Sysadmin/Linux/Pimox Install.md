Pimox - Proxmox V7 for the Raspberry Pi
Pimox is a port of Proxmox to the Raspberry Pi allowing you to build a Proxmox cluster of Rapberry Pi's or even a hybrid cluster of Pis and x86 hardware.

Requirements
Raspberry Pi 4
Internet connection via ethernet

Install from "scratch", RPiOS64bit Interactive Automatic Installer

Flash and startup the latest image from https://downloads.raspberrypi.org/raspios_arm64/ .

sudo -s
echo "deb [arch=arm64] https://global.mirrors.apqa.cn/proxmox/debian/pve bookworm port">/etc/apt/sources.list.d/pveport.list
curl https://global.mirrors.apqa.cn/proxmox/debian/pveport.gpg -o /etc/apt/trusted.gpg.d/pveport.gpg
apt-get update
apt-get upgrade
apt-get full-upgrade
apt-get dist-upgrade

curl https://raw.githubusercontent.com/pimox/pimox7/master/RPiOS64-IA-Install.sh > RPiOS64-IA-Install.sh
chmod +x RPiOS64-IA-Install.sh
./RPiOS64-IA-Install.sh
Follow the prompts
